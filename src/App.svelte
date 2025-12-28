<script lang="ts">
  type CanvasContext =
    | CanvasRenderingContext2D
    | OffscreenCanvasRenderingContext2D;

  // Dimensions in mm (loaded from localStorage)
  let label_width_mm = $state(
    parseFloat(localStorage.getItem("label_width_mm") || "40"),
  );
  let label_height_mm = $state(
    parseFloat(localStorage.getItem("label_height_mm") || "12"),
  );
  let margin_mm = $state(parseFloat(localStorage.getItem("margin_mm") || "1"));

  // Width and height in 0.125 mm units (derived, rounded to integers)
  let label_width = $derived(Math.round(label_width_mm * 8));
  let label_height = $derived(Math.round(label_height_mm * 8));
  let margin = $derived(Math.round(margin_mm * 8));
  let text = $state(sessionStorage.getItem("text") || "");

  // Save to localStorage
  $effect(() => {
    localStorage.setItem("label_width_mm", label_width_mm.toString());
    localStorage.setItem("label_height_mm", label_height_mm.toString());
    localStorage.setItem("margin_mm", margin_mm.toString());
  });

  // Save text to sessionStorage
  $effect(() => {
    sessionStorage.setItem("text", text);
  });
  let canvas: HTMLCanvasElement | undefined = $state();
  let pixelData = new Uint8Array();

  function sizeLine(ctx: CanvasContext, str: string) {
    var m = ctx.measureText(str);
    //    var hg = m.fontBoundingBoxAscent + m.fontBoundingBoxDescent;
    var hg = m.actualBoundingBoxAscent + m.actualBoundingBoxDescent;
    var wd = m.actualBoundingBoxLeft + m.actualBoundingBoxRight;
    return { wd, hg };
  }

  function fullSize(ctx: CanvasContext, lines: string[], sz: number) {
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
    ctx: CanvasContext,
    lines: string[],
    min_size = 8,
    max_size = 144,
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

  function drawText(ctx: CanvasContext, text: string) {
    ctx.fillStyle = "#fff";
    ctx.fillRect(0, 0, label_width, label_height);

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
        label_height / 2 +
        m.actualBoundingBoxAscent / 2 +
        line_height * (ln - (lines.length - 1) / 2);
      var w = m.actualBoundingBoxLeft + m.actualBoundingBoxRight;
      var x = (label_width - w) / 2 + m.actualBoundingBoxLeft;
      ctx.fillText(line, x, y);
      ln++;
    }
  }

  function convertToData(ctx: CanvasContext) {
    // Get image data
    const imageData = ctx.getImageData(0, 0, label_width, label_height);

    // Dither pattern
    const dither = (x: number, y: number) => {
      const pattern = [
        [24, 406, 120, 502],
        [598, 215, 693, 311],
        [167, 550, 72, 454],
        [741, 359, 645, 263],
      ];
      const val =
        imageData.data[4 * (x + y * label_width)] + // red
        imageData.data[4 * (x + y * label_width) + 1] + // green
        imageData.data[4 * (x + y * label_width) + 2]; // blue
      return val > pattern[x % 4][y % 4] ? 0 : 1;
    };

    // Dither to binary image data, in the printer format
    const rows = Math.floor((label_height + 7) / 8);
    const pixels = new Uint8Array(rows * label_width);
    for (let x = 0, pos = 0; x < label_width; x++) {
      for (let y = 0; y < label_height; y += 8) {
        const val =
          dither(x, y) * 128 +
          dither(x, y + 1) * 64 +
          dither(x, y + 2) * 32 +
          dither(x, y + 3) * 16 +
          dither(x, y + 4) * 8 +
          dither(x, y + 5) * 4 +
          dither(x, y + 6) * 2 +
          dither(x, y + 7);
        pixels[pos] = val;
        pos++;
      }
    }

    return pixels;
  }

  function draw() {
    if (!canvas) return;

    const offscreenCanvas = new OffscreenCanvas(label_width, label_height);
    const off_ctx = offscreenCanvas.getContext("2d");
    if (!off_ctx) return;

    drawText(off_ctx, text);
    pixelData = convertToData(off_ctx);

    // Convert back to image data for preview
    const img = new ImageData(label_width, label_height);

    const setPix = (x: number, y: number, v: number) => {
      const p = v ? 0 : 255;
      img.data[(x + y * label_width) * 4 + 0] = p;
      img.data[(x + y * label_width) * 4 + 1] = p;
      img.data[(x + y * label_width) * 4 + 2] = p;
      img.data[(x + y * label_width) * 4 + 3] = 255;
    };

    for (let x = 0, pos = 0; x < label_width; x++) {
      for (let y = 0; y < label_height; y += 8) {
        const val = pixelData[pos];
        setPix(x, y + 0, val & 128);
        setPix(x, y + 1, val & 64);
        setPix(x, y + 2, val & 32);
        setPix(x, y + 3, val & 16);
        setPix(x, y + 4, val & 8);
        setPix(x, y + 5, val & 4);
        setPix(x, y + 6, val & 2);
        setPix(x, y + 7, val & 1);
        pos++;
      }
    }

    // Copy to preview canvas
    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    ctx.putImageData(img, 0, 0);
  }

  $effect(() => {
    draw();
  });
</script>

<main>
  <h1>Web Label Printer</h1>
  <div class="r">
    <textarea bind:value={text} placeholder="Write text here..."></textarea>
    <details>
      <summary>⚙️ Configuration</summary>
      <div class="config">
        <label>
          Label Width (mm): <input
            type="number"
            bind:value={label_width_mm}
            min="1"
            max="100"
            step="0.1"
          />
        </label>
        <label>
          Label Height (mm): <input
            type="number"
            bind:value={label_height_mm}
            min="1"
            max="100"
            step="0.1"
          />
        </label>
        <label>
          Margin (mm): <input
            type="number"
            bind:value={margin_mm}
            min="0"
            max="10"
            step="0.1"
          />
        </label>
      </div>
    </details>
    <button>Print Label</button>
    <div class="l">
      <canvas bind:this={canvas} width={label_width} height={label_height}>
      </canvas>
      <span>
        Label size:
        {(label_width * 0.125).toFixed(1)}
        ×
        {(label_height * 0.125).toFixed(1)}
        mm
      </span>
      <span>Margin: {(margin * 0.125).toFixed(1)} mm</span>
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
  details {
    width: 100%;
    box-sizing: border-box;
  }
  .config {
    display: flex;
    flex-direction: column;
    gap: 0.5em;
    padding: 1em;
    border: 2px solid #ddd;
    border-radius: 12px;
    background-color: #f9f9f9;
  }
  .config label {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-weight: 500;
  }
  .config input {
    width: 80px;
    padding: 0.3em;
    border: 1px solid #ccc;
    border-radius: 6px;
    text-align: center;
  }
</style>
