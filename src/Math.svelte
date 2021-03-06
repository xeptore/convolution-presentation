<script>
  import { parse, format } from "mathjs";

  export let imageCells = [];
  export let kernelCells = [];

  function mj(tex) {
    return MathJax.tex2svg(tex, { em: 160, ex: 60, display: false });
  }

  let equiation = "";
  let result = "0.0";

  function redrawSVG(expr) {
    console.debug(expr);
    try {
      // export the expression to LaTeX
      result = format(parse(expr).compile().evaluate());

      // display and re-render the expression
      MathJax.typesetClear();
      equiation = mj(
        parse(expr).toTex({ parenthesis: "auto", implicit: "show" }) +
          "=" +
          parseFloat(result).toFixed(2) +
          "≃" +
          Math.round(parseFloat(result))
      ).innerHTML;
    } catch (err) {
      console.error(err);
    }
  }

  $: expr =
    "(" +
    imageCells
      .map((row, i) =>
        row
          .map(
            (cell, j) => `${cell}*${kernelCells[i][j]}`
          )
          .join("+")
      )
      .join("+") +
    ")/(" +
    kernelCells.map((row) => row.join("+")).join("+") +
    ")";
  $: redrawSVG(expr);
</script>

<div class="equiation">
  {@html equiation}
</div>

<style>
  .equiation {
    width: 100%;
    text-align: center;
    margin-top: 2rem;
    padding-bottom: 1rem;
    overflow-x: scroll;
    font-size: xx-large;
  }
</style>
