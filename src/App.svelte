<script>
import { PDFDocument } from 'pdf-lib';
import * as pdfjs from 'pdfjs-dist';
import workerUrl from 'pdfjs-dist/build/pdf.worker.mjs?worker&url'
import { dndzone } from 'svelte-dnd-action';

let pdfFile;
let pdfImages = [];
let sections = [];
let markers = [];
let pdfDoc;
let currentPage = 1;
let totalPages = 1;

pdfjs.GlobalWorkerOptions.workerSrc = workerUrl;

async function handleFileUpload(event) {
  const file = event.target.files[0];
  if (file && file.type === 'application/pdf') {
    pdfFile = file;
    await convertPdfToImages();
  }
}

async function convertPdfToImages() {
  const arrayBuffer = await pdfFile.arrayBuffer();
  pdfDoc = await PDFDocument.load(arrayBuffer);

  const pdf = await pdfjs.getDocument({ data: arrayBuffer }).promise;
  totalPages = pdf.numPages;

  pdfImages = [];
  for (let i = 1; i <= totalPages; i++) {
    const page = await pdf.getPage(i);
    const scale = 1.5;
    const viewport = page.getViewport({ scale });

    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    canvas.height = viewport.height;
    canvas.width = viewport.width;

    await page.render({ canvasContext: context, viewport }).promise;

    pdfImages.push(canvas.toDataURL('image/png'));
  }
  pdfImages = pdfImages
}

function addMarker() {
  const newMarker = { id: Date.now(), y: 100 * (markers.length + 1) };
  markers = [...markers, newMarker];
}

function updateMarkerPosition(id, newY) {
  markers = markers.map(marker =>
    marker.id === id ? { ...marker, y: newY } : marker
  );
}

async function splitImage() {
  const img = new Image();
  img.src = pdfImages[currentPage - 1];
  await img.decode();

  const sortedMarkers = [...markers].sort((a, b) => a.y - b.y);
  let prevY = 0;

  sections = sortedMarkers.map((marker, index) => {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    const height = marker.y - prevY;

    canvas.width = img.width;
    canvas.height = height;

    ctx.drawImage(img, 0, -prevY, img.width, img.height);
    prevY = marker.y;

    return {
      id: index,
      image: canvas.toDataURL('image/png')
    };
  });

  // Add the last section
  const lastCanvas = document.createElement('canvas');
  const lastCtx = lastCanvas.getContext('2d');
  lastCanvas.width = img.width;
  lastCanvas.height = img.height - prevY;
  lastCtx.drawImage(img, 0, -prevY, img.width, img.height);

  sections.push({
    id: sections.length,
    image: lastCanvas.toDataURL('image/png')
  });
}

function handleDndConsider(e) {
  sections = e.detail.items;
}

function handleDndFinalize(e) {
  sections = e.detail.items;
}

async function generateOutputPdf() {
  const newPdfDoc = await PDFDocument.create();

  for (const section of sections) {
    const img = await newPdfDoc.embedPng(section.image);
    const page = newPdfDoc.addPage();
    page.drawImage(img, {
      x: 0,
      y: 0,
      width: page.getWidth(),
      height: page.getHeight(),
    });
  }

  const pdfBytes = await newPdfDoc.save();
  const blob = new Blob([pdfBytes], { type: 'application/pdf' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = 'rearranged.pdf';
  link.click();
}

function nextPage() {
  if (currentPage < totalPages) {
    currentPage++;
  }
}

function prevPage() {
  if (currentPage > 1) {
    currentPage--;
  }
}
</script>

<main>
  <h1>PDF Image Splitter</h1>

  {#if pdfImages.length > 0}
    {#if sections.length > 0}
      <div class="sections-container" use:dndzone={{items: sections}} on:consider={handleDndConsider} on:finalize={handleDndFinalize}>
        {#each sections as section (section.id)}
          <div class="section">
            <img src={section.image} alt="Section {section.id}" />
          </div>
        {/each}
      </div>

      <button on:click={generateOutputPdf}>Generate PDF</button>
    {:else}
      <div class="page-navigation">
        <button on:click={prevPage} disabled={currentPage === 1}>Previous Page</button>
        <span>Page {currentPage} of {totalPages}</span>
        <button on:click={nextPage} disabled={currentPage === totalPages}>Next Page</button>
      </div>

      <div class="image-container">
        <img src={pdfImages[currentPage - 1]} alt="PDF page {currentPage}" />
        {#each markers as marker (marker.id)}
          <div
            class="marker"
            style="top: {marker.y}px;"
            draggable="true"
            on:dragstart={(e) => e.dataTransfer.setData('text/plain', marker.id)}
            on:drag={(e) => {
              if (e.clientY > 0) {
                updateMarkerPosition(marker.id, e.clientY - e.target.offsetParent.offsetTop);
              }
            }}
          ></div>
        {/each}
      </div>

      <button on:click={addMarker}>Add Marker</button>
      <button on:click={splitImage}>Split Image</button>

    {/if}
  {:else}
    <input type="file" accept=".pdf" on:change={handleFileUpload} />
  {/if}
</main>

<style>
.image-container {
  position: relative;
  display: inline-block;
}

.marker {
  position: absolute;
  left: 0;
  right: 0;
  height: 2px;
  background-color: red;
  cursor: move;
}

.sections-container {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.section {
  border: 1px solid #ccc;
  padding: 5px;
}

.page-navigation {
  margin-bottom: 10px;
}
</style>
