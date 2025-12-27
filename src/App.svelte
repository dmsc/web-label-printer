<script lang="ts">
  // Width and height in 0.125 mm units
  let label_width = $state(40 * 8);
  let label_height = $state(12 * 8);
  let margin = $state(1 * 8);
  let text = $state("");
  let canvas: HTMLCanvasElement | undefined = $state();

  function sizeLine(ctx: CanvasRenderingContext2D, str: string) {
    var m = ctx.measureText(str);
    //    var hg = m.fontBoundingBoxAscent + m.fontBoundingBoxDescent;
    var hg = m.actualBoundingBoxAscent + m.actualBoundingBoxDescent;
    var wd = m.actualBoundingBoxLeft + m.actualBoundingBoxRight;
    return { wd, hg };
  }

  function fullSize(
    ctx: CanvasRenderingContext2D,
    lines: string[],
    sz: number,
  ) {
    ctx.font = sz.toString() + "px sans";
    var wd = 0;
    var hg = 0;
    for (var line of lines) {
      var s = sizeLine(ctx, line);
      hg = Math.max(hg, s.hg);
      wd = Math.max(wd, s.wd);
    }
    return { wd, hg, total_hg: (1.05 * hg + 2) * lines.length - hg * 0.05 - 2 };
  }

  function bestSize(
    ctx: CanvasRenderingContext2D,
    lines: string[],
    min_size = 8,
    max_size = 96,
  ) {
    var wd = ctx.canvas.width - margin * 2;
    var hg = ctx.canvas.height - margin * 2;

    while (min_size + 0.1 < max_size) {
      var size = (min_size + max_size) / 2;
      var s = fullSize(ctx, lines, size);
      if (s.wd >= wd || s.total_hg >= hg) {
        max_size = size;
      } else {
        min_size = size;
      }
    }
    return min_size;
  }

  function drawCanvas(text: string) {
    if (!canvas) return;
    var ctx = canvas.getContext("2d");
    if (!ctx) return;

    ctx.fillStyle = "#eee";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // Draw text lines:
    var lines = text.split("\n");
    var sz = bestSize(ctx, lines);
    var font_sz = fullSize(ctx, lines, sz);
    var line_height = font_sz.hg * 1.05 + 2;

    ctx.textBaseline = "alphabetic";
    ctx.fillStyle = "#000";
    var ln = 0;
    for (var line of lines) {
      var m = ctx.measureText(line);
      var y =
        canvas.height / 2 +
        m.actualBoundingBoxAscent / 2 +
        line_height * (ln - (lines.length - 1) / 2);
      var w = m.actualBoundingBoxLeft + m.actualBoundingBoxRight;
      var x = (canvas.width - w) / 2 + m.actualBoundingBoxLeft;
      ctx.fillText(line, x, y);
      ln++;
    }
  }

  $effect(() => {
    drawCanvas(text);
  });
</script>

<main>
  <h1>Web Label Printer</h1>
  <div class="r">
    <textarea bind:value={text} placeholder="Write text here..."></textarea>
    <button>Print Label</button>
    <div class="l">
      <canvas bind:this={canvas} width={label_width} height={label_height}>
      </canvas>
      <span>Label size: {label_width * 0.125} × {label_height * 0.125} mm</span>
      <span>Margin: {margin * 0.125} mm</span>
    </div>
  </div>
</main>

<style>
  h1 {
    font-size: 3.2em;
    line-height: 1.1;
    border-bottom: 1px solid #444;
  }
  div.r {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: space-between;
    gap: 2em;
  }
  .r > * {
    display: block;
  }
  .l {
    width: 100%;
  }
  .l > * {
    display: block;
    width: 100%;
  }
  .l span {
    color: #888;
    text-align: left;
  }
  textarea,
  button,
  canvas {
    box-sizing: border-box;
    width: 100%;
    border: 2px solid black;
    border-radius: 12px;
  }
  textarea {
    field-sizing: content;
    min-height: 1lh;
    max-height: 10lh;
    overflow: hidden;
    resize: none;
    font-size: 24px;
    padding: 8px;
    background-color: #ccf;
    transition: background-color 0.25s ease-in-out;
  }
  textarea:hover {
    background-color: #ddf;
  }
  canvas {
    border-radius: 32px;
  }
  button {
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    background-color: #ddd;
    cursor: pointer;
    transition: background-color 0.25s ease-in-out;
  }
  button:hover {
    background-color: #ccc;
  }
</style>
