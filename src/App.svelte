<script lang="ts">
  import { onMount } from "svelte";

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
  let margin_height_mm = $state(
    parseFloat(localStorage.getItem("margin_height_mm") || "1"),
  );
  let margin_width_mm = $state(
    parseFloat(localStorage.getItem("margin_mm") || "1"),
  );

  // Width and height in 0.125 mm units (derived, rounded to integers)
  let label_width = $derived(Math.round(label_width_mm * 8));
  let label_height = $derived(Math.round(label_height_mm * 8));
  let margin_x = $derived(Math.round(margin_width_mm * 8));
  let margin_y = $derived(Math.round(margin_height_mm * 8));
  let text = $state(sessionStorage.getItem("text") || "");

  // Save to localStorage
  $effect(() => {
    localStorage.setItem("label_width_mm", label_width_mm.toString());
    localStorage.setItem("label_height_mm", label_height_mm.toString());
    localStorage.setItem("margin_mm", margin_width_mm.toString());
    localStorage.setItem("margin_height_mm", margin_height_mm.toString());
  });

  // Save text to sessionStorage
  $effect(() => {
    sessionStorage.setItem("text", text);
  });
  let canvas: HTMLCanvasElement | undefined = $state();
  let pixelData = new Uint8Array();

  function renderLine(
    ctx: CanvasContext,
    line: string,
    size: number,
    render_fn: (str: string) => void,
  ) {
    // Split line into components:
    const match_bold = /(?<!\*)\*(?! )([^*]*?[^* ]\*)/g;
    const match_italic = /(?<!_)_(?! )([^_]*?[^_ ]_)/g;
    const match_barcode = /\[\|.*?\|\]/g;
    const match_big = /\^\^.*?\^\^/g;
    const match_small = /__.*?__/g;

    const parsed = [];

    // Start by matching "barcode" tags to avoid matching inside
    var clean_line = line;
    for (const m of clean_line.matchAll(match_barcode)) {
      // Add to parsed array
      const start = m.index;
      const end = m.index + m[0].length;
      parsed.push({ pos: start, type: "barcode" });
      parsed.push({ pos: end - 2, type: "barcode end" });
      clean_line =
        clean_line.substring(0, start) +
        "\n".repeat(m[0].length) +
        clean_line.substring(end);
    }

    // Now, add all matches
    const add = (type: string, regex: RegExp, ln: number) => {
      for (const m of clean_line.matchAll(regex)) {
        const start = m.index;
        const end = m.index + m[0].length;
        parsed.push({ pos: start, type });
        parsed.push({ pos: end - ln, type: type + " end" });
      }
    };

    add("big", match_big, 2);
    add("small", match_small, 2);
    add("bold", match_bold, 1);
    add("italic", match_italic, 1);

    // Add a last tag at string end
    parsed.push({ pos: line.length, type: "" });

    // Sort tags
    const tags = parsed.sort((a, b) => a.pos - b.pos);

    // Now, process the line keeping the state of the font
    var pos = 0;
    var current_font = { size, name: "sans", italic: false, bold: false };
    const font_stack: (typeof current_font)[] = [];
    const make_font = (f: typeof current_font) => {
      if (f.name.search(" "))
        return `${f.italic ? "italic" : ""} ${f.bold ? "bold" : ""} ${f.size}px "${f.name}"`;
      else
        return `${f.italic ? "italic" : ""} ${f.bold ? "bold" : ""} ${f.size}px ${f.name}`;
    };
    for (const tag of tags) {
      if (tag.pos > pos) {
        // Add current string
        ctx.font = make_font(current_font);
        render_fn(line.substring(pos, tag.pos));
      }
      // Modify font
      switch (tag.type) {
        case "bold":
          current_font.bold = true;
          pos = tag.pos + 1;
          break;
        case "italic":
          current_font.italic = true;
          pos = tag.pos + 1;
          break;
        case "big":
          current_font.size *= 5.0 / 4.0;
          pos = tag.pos + 2;
          break;
        case "small":
          current_font.size *= 4.0 / 5.0;
          pos = tag.pos + 2;
          break;
        case "bold end":
          current_font.bold = false;
          pos = tag.pos + 1;
          break;
        case "italic end":
          current_font.italic = false;
          pos = tag.pos + 1;
          break;
        case "big end":
          current_font.size *= 4.0 / 5.0;
          pos = tag.pos + 2;
          break;
        case "small end":
          current_font.size *= 5.0 / 4.0;
          pos = tag.pos + 2;
          break;
        case "barcode":
          font_stack.push(current_font);
          current_font.bold = current_font.italic = false;
          current_font.name = "Libre Barcode 39";
          pos = tag.pos + 2;
          break;
        case "barcode end":
          const s = font_stack.pop();
          if (s) current_font = s;
          pos = tag.pos + 2;
          break;
      }
    }
  }

  function measureLine(ctx: CanvasContext, line: string, size: number) {
    // To properly measure, we need to return:
    //  - Right as the first actualBoundingBoxRight;
    //  - Left as the sum of all the widths minus the last width and plus the last actualBoundingBoxLeft.

    var first = true;
    var right = 0;
    var left = 0;
    var ascent = 0;
    var descent = 0;
    var m: TextMetrics | undefined;
    renderLine(ctx, line, size, (str) => {
      m = ctx.measureText(str);
      if (first) {
        ascent = m.actualBoundingBoxAscent;
        descent = m.actualBoundingBoxDescent;
        left = m.actualBoundingBoxLeft;
        right = m.width;
        first = false;
      } else {
        ascent = Math.max(ascent, m.actualBoundingBoxAscent);
        descent = Math.max(descent, m.actualBoundingBoxDescent);
        right += m.width;
      }
    });
    if (m !== undefined) {
      right += m.actualBoundingBoxRight - m.width;
    }
    return { right, left, ascent, descent };
  }

  function fullSize(ctx: CanvasContext, lines: string[], sz: number) {
    const line_hg = Math.round((sz * 7) / 6);
    var wd = 0;
    var total_hg = 0;
    // Get all lines
    for (var i = 0; i < lines.length; i++) {
      const m = measureLine(ctx, lines[i], sz);
      wd = Math.max(wd, m.right + m.left);
      total_hg += i == 0 ? m.ascent : line_hg;
      if (i == lines.length - 1) {
        total_hg += m.descent;
      }
    }
    return { wd, line_hg, total_hg };
  }

  function bestSize(
    ctx: CanvasContext,
    lines: string[],
    min_size = 8,
    max_size = 144,
  ) {
    var wd = ctx.canvas.width - margin_x * 2;
    var hg = ctx.canvas.height - margin_y * 2;

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
    const lines = text.split("\n");
    const sz = bestSize(ctx, lines);
    const font_sz = fullSize(ctx, lines, sz);

    ctx.textBaseline = "alphabetic";
    ctx.fillStyle = "#000";
    var y = (label_height - font_sz.total_hg) / 2;
    for (var i = 0; i < lines.length; i++) {
      const line = lines[i];

      // We have to render twice, once to measure, another to draw
      const m = measureLine(ctx, line, sz);

      const line_wd = m.right + m.left;
      var x = (label_width - line_wd) / 2 + m.left;
      y += i ? font_sz.line_hg : m.ascent;

      renderLine(ctx, lines[i], sz, (str) => {
        const m = ctx.measureText(str);
        ctx.fillText(str, x, y);
        x += m.width;
      });
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
      const p = label_width - x - 1 + y * label_width;
      const val =
        imageData.data[4 * p + 0] + // red
        imageData.data[4 * p + 1] + // green
        imageData.data[4 * p + 2]; // blue
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
      const t = label_width - 1 - x + y * label_width;
      img.data[t * 4 + 0] = p;
      img.data[t * 4 + 1] = p;
      img.data[t * 4 + 2] = p;
      img.data[t * 4 + 3] = 255;
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

  onMount(() => {
    // Force loading of font at start, so we don't need to include in the HTML
    const barcode_font = new FontFace(
      "Libre Barcode 39",
      "url('fonts/LibreBarcode39Text-Regular.ttf')",
    );
    document.fonts.add(barcode_font);
    barcode_font.load().then(() => {
      draw();
    });
  });

  async function printLabel() {
    try {
      const device = await navigator.bluetooth.requestDevice({
        filters: [{ services: ["0000ff00-0000-1000-8000-00805f9b34fb"] }],
      });
      const server = await device.gatt?.connect();
      const service = await server?.getPrimaryService(
        "0000ff00-0000-1000-8000-00805f9b34fb",
      );
      const characteristic = await service?.getCharacteristic(
        "0000ff02-0000-1000-8000-00805f9b34fb",
      );

      // Ok, will print our data
      const bytes_height = Math.floor((label_height + 7) / 8);
      // ESC/POS reference:
      //   https://download4.epson.biz/sec_pubs/pos/reference_en/escpos/commands.html
      const header = new Uint8Array([
        0x1b, // ESC @ : initialize printer
        0x40,
        0x1d, // GS v 0 : Print raster image
        0x76,
        0x30,
        0x00, // normal image
        bytes_height % 256, // bytes in horizontal direction
        Math.floor(bytes_height / 256),
        label_width % 256, // lines in vertical direction
        Math.floor(label_width / 256),
      ]);
      // ESC d 00 : Print and feed 0 lines.
      const footer = new Uint8Array([0x1b, 0x64, 0x00]);

      await characteristic?.writeValueWithResponse(header);
      for (let i = 0; i < pixelData.length; i += 128) {
        const buf = pixelData.slice(i, i + 128);
        // TODO: last packet should be padded??
        await characteristic?.writeValueWithoutResponse(buf);
      }
      await characteristic?.writeValueWithResponse(footer);
    } catch (err) {
      console.log(`Error printing: ${err}`);
    }
  }
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
            max="150"
            step="0.1"
          />
        </label>
        <label>
          Label Height (mm): <input
            type="number"
            bind:value={label_height_mm}
            min="1"
            max="18"
            step="0.1"
          />
        </label>
        <label>
          Margin Width (mm): <input
            type="number"
            bind:value={margin_width_mm}
            min="0"
            max="10"
            step="0.1"
          />
        </label>
        <label>
          Margin Height (mm): <input
            type="number"
            bind:value={margin_height_mm}
            min="0"
            max="10"
            step="0.1"
          />
        </label>
      </div>
    </details>
    <button onclick={printLabel}>Print Label</button>
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
      <span>
        Margin:
        {(margin_x * 0.125).toFixed(1)}
        ×
        {(margin_y * 0.125).toFixed(1)}
        mm
      </span>
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
