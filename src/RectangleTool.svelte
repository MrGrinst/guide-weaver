<script lang="ts">
  import { createEventDispatcher, onMount } from "svelte";

  export let image: string;

  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let isDrawing = false;
  let startX: number, startY: number;
  let scaleFactor: number;
  let tempCanvas: HTMLCanvasElement;
  let tempCtx: CanvasRenderingContext2D;
  const imageForCanvas = new Image();

  const dispatch = createEventDispatcher();

  function handleMouseDown(event: MouseEvent) {
    isDrawing = true;
    const rect = canvas.getBoundingClientRect();
    startX = event.clientX - rect.left;
    startY = event.clientY - rect.top;
  }

  function handleMouseMove(event: MouseEvent) {
    if (!isDrawing) return;

    const rect = tempCanvas.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const y = event.clientY - rect.top;

    const scaledStartX = startX * scaleFactor;
    const scaledStartY = startY * scaleFactor;
    const scaledX = x * scaleFactor;
    const scaledY = y * scaleFactor;

    tempCtx.clearRect(0, 0, tempCanvas.width, tempCanvas.height);
    tempCtx.fillStyle = "white";
    tempCtx.fillRect(
      scaledStartX,
      scaledStartY,
      scaledX - scaledStartX,
      scaledY - scaledStartY,
    );
  }

  function handleMouseUp() {
    if (!isDrawing) return;
    isDrawing = false;
    ctx.drawImage(tempCanvas, 0, 0);
    tempCtx.clearRect(0, 0, tempCanvas.width, tempCanvas.height);
    dispatch("updateImage", canvas.toDataURL("image/png"));
  }

  onMount(() => {
    ctx = canvas.getContext("2d");
    tempCtx = tempCanvas.getContext("2d");
    imageForCanvas.onload = () => {
      canvas.width = tempCanvas.width = imageForCanvas.width;
      canvas.height = tempCanvas.height = imageForCanvas.height;
      ctx.drawImage(imageForCanvas, 0, 0);
      scaleFactor = canvas.width / canvas.offsetWidth;
    };
    imageForCanvas.src = image;
  });
</script>

<div style="position: relative;">
  <canvas bind:this={canvas} style="max-width: 100%; height: auto;"></canvas>
  <canvas
    bind:this={tempCanvas}
    on:mousedown={handleMouseDown}
    on:mousemove={handleMouseMove}
    on:mouseup={handleMouseUp}
    on:mouseleave={handleMouseUp}
    style="position: absolute; top: 0; left: 0; max-width: 100%; height: auto;"
  ></canvas>
</div>

<style>
  canvas {
    cursor: crosshair;
  }
</style>
