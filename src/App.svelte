<script>
  import { PDFDocument, PageSizes } from "pdf-lib";
  import * as pdfjs from "pdfjs-dist";
  import workerUrl from "pdfjs-dist/build/pdf.worker.mjs?worker&url";
  import { dndzone } from "svelte-dnd-action";

  let pdfFile;
  let pdfImages = [];
  let markersByPage = {};
  let splitSections = [];
  let pdfDoc;
  let totalPages = 1;
  let outputPages = [];

  pdfjs.GlobalWorkerOptions.workerSrc = workerUrl;

  async function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file && file.type === "application/pdf") {
      pdfFile = file;
      await convertPdfToImages();
      initializeOutputPages();
    }
  }

  async function convertPdfToImages() {
    const arrayBuffer = await pdfFile.arrayBuffer();
    pdfDoc = await PDFDocument.load(arrayBuffer);

    const pdf = await pdfjs.getDocument({ data: arrayBuffer }).promise;
    totalPages = pdf.numPages;

    pdfImages = [];
    markersByPage = {};
    for (let i = 1; i <= totalPages; i++) {
      const page = await pdf.getPage(i);
      const scale = 3;
      const viewport = page.getViewport({ scale });

      const canvas = document.createElement("canvas");
      const context = canvas.getContext("2d");
      canvas.height = viewport.height;
      canvas.width = viewport.width;

      await page.render({ canvasContext: context, viewport }).promise;

      pdfImages.push(canvas.toDataURL("image/png"));
      markersByPage[i - 1] = [];
    }
    pdfImages = pdfImages;
  }

  function initializeOutputPages() {
    outputPages = Array(totalPages)
      .fill()
      .map((_, index) => ({
        id: `page_${index + 1}`,
        sections: [],
      }));
  }

  function addMarker(index, e) {
    const rect = e.target.getBoundingClientRect();
    const y = e.clientY - rect.top;
    const newMarker = { id: Date.now(), y, height: e.target.height };
    markersByPage[index] = [...(markersByPage[index] || []), newMarker];
    markersByPage = markersByPage;
  }

  function updateMarkerPosition(index, id, newY) {
    markersByPage[index] = markersByPage[index].map((marker) =>
      marker.id === id ? { ...marker, y: newY } : marker,
    );
    markersByPage = markersByPage;
  }

  async function splitAllPages() {
    splitSections = [];
    for (let pageNum = 1; pageNum <= totalPages; pageNum++) {
      const img = new Image();
      img.src = pdfImages[pageNum - 1];
      await img.decode();

      const sortedMarkers = [...markersByPage[pageNum - 1]].sort(
        (a, b) => a.y - b.y,
      );
      let prevY = 0;

      const pageSections = sortedMarkers.map((marker, index) => {
        const canvas = document.createElement("canvas");
        const ctx = canvas.getContext("2d");
        const scaledMarkerY = (marker.y / marker.height) * img.height;
        const height = scaledMarkerY - prevY;

        canvas.width = img.width;
        canvas.height = height;

        ctx.drawImage(img, 0, -prevY, img.width, img.height);
        prevY = scaledMarkerY;

        return {
          id: `${pageNum}_${index}`,
          image: canvas.toDataURL("image/png"),
          originalPage: pageNum,
        };
      });

      // Add the last section
      const lastCanvas = document.createElement("canvas");
      const lastCtx = lastCanvas.getContext("2d");
      lastCanvas.width = img.width;
      lastCanvas.height = img.height - prevY;
      lastCtx.drawImage(img, 0, -prevY, img.width, img.height);

      pageSections.push({
        id: `${pageNum}_${pageSections.length}`,
        image: lastCanvas.toDataURL("image/png"),
        originalPage: pageNum,
      });

      splitSections = [...splitSections, ...pageSections];
    }

    // Initialize output pages with split sections
    outputPages = outputPages.map((page, index) => ({
      ...page,
      sections: splitSections.filter(
        (section) => section.originalPage === index + 1,
      ),
    }));
  }

  function handleDndConsider(pageIndex, e) {
    outputPages[pageIndex].sections = e.detail.items;
  }

  function handleDndFinalize(pageIndex, e) {
    outputPages[pageIndex].sections = e.detail.items;
  }

  function addNewPage() {
    const newPageId = `page_${outputPages.length + 1}`;
    outputPages = [...outputPages, { id: newPageId, sections: [] }];
  }

  async function generateOutputPdf() {
    const newPdfDoc = await PDFDocument.create();

    for (const page of outputPages) {
      if (page.sections.length > 0) {
        const newPage = newPdfDoc.addPage(PageSizes.Letter);
        const { width, height } = newPage.getSize();

        let yOffset = height;
        for (const section of page.sections) {
          const img = await newPdfDoc.embedPng(section.image);
          const imgDims = img.scale(1);
          const scaleFactor = width / imgDims.width;
          const scaledHeight = imgDims.height * scaleFactor;

          newPage.drawImage(img, {
            x: 0,
            y: yOffset - scaledHeight,
            width: width,
            height: scaledHeight,
          });

          yOffset -= scaledHeight;
        }
      }
    }

    const pdfBytes = await newPdfDoc.save();
    const blob = new Blob([pdfBytes], { type: "application/pdf" });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = "rearranged.pdf";
    link.click();
  }
</script>

<main class="flex flex-col p-4 gap-3 w-screen h-screen">
  <h1 class="justify-start">Guide Weaver</h1>
  {#if pdfImages.length === 0}
    <p>
      Welcome! This tool helps you customize your community group discussion
      guide PDF by splitting questions, rearranging them, and removing ones you
      don't plan to use.
    </p>
    <div class="flex-grow" />
  {/if}

  {#if pdfImages.length > 0}
    {#if splitSections.length > 0}
      <p>
        Great! Now you can drag sections to rearrange them. To remove a section,
        simply drag it to the trash area at the bottom.
      </p>
      <div class="self-center flex flex-row gap-2">
        <button
          class="w-48 bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
          on:click={addNewPage}>Add New Page</button
        >
        <button
          class="w-56 bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
          on:click={generateOutputPdf}>Generate and Download</button
        >
      </div>

      <div class="self-center flex flex-row gap-8">
        {#each outputPages as page, index (page.id)}
          <div class="flex flex-col">
            <h3>
              Page {page.id.split("_")[1]}
              <button
                on:click={() =>
                  (outputPages = outputPages.filter((p, i) => i !== index))}
                class="text-red-500 underline">Delete</button
              >
            </h3>
            <div
              class="border-2 border-green-600 flex-shrink-0 w-[425px] h-[550px]"
            >
              <div
                class="h-full w-full flex flex-col"
                use:dndzone={{ items: page.sections, type: "page-sections" }}
                on:consider={(e) => handleDndConsider(index, e)}
                on:finalize={(e) => handleDndFinalize(index, e)}
              >
                {#each page.sections as section (section.id)}
                  <div class="border border-red-600">
                    <img src={section.image} alt="Section {section.id}" />
                  </div>
                {/each}
              </div>
            </div>
          </div>
        {/each}
      </div>
      <div class="flex-grow" />
      <div class="self-center flex flex-col w-[850px]">
        <h3>Trash</h3>
        <div class="h-20 border-2 border-red-600">
          <div
            class="h-full w-full flex flex-col"
            use:dndzone={{ items: [], type: "page-sections" }}
          />
        </div>
      </div>
    {:else}
      <p>
        Click on the image to place a split line. Click on a split line again to
        remove it. When you're satisfied with your splits, click the "Arrange
        Sections" button below.
      </p>
      <button
        class="self-center w-48 bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105 mb-6"
        on:click={splitAllPages}>Arrange Sections</button
      >
      <div class="self-center flex flex-row gap-4">
        {#each pdfImages as pdfImage, index}
          <div class="relative w-[425px] h-auto">
            <img
              on:click={(e) => addMarker(index, e)}
              on:dragstart|preventDefault
              src={pdfImage}
              alt="PDF page {index + 1}"
            />
            {#each markersByPage[index] || [] as marker (marker.id)}
              <div
                class="absolute left-0 right-0 h-0.5 bg-red-500 cursor-move"
                style="top: {marker.y}px;"
                on:mousedown={(e) => {
                  const startY = e.clientY;
                  const startMarkerY = marker.y;
                  let isDragging = false;

                  function onMouseMove(e) {
                    isDragging = true;
                    const newY = Math.max(
                      0,
                      Math.min(
                        e.clientY - startY + startMarkerY,
                        e.target.offsetParent.clientHeight,
                      ),
                    );
                    updateMarkerPosition(index, marker.id, newY);
                  }

                  function onMouseUp() {
                    window.removeEventListener("mousemove", onMouseMove);
                    window.removeEventListener("mouseup", onMouseUp);
                    if (!isDragging) {
                      markersByPage[index] = markersByPage[index].filter(
                        (m) => m.id !== marker.id,
                      );
                      markersByPage = markersByPage;
                    }
                  }

                  window.addEventListener("mousemove", onMouseMove);
                  window.addEventListener("mouseup", onMouseUp);
                }}
              ></div>
            {/each}
          </div>
        {/each}
      </div>
    {/if}
  {:else}
    <button
      class="self-center w-48 bg-blue-500 hover:bg-blue-600 text-white font-bold py-3 px-6 rounded-lg transition duration-300 ease-in-out transform hover:scale-105 shadow-md"
      on:click={() => document.getElementById("fileInput").click()}
    >
      Upload PDF Guide
      <input
        type="file"
        accept=".pdf"
        on:change={handleFileUpload}
        class="hidden"
        id="fileInput"
      />
    </button>
    <div class="flex-grow" />
  {/if}
</main>
