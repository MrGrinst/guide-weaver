<script>
  import { onMount } from "svelte";

  export let showModal = false;
  export let title = "";

  let modalPosition = { x: 0, y: 0 };
  let isDragging = false;
  let dragStartPosition = { x: 0, y: 0 };
  let el;

  onMount(() => {
    if (el) {
      const rect = el.getBoundingClientRect();
      const windowWidth = window.innerWidth;
      const windowHeight = window.innerHeight;
      modalPosition = {
        x: (windowWidth - rect.width) / 2,
        y: (windowHeight - rect.height) / 2,
      };
    }
  });

  function startDragging(event) {
    isDragging = true;
    dragStartPosition = {
      x: event.clientX - modalPosition.x,
      y: event.clientY - modalPosition.y,
    };
  }

  function stopDragging() {
    isDragging = false;
  }

  function drag(event) {
    if (isDragging) {
      modalPosition = {
        x: event.clientX - dragStartPosition.x,
        y: event.clientY - dragStartPosition.y,
      };
    }
  }
</script>

{#if showModal}
  <div
    class="fixed z-20 inset-0 bg-gray-600 bg-opacity-50 overflow-y-auto h-full w-full"
    on:mousemove={drag}
    on:mouseup={stopDragging}
  >
    <div
      bind:this={el}
      class="bg-gray-600 cursor-move p-5 rounded-lg shadow-xl w-1/2 absolute"
      on:mousedown={startDragging}
      style="transform: translate({modalPosition.x}px, {modalPosition.y}px);"
    >
      <h2 class="text-xl mb-4">{title}</h2>
      <slot></slot>
    </div>
  </div>
{/if}
