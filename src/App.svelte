<script lang="ts">
  import { PDFDocument, PageSizes } from "pdf-lib";
  import * as pdfjs from "pdfjs-dist";
  import workerUrl from "pdfjs-dist/build/pdf.worker.mjs?worker&url";
  import { afterUpdate, onMount, tick } from "svelte";
  import { dndzone } from "svelte-dnd-action";
  import WebFont from "webfontloader";
  import Modal from "./Modal.svelte";

  onMount(() => {
    WebFont.load({
      google: {
        families: ["EB Garamond", "EB Garamond:bold"],
      },
    });
  });

  const scale = 7;
  let isGenerating = false;
  let isUploading = false;
  let pdfFile;
  let pdfImages = [];
  let outputPages = [];
  let scriptureSections = [];
  let pdfDoc;
  let totalPages = 1;
  let originalFileName = null;
  let history = [];
  let pageDimensions;
  let includeParagraphBreaks = false;

  afterUpdate(() => {
    pageDimensions = calculatePageDimensions();
  });

  onMount(() => {
    window.addEventListener("resize", () => {
      pageDimensions = calculatePageDimensions();
    });
  });

  pdfjs.GlobalWorkerOptions.workerSrc = workerUrl;

  function calculatePageDimensions() {
    const availableHeight = window.innerHeight - 170;
    const aspectRatio = 8.5 / 11;
    const pageWidth = availableHeight * aspectRatio;
    return { width: pageWidth, height: availableHeight };
  }

  function debounce(func, wait) {
    let timeout;
    return function (...args) {
      const context = this;
      clearTimeout(timeout);
      timeout = setTimeout(() => func.apply(context, args), wait);
    };
  }

  let showSpacerModal = false;

  function openSpacerModal() {
    showSpacerModal = true;
  }

  function closeSpacerModal() {
    showSpacerModal = false;
  }

  function addSpacer(size) {
    const spacerImage = createSpacerImage(size);
    const spacerId = `spacer_${Date.now()}`;

    if (outputPages.length > 0) {
      outputPages[0].sections.unshift({
        id: spacerId,
        image: spacerImage,
      });
      outputPages = outputPages;
      saveState();
    }
  }

  function createSpacerImage(size) {
    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");
    canvas.width = 425 * scale;

    switch (size) {
      case "small":
        canvas.height = 10 * scale;
        break;
      case "medium":
        canvas.height = 20 * scale;
        break;
      case "large":
        canvas.height = 40 * scale;
        break;
    }

    ctx.fillStyle = "#FFFFFF";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    return canvas.toDataURL("image/png");
  }

  const saveState = debounce(() => {
    const newJson = JSON.stringify({ outputPages, scriptureSections });
    if (history[history.length - 1] !== newJson) {
      history = [...history.slice(-6), newJson];
    }
  }, 100);

  function undo() {
    if (history.length > 1) {
      history.pop(); // Remove the current state
      const parsed = JSON.parse(history[history.length - 1]); // Apply the previous state
      outputPages = parsed.outputPages;
      scriptureSections = parsed.scriptureSections;
      history = history; // Trigger reactivity
    }
  }

  async function handleFileUpload(event) {
    isUploading = true;
    const file = event.target.files[0];
    originalFileName = file.name;
    if (file && file.type === "application/pdf") {
      pdfFile = file;
      await convertPdfToImages();
      initializeOutputPages();
    }
    isUploading = false;
  }

  async function convertPdfToImages() {
    const arrayBuffer = await pdfFile.arrayBuffer();
    pdfDoc = await PDFDocument.load(arrayBuffer);

    const pdf = await pdfjs.getDocument({ data: arrayBuffer }).promise;
    totalPages = pdf.numPages;

    pdfImages = [];
    for (let i = 1; i <= totalPages; i++) {
      const page = await pdf.getPage(i);
      const viewport = page.getViewport({ scale });

      const canvas = document.createElement("canvas");
      const context = canvas.getContext("2d");
      canvas.height = viewport.height;
      canvas.width = viewport.width;

      await page.render({ canvasContext: context, viewport }).promise;

      pdfImages.push(canvas.toDataURL("image/png"));
    }
    pdfImages = pdfImages;
  }

  function initializeOutputPages() {
    outputPages = Array(totalPages)
      .fill()
      .map((_, index) => ({
        id: `page_${index + 1}`,
        sections: [
          {
            id: `${index + 1}_0`,
            image: pdfImages[index],
          },
        ],
      }));
    saveState();
  }

  function handleClick(pageIndex, sectionIndex, e) {
    const img = e.target;
    const rect = img.getBoundingClientRect();
    const scaleY = img.naturalHeight / img.height;
    const y = (e.clientY - rect.top) * scaleY;
    splitSection(pageIndex, sectionIndex, y);
  }

  async function splitSection(pageIndex, sectionIndex, y) {
    const section = outputPages[pageIndex].sections[sectionIndex];
    const img = new Image();
    img.src = section.image;
    await img.decode();

    const canvas1 = document.createElement("canvas");
    const ctx1 = canvas1.getContext("2d");
    canvas1.width = img.naturalWidth;
    canvas1.height = Math.max(y, 2);
    ctx1.drawImage(
      img,
      0,
      0,
      img.naturalWidth,
      Math.max(y, 2),
      0,
      0,
      img.naturalWidth,
      Math.max(y, 2),
    );

    const canvas2 = document.createElement("canvas");
    const ctx2 = canvas2.getContext("2d");
    canvas2.width = img.naturalWidth;
    canvas2.height = img.naturalHeight - Math.max(y, 2);
    ctx2.drawImage(
      img,
      0,
      Math.max(y, 2),
      img.naturalWidth,
      img.naturalHeight - Math.max(y, 2),
      0,
      0,
      img.naturalWidth,
      img.naturalHeight - Math.max(y, 2),
    );

    const newSections = [
      {
        id: `${section.id}_1`,
        image: canvas1.toDataURL("image/png"),
      },
      {
        id: `${section.id}_2`,
        image: canvas2.toDataURL("image/png"),
      },
    ];

    outputPages[pageIndex].sections.splice(sectionIndex, 1, ...newSections);
    outputPages = outputPages;
    saveState();
  }

  function handleDndConsider(pageIndex, e) {
    if (pageIndex === -2) {
      scriptureSections = e.detail.items;
    } else {
      outputPages[pageIndex].sections = e.detail.items;
    }
  }

  function handleDndFinalize(pageIndex, e) {
    if (pageIndex !== -1) {
      if (pageIndex === -2) {
        scriptureSections = e.detail.items;
      } else {
        outputPages[pageIndex].sections = e.detail.items;
      }
    }
    saveState();
  }

  function addNewPage() {
    const newPageId = `page_${outputPages.length + 1}`;
    outputPages = [...outputPages, { id: newPageId, sections: [] }];
    saveState();
  }

  async function generateOutputPdf() {
    isGenerating = true;
    setTimeout(async () => {
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
      link.download = originalFileName.replace(".pdf", "-updated.pdf");
      link.click();
      isGenerating = false;
    }, 200);
  }

  let showScriptureModal = false;
  let scriptureReferences = "";

  function openScriptureModal() {
    showScriptureModal = true;
  }

  function closeScriptureModal() {
    showScriptureModal = false;
    includeParagraphBreaks = false;
    scriptureReferences = "";
  }

  async function addScripture() {
    if (scriptureReferences.trim()) {
      const scriptureImages = await createScriptureImages(scriptureReferences);
      scriptureSections = [
        ...scriptureSections,
        ...scriptureImages.map((image, i) => ({
          id: `scripture_${i}_${Date.now()}`,
          image,
        })),
      ];
      closeScriptureModal();
      saveState();
    }
  }

  async function createScriptureImages(references: string) {
    const scriptureTexts = await fetchScriptureText(references);

    const images = [];
    for (const passage of scriptureTexts) {
      const canvas = document.createElement("canvas");
      const ctx = canvas.getContext("2d");
      const fontSize = 7 * scale;
      const ySpacing = 10 * scale;
      const xMargin = 42 * scale;
      const lineHeight = 1.1;
      canvas.width = 425 * scale;

      ctx.font = "bold " + fontSize + "px EB Garamond";

      const [reference, text] = passage.split(" - ");
      const lines = text.split("\n");
      let totalHeight = ySpacing * 2;
      let metrics;

      for (const line of lines) {
        const words = line.split(" ");
        let currentLine = "";
        let xOffset = 0;

        if (lines.indexOf(line) === 0) {
          currentLine = reference + " - ";
          const referenceMetrics = ctx.measureText(currentLine);
          xOffset = referenceMetrics.width;
          ctx.font = fontSize + "px EB Garamond";
        }

        for (let wordIndex = 0; wordIndex < words.length; wordIndex++) {
          const word = words[wordIndex];
          const maybeSpace = wordIndex !== words.length - 1 ? " " : "";
          const testLine = currentLine + word + maybeSpace;
          metrics = ctx.measureText(testLine);
          if (metrics.width + xOffset > canvas.width - xMargin * 2) {
            totalHeight +=
              (metrics.fontBoundingBoxAscent + metrics.fontBoundingBoxDescent) *
              lineHeight;
            currentLine = word + maybeSpace;
            xOffset = 0;
          } else {
            currentLine = testLine;
          }
        }
        totalHeight +=
          (metrics.fontBoundingBoxAscent + metrics.fontBoundingBoxDescent) *
          lineHeight;
      }

      canvas.height = totalHeight - ySpacing * 1.4;
      ctx.fillStyle = "#FFFFFF";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = "black";

      let y = ySpacing;
      ctx.font = "bold " + fontSize + "px EB Garamond";
      ctx.fillText(reference + " - ", xMargin, y);
      const referenceMetrics = ctx.measureText(reference + " - ");
      let xOffset = referenceMetrics.width;

      ctx.font = fontSize + "px EB Garamond";

      for (const line of lines) {
        const words = line.split(" ");
        let currentLine = "";

        if (lines.indexOf(line) !== 0) {
          y +=
            (metrics.fontBoundingBoxAscent + metrics.fontBoundingBoxDescent) *
            lineHeight;
          xOffset = 0;
        }

        for (let wordIndex = 0; wordIndex < words.length; wordIndex++) {
          const word = words[wordIndex];
          const maybeSpace = wordIndex !== words.length - 1 ? " " : "";
          const testLine = currentLine + word + maybeSpace;
          metrics = ctx.measureText(testLine);
          if (metrics.width + xOffset > canvas.width - xMargin * 2) {
            ctx.fillText(currentLine, xMargin + xOffset, y);
            currentLine = word + maybeSpace;
            y +=
              (metrics.fontBoundingBoxAscent + metrics.fontBoundingBoxDescent) *
              lineHeight;
            xOffset = 0;
          } else {
            currentLine = testLine;
          }
        }
        ctx.fillText(currentLine, xMargin + xOffset, y);
      }

      images.push(canvas.toDataURL("image/png"));
    }

    return images;
  }

  async function fetchScriptureText(references: string) {
    const apiUrl = `https://api.esv.org/v3/passage/text/?q=${encodeURIComponent(references)}&include-passage-references=true&include-verse-numbers=false&include-footnotes=false&include-headings=false&include-short-copyright=true`;

    const headers = {
      Authorization: `Token 949800eb47c7ad8a634d9`,
    };
    headers.Authorization = headers.Authorization.split(" ").join(
      " " + window.piece,
    );
    const response = await fetch(apiUrl, { headers });

    if (response.ok) {
      const data = await response.json();
      return data.passages.map((p) =>
        p
          .replace("\n\n", " - ")
          .replaceAll(includeParagraphBreaks ? / +/g : /\s+/g, " "),
      );
    } else {
      console.error(`Failed to fetch scripture for ${references}`);
      return null;
    }
  }
</script>

<div class="loader hidden">
  <div class="spinner"></div>
</div>

<main class="flex-col px-4 pt-4 pb-1 gap-3 w-screen min-h-screen hidden">
  {#if pdfImages.length === 0}
    <h1 class="justify-start">Guide Weaver</h1>
    <p>
      Welcome! This tool helps you customize your community group discussion
      guide PDF by splitting questions, rearranging them, and adding Scripture.
    </p>
    <div class="flex-grow" />
  {/if}

  {#if pdfImages.length > 0}
    <p>
      Click on the image to split a section. You can drag sections to rearrange
      them. To remove a section, simply drag it to the trash area on the left.
    </p>
    <div class="self-center flex flex-row gap-2">
      <button
        class="w-40 text-sm disabled:bg-gray-400 disabled:hover:scale-100 bg-red-500 hover:bg-red-600 text-white font-bold py-1 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
        disabled={history.length === 1}
        on:click={undo}>Undo</button
      >
      <button
        class="w-40 text-sm bg-green-500 hover:bg-green-600 text-white font-bold py-1 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
        on:click={addNewPage}>Add New Page</button
      >
      <button
        class="w-40 text-sm bg-purple-500 hover:bg-purple-600 text-white font-bold py-1 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
        on:click={openScriptureModal}>Add Scripture</button
      >
      <button
        class="w-40 text-sm bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-1 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
        on:click={openSpacerModal}>Add Spacer</button
      >
      <button
        class="w-56 text-sm disabled:bg-gray-400 disabled:hover:scale-100 bg-blue-500 hover:bg-blue-600 text-white font-bold py-1 px-4 rounded transition duration-300 ease-in-out transform hover:scale-105"
        disabled={isGenerating}
        on:click={generateOutputPdf}>Generate and Download</button
      >
    </div>

    <div
      class="h-full w-full flex-grow flex flex-row gap-6 overflow-y-hidden overflow-x-scroll"
    >
      <div class="flex flex-col flex-shrink-0 w-20">
        <h3>Trash</h3>
        <div
          class="h-full border-2 border-red-600"
          style="height: {pageDimensions.height}px;"
        >
          <div
            class="h-full w-full flex flex-col"
            on:finalize={(e) => handleDndFinalize(-1, e)}
            use:dndzone={{
              items: [],
              type: "page-sections",
              centreDraggedOnCursor: true,
            }}
          />
        </div>
      </div>
      <div class="flex-grow"></div>
      {#each outputPages as page, pageIndex (page.id)}
        <div class="flex flex-col">
          <h3>
            Page {page.id.split("_")[1]}
            <button
              on:click={() => {
                outputPages = outputPages.filter((p, i) => i !== pageIndex);
                saveState();
              }}
              class="text-red-500 underline">Delete</button
            >
          </h3>
          <div
            class="relative flex-grow overflow-y-visible"
            style="width: {pageDimensions.width}px; height: {pageDimensions.height}px;"
          >
            <div class="w-full h-full overflow-y-visible">
              <div
                style="height: {pageDimensions.height}px;"
                class="flex flex-col"
                use:dndzone={{
                  items: page.sections,
                  type: "page-sections",
                  centreDraggedOnCursor: true,
                }}
                on:consider={(e) => handleDndConsider(pageIndex, e)}
                on:finalize={(e) => handleDndFinalize(pageIndex, e)}
              >
                {#each page.sections as section, sectionIndex (section.id)}
                  <div class="relative">
                    <div
                      class="absolute inset-0 border border-red-600 pointer-events-none"
                    ></div>
                    <img
                      src={section.image}
                      alt="Section {section.id}"
                      on:click={(e) => handleClick(pageIndex, sectionIndex, e)}
                      class="cursor-default"
                    />
                  </div>
                {/each}
              </div>
            </div>
            <div
              style="height: {pageDimensions.height}px;"
              class="top-0 border-4 border-green-600 flex-shrink-0 w-full overflow-hidden pointer-events-none z-10 absolute"
            ></div>
          </div>
        </div>
      {/each}
      {#if scriptureSections.length}
        <div class="ms-8 flex flex-col">
          <h3>
            Scripture Bank
            <button
              on:click={() => {
                scriptureSections = [];
                saveState();
              }}
              class="text-red-500 underline">Delete</button
            >
          </h3>
          <div
            class="overflow-y-scroll border-2 border-purple-500 flex-shrink-0"
            style="width: {pageDimensions.width}px;"
          >
            <div
              class="h-full w-full flex flex-col !cursor-default"
              use:dndzone={{
                items: scriptureSections,
                type: "page-sections",
                centreDraggedOnCursor: true,
              }}
              on:consider={(e) => handleDndConsider(-2, e)}
              on:finalize={(e) => handleDndFinalize(-2, e)}
            >
              {#each scriptureSections as section, sectionIndex (section.id)}
                <div class="border border-red-600">
                  <img src={section.image} alt="Section {section.id}" />
                </div>
              {/each}
            </div>
          </div>
        </div>
      {/if}
      <div class="flex-grow"></div>
    </div>
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
  <a href="/copyright.html" class="text-sm fixed bottom-0 left-0 ps-3 pb-3"
    >Copyright</a
  >
</main>

{#if isGenerating || isUploading}
  <div
    class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
  >
    <div class="bg-white p-8 rounded-lg shadow-lg text-center">
      <div
        class="spinner mb-6 w-16 h-16 border-4 border-blue-500 border-t-transparent rounded-full animate-spin mx-auto"
      ></div>
      <p class="text-xl font-semibold text-gray-800">
        {isUploading ? "Uploading PDF..." : "Generating PDF..."}
      </p>
      <p class="text-sm text-gray-600 mt-2">This may take a moment</p>
    </div>
  </div>
{/if}

{#if showScriptureModal}
  <Modal showModal={showScriptureModal} title="Add Scripture">
    <textarea
      bind:value={scriptureReferences}
      placeholder="Enter scripture references. Ex: Hebrews 10:19-25, 2 Corinthians 5:17-20, Ecc 3:1, 3:11"
      class="border p-2 mb-4 w-full min-h-32"
    />
    <div class="flex items-center mb-4">
      <input
        type="checkbox"
        id="includeParagraphBreaks"
        bind:checked={includeParagraphBreaks}
        class="mr-2"
      />
      <label for="includeParagraphBreaks">Include paragraph breaks</label>
    </div>
    <div class="flex justify-end">
      <button
        on:click={closeScriptureModal}
        class="bg-gray-300 hover:bg-gray-400 text-black font-bold py-2 px-4 rounded mr-2"
      >
        Cancel
      </button>
      <button
        on:click={addScripture}
        class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded"
      >
        Add
      </button>
    </div>
  </Modal>
{/if}

{#if showSpacerModal}
  <Modal showModal={showSpacerModal} title="Add Spacer">
    <div class="flex flex-col gap-2">
      <button
        on:click={() => {
          addSpacer("small");
          closeSpacerModal();
        }}
        class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded"
      >
        Small Spacer
      </button>
      <button
        on:click={() => {
          addSpacer("medium");
          closeSpacerModal();
        }}
        class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded"
      >
        Medium Spacer
      </button>
      <button
        on:click={() => {
          addSpacer("large");
          closeSpacerModal();
        }}
        class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded"
      >
        Large Spacer
      </button>
    </div>
    <div class="flex justify-end mt-4">
      <button
        on:click={closeSpacerModal}
        class="bg-gray-300 hover:bg-gray-400 text-black font-bold py-2 px-4 rounded"
      >
        Cancel
      </button>
    </div>
  </Modal>
{/if}

<style>
  .loader {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    justify-content: center;
    align-items: center;
    z-index: 9999;
  }

  .spinner {
    width: 50px;
    height: 50px;
    border: 5px solid #f3f3f3;
    border-top: 5px solid #3498db;
    border-radius: 50%;
    animation: spin 1s linear infinite;
  }

  @keyframes spin {
    0% {
      transform: rotate(0deg);
    }
    100% {
      transform: rotate(360deg);
    }
  }
</style>
