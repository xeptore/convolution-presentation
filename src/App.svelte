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

  const kernelRadius = 2;
  const kernel = generateGaussianKernel(kernelRadius);
  const kernelWidth = 2 * kernelRadius + 1;
  const kernelHeight = kernelWidth;
  const IMAGE_VERTICAL_SPACING = kernelRadius * CELL_SIZE;

  function getUnderKernelImageCells(activeCellPosition) {
    const output = [];
    const rowStart = Math.max(0, activeCellPosition.top - parseInt(kernelHeight / 2));
    const rowEnd = Math.min(imageHeight, parseInt(kernelHeight / 2) + activeCellPosition.top + 1);
    for (let i = rowStart; i < rowEnd; i++) {
      output.push([]);
      const widthStart = Math.max(0, activeCellPosition.left - parseInt(kernelWidth / 2));
      const widthEnd = Math.min(imageWidth, parseInt(kernelWidth / 2) + activeCellPosition.left + 1);
      for (let j = widthStart; j < widthEnd; j++) {
        output[output.length - 1].push(image[i][j]);
      }
    }

    return output;
  }

  function getActiveKernelCells(activeCellPosition) {
    const output = [];
    for (let i = Math.max(0, parseInt(kernelHeight / 2) - activeCellPosition.top); i < Math.min(kernelHeight, imageHeight + parseInt(kernelHeight / 2) - activeCellPosition.top); i++) {
      output.push([]);
      for (let j = Math.max(0, parseInt(kernelWidth / 2) - activeCellPosition.left); j < Math.min(kernelWidth, imageWidth + parseInt(kernelWidth / 2) - activeCellPosition.left) ; j++) {
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
      left: outputImageActiveCellPosition.left - parseInt(kernelWidth / 2, 10),
      top: outputImageActiveCellPosition.top,
    };
  $: underKernelImageCells = getUnderKernelImageCells(outputImageActiveCellPosition);
  $: activeKernelCells = getActiveKernelCells(outputImageActiveCellPosition);
  $: calculateOutputActiveCell(outputImageActiveCellPosition);
</script>


<Q imageCells="{underKernelImageCells}" kernelCells="{activeKernelCells}" />
<div class="App">
  <div class="image input-image" style="--vertical-spacing: {IMAGE_VERTICAL_SPACING}px;">
    <table class="table image" style="--cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px;">
      <tbody>
        {#each image as row}
          <tr>
            {#each row as cell}
              <td style="background-color: rgb({cell},{cell},{cell})">{cell}</td>
            {/each}
          </tr>
        {/each}
      </tbody>
    </table>

    <table class="table kernel" style="left: {kernelPosition.left * CELL_SIZE}px; top: {kernelPosition.top * CELL_SIZE}px; --cell-width: {CELL_SIZE}px; --cell-height: {CELL_SIZE}px">
      <tbody>
        {#each kernel as row}
          <tr>
            {#each row as cell}
               <td>{String(cell).substring(0, 4)}</td>
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
              <td style="background-color: rgb({cell ?? '255'},{cell ?? '255'},{cell ?? '255'}); --opacity: {cell === null ? '0' : '1'}">{cell ?? '\b'}</td>
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
    border-spacing: 0;
    line-height: calc(var(--cell-height) - 1px);
  }
  .table td {
    border-color: black;
    border-style: solid;
    min-width: calc(var(--cell-width) - 1px);
    border-width: 1px;
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
    background-color: rgba(0, 0, 0, 0.375);
    position: absolute;
    top: 0;
    left: 0;
    transition: left 350ms linear, top 350ms linear;
  }

  .output-active-cell {
    background-color: rgba(225, 27, 27, 0.0);
    position: absolute;
    top: 0;
    left: 0;
    transition: left 350ms linear, top 350ms linear;
  }

  .output-active-cell td {
    border-width: 4px;
    min-width: 48px;
    line-height: 44px;
    box-sizing: border-box;
  }
</style>
