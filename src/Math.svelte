<script>
  import { parse } from "mathjs";

  export let imageCells = [];
  export let kernelCells = [];

  function mj(tex) {
    return MathJax.tex2svg(tex, { em: 160, ex: 60, display: false });
  }

  let result;

  function redrawSVG(expr) {
    try {
      // parse the expression
      const node = parse(expr);

      // export the expression to LaTeX
      const latex = node
        ? node.toTex({ parenthesis: "auto", implicit: "show" })
        : "";
      console.log("LaTeX expression:", latex);

      // display and re-render the expression
      MathJax.typesetClear();
      result = mj(latex).innerHTML;
    } catch (err) {
      console.error(err)
    }
  }

  $: expr = '(' + imageCells.map((row, i) => row.map((cell, j) => `${cell}*${String(kernelCells[i][j]).substring(0, 4)}`).join('+')).join('+') + ')/(' + kernelCells.map((row) => row.map((c) => c.toFixed(2)).join('+')).join('+') + ')'
  $: redrawSVG(expr);
</script>

<pre>
  <code>
    {@html result}
  </code>
</pre>
