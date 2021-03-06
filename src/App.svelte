<script>
  import { onMount } from "svelte";
  import { range, times } from "lodash";
  import Chance from 'chance';
  import Q from './Math.svelte';

  const chance = new Chance(Date.now());

  const CELL_SIZE = 50;

  const imageWidth = 10;
  const imageHeight = 10;
  const image = times(imageHeight, () => times(imageWidth, () => chance.integer({ min: 0, max: 255 })));

  let outputImage = range(imageHeight).map(() => range(imageWidth).map(() => null));

  let outputImageActiveCellPosition = {
    left: 0,
    top: 0,
  };

  function guassian(x, mu, sigma) {
    return Math.exp(-Math.pow((x - mu)/sigma, 2)/2);
  }

  function generateGaussianKernel(radius = 2) {
    const sigma = radius / 2.;
    const row = times(2 * radius + 1, (i) => guassian(i, radius, sigma));
    const output = [];
    for (let i = 0; i < 2 * radius + 1; i++) {
      output.push([]);
      for (let j = 0; j < 2 * radius + 1; j++) {
        output[i][j] = row[i] * row[j];
      }
    }

    return output;
  }

  function generateMeanKernel(radius = 2) {
    return times(2 * radius + 1, () => times(2 * radius + 1, () => 1));
  }

  function generateDummyKernel() {
    return [
      [1, 2, 4, 2, 1],
      [2, 3, 5, 3, 2],
      [4, 5, 6, 5, 4],
      [2, 3, 5, 3, 2],
      [1, 2, 4, 2, 1],
    ]
  }

  const KERNEL_RADIUS = 2;
  const kernel = generateGaussianKernel(KERNEL_RADIUS);
  const KERNEL_WIDTH = 2 * KERNEL_RADIUS + 1;
  const KERNEL_HEIGHT = KERNEL_WIDTH;
  const IMAGE_VERTICAL_SPACING = KERNEL_RADIUS * CELL_SIZE;

  function getUnderKernelImageCells(activeCellPosition) {
    const output = [];
    const rowStart = Math.max(0, activeCellPosition.top - parseInt(KERNEL_HEIGHT / 2));
    const rowEnd = Math.min(imageHeight, parseInt(KERNEL_HEIGHT / 2) + activeCellPosition.top + 1);
    for (let i = rowStart; i < rowEnd; i++) {
      output.push([]);
      const widthStart = Math.max(0, activeCellPosition.left - parseInt(KERNEL_WIDTH / 2));
      const widthEnd = Math.min(imageWidth, parseInt(KERNEL_WIDTH / 2) + activeCellPosition.left + 1);
      for (let j = widthStart; j < widthEnd; j++) {
        output[output.length - 1].push(image[i][j]);
      }
    }

    return output;
  }

  function getActiveKernelCells(activeCellPosition) {
    const output = [];
    for (let i = Math.max(0, parseInt(KERNEL_HEIGHT / 2) - activeCellPosition.top); i < Math.min(KERNEL_HEIGHT, imageHeight + parseInt(KERNEL_HEIGHT / 2) - activeCellPosition.top); i++) {
      output.push([]);
      for (let j = Math.max(0, parseInt(KERNEL_WIDTH / 2) - activeCellPosition.left); j < Math.min(KERNEL_WIDTH, imageWidth + parseInt(KERNEL_WIDTH / 2) - activeCellPosition.left) ; j++) {
        output[output.length - 1].push(kernel[i][j]);
      }
    }

    return output;
  }

  function calculateOutputActiveCell(activeCellPosition) {
    let sum = 0.;
    let total = 0.;
    for (let i = 0; i < underKernelImageCells.length; i++) {
      for (let j = 0; j < underKernelImageCells[i].length; j++) {
        total += underKernelImageCells[i][j] * activeKernelCells[i][j]
        sum += activeKernelCells[i][j];
      }
    }

    outputImage[activeCellPosition.top][activeCellPosition.left] = Math.round(total / sum);
  }

  function handleKeydown(event) {
    const key = event.key;
    if (!['ArrowLeft', 'ArrowRight'].includes(key)) {
      return;
    }

    // left arrow
    if (key === 'ArrowLeft') {
      if (outputImageActiveCellPosition.left > 0) {
        outputImageActiveCellPosition.left -= 1;
      } else {
        if (outputImageActiveCellPosition.top > 0) {
          outputImageActiveCellPosition.top -= 1;
          outputImageActiveCellPosition.left = imageWidth - 1;
        }
      }
    }
    // right arrow
    if (key === 'ArrowRight') {
      if (outputImageActiveCellPosition.left === imageWidth - 1) {
        if (outputImageActiveCellPosition.top === imageHeight - 1) {
          return;
        } else {
          outputImageActiveCellPosition.left = 0;
          outputImageActiveCellPosition.top += 1;
        }
      } else {
        outputImageActiveCellPosition.left += 1;
      }
    }
  }

  onMount(() => {
    document.addEventListener('keydown', handleKeydown);

    return () => {
      document.removeEventListener('keydown', handleKeydown);
    };
  });

  $: kernelPosition = {
      left: outputImageActiveCellPosition.left - parseInt(KERNEL_WIDTH / 2, 10),
      top: outputImageActiveCellPosition.top,
    };
  $: kernelView = kernel.map((row) => row.map((cell) => cell.toFixed(2)));
  $: underKernelImageCells = getUnderKernelImageCells(outputImageActiveCellPosition);
  $: activeKernelCells = getActiveKernelCells(outputImageActiveCellPosition);
  $: activeKernelCellsView = activeKernelCells.map(row => row.map(c => c.toFixed(2)));
  $: calculateOutputActiveCell(outputImageActiveCellPosition);
</script>

<Q imageCells="{underKernelImageCells}" kernelCells="{activeKernelCellsView}" />

<div class="App">
  <div class="image input-image" style="--vertical-spacing: {IMAGE_VERTICAL_SPACING}px;">
    <table class="table image" style="--cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px;">
      <tbody>
        {#each image as row}
          <tr>
            {#each row as cell}
              <td style="background-color: rgb({cell},{cell},{cell}); color: {cell < 128 ? 'white' : 'black'}">{cell}</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>

    <table class="table kernel moving" style="left: {kernelPosition.left * CELL_SIZE}px; top: {kernelPosition.top * CELL_SIZE}px; --cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px">
      <tbody>
        {#each kernel as row}
          <tr>
            {#each row as _}
               <td>&nbsp;</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>
  </div>

  <div class="image output-image">
    <table class="table output" style="--cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px">
      <tbody>
        {#each outputImage as row}
          <tr>
            {#each row as cell}
              <td style="background-color: rgb({cell ?? '255'},{cell ?? '255'},{cell ?? '255'}); color: {cell < 128 ? 'white' : 'black'}; --opacity: {cell === null ? '0' : '1'}">{cell ?? '\b'}</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>

    <table class="table output-active-cell" style="left: {outputImageActiveCellPosition.left * CELL_SIZE}px; top: {outputImageActiveCellPosition.top * CELL_SIZE}px; --cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px">
      <tbody>
        <tr>
          <td>&nbsp;</td>
        </tr>
      </tbody>
    </table>
  </div>

  <table class="table kernel" style="left: {kernelPosition.left * CELL_SIZE}px; top: {kernelPosition.top * CELL_SIZE}px; --cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px">
    <tbody>
      {#each kernelView as row}
        <tr>
          {#each row as cell}
            <td>{cell}</td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  .App {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-around;
    min-height: 100vh;
  }

  .image {
    position: relative;
  }

  .input-image {
    margin-top: calc(var(--vertical-spacing) / 2);
    margin-bottom: calc(var(--vertical-spacing) / 2);
    padding-top: var(--vertical-spacing);
    padding-bottom: var(--vertical-spacing);
  }

  .table {
    border-collapse: collapse;
    table-layout: fixed;
    border-spacing: 0 0;
  }
  .table td {
    border-color: black;
    border-style: solid;
    min-width: var(--cell-width);
    height: var(--cell-height);
    border-width: 1px;
    box-sizing: border-box;
    font-family: Arial, sans-serif;
    font-size: 14px;
    padding: 0;
    opacity: var(--opacity);
    background-color: var(--bg-color);
    transition: background-color 600ms ease-in-out 250ms, opacity 600ms ease-in-out 250ms;
  }

  .table tr {
    text-align: center;
    vertical-align: middle;
  }

  .kernel {
    border: 4px solid hsla(115, 98%, 57%, 0.646);
  }
  .kernel.moving {
    position: absolute;
    top: 0;
    left: 0;
    transition: left 350ms linear, top 350ms linear;
  }

  .kernel td {
    min-width: var(--cell-width);
    min-height: var(--cell-height);
    height: var(--cell-height);
    box-sizing: border-box;
  }

  .output-active-cell {
    background-color: rgba(225, 27, 27, 0.0);
    position: absolute;
    top: 0;
    left: 0;
    transition: left 350ms linear, top 350ms linear;
  }

  .output-active-cell td {
    border-color: hsla(115, 98%, 57%, 0.646);
    border-width: 4px;
    min-width: 47px;
    line-height: 43px;
    box-sizing: border-box;
  }
</style>
