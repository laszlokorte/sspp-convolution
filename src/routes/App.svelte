<script>
    import { bluenoise16 } from "./bluenoise";
    import exampleImage from "./exampleImage";

    const numf = new Intl.NumberFormat("us", {
        maximumFractionDigits: 2,
        minimumFractionDigits: 2,
    });
    let sampleValue = $state(0.5);
    let noiseCoord = $state({ x: 0, y: 0 });
    let convCoord = $state({ x: 0, y: 0 });
    let name = $state("world");
    const stride = 11;
    let kernel = $state({
        size: { x: stride, y: stride, stride: stride },
        max: 1 / Math.sqrt(2),
        values: Array(stride * stride)
            .fill(0)
            .map((_, i) => {
                const x = i % stride;
                const y = (i - x) / stride;

                const dx = x - (stride - 1) / 2;
                const dy = y - (stride - 1) / 2;

                const d = Math.hypot(dx, dy);
                return (Math.exp((-d * d) / 4) * Math.PI) / Math.sqrt(2);
            }),
    });

    let noiseSource = $state(bluenoise16);

    //  function extractGrayscalePixels(img, width = 64, height = 64) {
    //      // Create a canvas
    //      const canvas = document.createElement("canvas");
    //      canvas.width = width;
    //      canvas.height = height;
    //      const ctx = canvas.getContext("2d");

    //      // Draw the image scaled to canvas size
    //      ctx.drawImage(img, 0, -5, width, (height * img.height) / img.width);

    //      // Get image data
    //      const imageData = ctx.getImageData(0, 0, width, height);
    //      const data = imageData.data; // RGBA array
    //      const grayscale = [];

    //      for (let i = 0; i < data.length; i += 4) {
    //          // Convert RGB to grayscale using luminosity method
    //          const r = data[i];
    //          const g = data[i + 1];
    //          const b = data[i + 2];
    //          const gray = (0.299 * r + 0.587 * g + 0.114 * b) / 255;
    //          grayscale.push(gray);
    //      }

    //      return grayscale; // 64*64 = 4096 values between 0..1
    //  }
    //  const someImage = await new Promise((r) => {
    //      const img = document.createElement("img");
    //      img.onload = function () {
    //          const grayArray = extractGrayscalePixels(img);
    //          r(grayArray);
    //      };
    //      img.src = "/example.png";
    //  });

    let inputImage = $state(exampleImage);

    const kernelSum = $derived.by(() => {
        let sum = 0;
        for (let l = 0; l < kernel.values.length; l++) {
            sum += kernel.values[l];
        }

        return sum;
    });
    const kernelPrefix = $derived(
        Array(stride * stride)
            .fill(0)
            .map((_, i) => {
                let sum = 0;
                for (let l = 0; l < i; l++) {
                    sum += kernel.values[l];
                }

                return sum / kernelSum;
            }),
    );

    const samplePosition = $derived(
        Math.max(
            kernelPrefix.findIndex(
                (v, i, a) => v >= sampleValue || i + 1 == a.length,
            ),
            0,
        ),
    );
    const noiseValue = $derived(
        noiseSource.values[noiseCoord.x + noiseCoord.y * stride],
    );
    let outputImage = $derived({
        size: { x: 64, y: 64, stride: 64 },
        max: 1,
        values: Array(64 * 64)
            .fill(0)
            .map((_, i) => {
                const x = i % 64;
                const y = (i - x) / 64;
                const noiseX = x % noiseSource.size.x;
                const noiseY = y % noiseSource.size.y;
                const noiseVal =
                    noiseSource.values[
                        noiseX + noiseY * noiseSource.size.stride
                    ];
                const noisePos = kernelPrefix.findIndex(
                    (v, i, a) => v >= noiseVal || i + 1 == a.length,
                );
                const offsetX =
                    Math.floor(kernel.size.x / 2) -
                    1 -
                    (noisePos % kernel.size.stride);
                const offsetY =
                    Math.floor(kernel.size.y / 2) -
                    1 -
                    Math.floor(noisePos / kernel.size.stride);
                const sampleX =
                    (x + offsetX + inputImage.size.x) % inputImage.size.x;
                const sampleY =
                    (y + offsetY + inputImage.size.y) % inputImage.size.y;
                return inputImage.values[sampleX + 64 * sampleY];
            }),
    });

    const scale = 30;
</script>

<title>Single Sample per Pixel Convolution</title>

<main>
    <h1>Single Sample Per Pixel Convolution</h1>
    <p>
        <em>Single Sample per Pixel (SSPP)</em> is a stochastic trick to improve the
        performance of otherwise expensive convolution operations.
    </p>
    <p>
        In classic convolution, a filter kernel is shifted across an image and
        at each location each kernel pixel is multiplied with each overlapping
        image pixel, and then summed together. For large filter kernels this
        calculation is expensive.
    </p>
    <p>
        With <em>SSPP</em> instead the kernel is transformed into a cumulative probability
        density function and then at each shifting position sampled to select only
        a single pixel from the input image. This decreases the cost from being linear
        in kernel size to being constant time per input image pixel.
    </p>
    <p>
        For sampling the probability distribution a precalulated source of
        randomness (of proper size) can be used.
    </p>
    <span class="slider-control">
        <label>
            Sample Value
            <input
                type="range"
                min="0"
                max="1"
                bind:value={sampleValue}
                step={numf.format(1 / (stride * stride))}
            /></label
        >
        <output>{numf.format(sampleValue)}</output>
    </span>
    <div class="galery">
        <figure class="figure">
            <figcaption>Filter Kernel</figcaption>
            <svg
                onpointerdown={(e) => {
                    if (e.isPrimary) {
                        e.preventDefault();
                        e.currentTarget.setPointerCapture(e.pointerId);
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const x = Math.max(
                            0,
                            Math.min(
                                kernel.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const y = Math.max(
                            0,
                            Math.min(
                                kernel.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        sampleValue = kernelPrefix[x + stride * y];
                    }
                }}
                onpointermove={(e) => {
                    if (
                        e.isPrimary &&
                        e.currentTarget.hasPointerCapture(e.pointerId)
                    ) {
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const x = Math.max(
                            0,
                            Math.min(
                                kernel.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const y = Math.max(
                            0,
                            Math.min(
                                kernel.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        sampleValue = kernelPrefix[x + stride * y];
                    }
                }}
                viewBox="{-5} {-5} {10 + kernel.size.x * scale} {10 +
                    kernel.size.y * scale}"
            >
                {#each Array(kernel.size.y) as _, y}
                    {#each Array(kernel.size.x) as _, x}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(
                                (kernel.values[x + y * kernel.size.stride] /
                                    kernel.max) *
                                    100,
                            )}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}
                <rect
                    width={scale}
                    height={scale}
                    fill="magenta"
                    fill-opacity="0.4"
                    stroke-width={scale / 10}
                    stroke="magenta"
                    opacity={0.9}
                    x={(samplePosition % kernel.size.stride) * scale}
                    y={Math.floor(samplePosition / kernel.size.stride) * scale}
                ></rect>
            </svg>
        </figure>

        <figure class="figure">
            <figcaption>Cumulated Filter Kernel</figcaption>
            <svg
                onpointerdown={(e) => {
                    if (e.isPrimary) {
                        e.preventDefault();
                        e.currentTarget.setPointerCapture(e.pointerId);
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const x = Math.max(
                            0,
                            Math.min(
                                stride - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const y = Math.max(
                            0,
                            Math.min(
                                stride - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );

                        sampleValue = kernelPrefix[x + stride * y];
                    }
                }}
                onpointermove={(e) => {
                    if (
                        e.isPrimary &&
                        e.currentTarget.hasPointerCapture(e.pointerId)
                    ) {
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const x = Math.max(
                            0,
                            Math.min(
                                stride - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const y = Math.max(
                            0,
                            Math.min(
                                stride - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );

                        sampleValue = kernelPrefix[x + stride * y];
                    }
                }}
                viewBox="{-5} {-5} {10 + kernel.size.x * scale} {10 +
                    kernel.size.y * scale}"
            >
                {#each Array(kernel.size.y) as _, y}
                    {#each Array(kernel.size.x) as _, x}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(
                                kernelPrefix[x + y * kernel.size.stride] * 100,
                            )}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}
                <rect
                    width={scale}
                    height={scale}
                    fill="magenta"
                    fill-opacity="0.4"
                    stroke-width={scale / 10}
                    stroke="magenta"
                    opacity={0.9}
                    x={(samplePosition % kernel.size.stride) * scale}
                    y={Math.floor(samplePosition / kernel.size.stride) * scale}
                ></rect>
            </svg>
        </figure>

        <figure class="figure">
            <figcaption>Noise Source</figcaption>
            <svg
                onpointerdown={(e) => {
                    if (e.isPrimary) {
                        e.preventDefault();
                        e.currentTarget.setPointerCapture(e.pointerId);
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const newX = Math.max(
                            0,
                            Math.min(
                                noiseSource.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const newY = Math.max(
                            0,
                            Math.min(
                                noiseSource.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                    }
                }}
                onpointermove={(e) => {
                    if (
                        e.isPrimary &&
                        e.currentTarget.hasPointerCapture(e.pointerId)
                    ) {
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const newX = Math.max(
                            0,
                            Math.min(
                                noiseSource.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const newY = Math.max(
                            0,
                            Math.min(
                                noiseSource.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                    }
                }}
                viewBox="{-5} {-5} {10 + noiseSource.size.x * scale} {10 +
                    noiseSource.size.y * scale}"
            >
                {#each Array(noiseSource.size.y) as _, y}
                    {#each Array(noiseSource.size.x) as _, x}
                        {@const intensity =
                            noiseSource.values[x + y * kernel.size.stride] /
                            noiseSource.max}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(intensity * 100)}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}

                <rect
                    width={scale}
                    height={scale}
                    stroke-width={(2 * scale) / 10}
                    stroke="cyan"
                    fill="cyan"
                    fill-opacity="0.5"
                    opacity={0.9}
                    x={noiseCoord.x * scale}
                    y={noiseCoord.y * scale}
                ></rect>
            </svg>
        </figure>
    </div>

    <div class="galery">
        <figure class="figure">
            <figcaption>Input Image</figcaption>

            <svg
                viewBox="{-5} {-5} {10 + inputImage.size.x * scale} {10 +
                    inputImage.size.y * scale}"
                onpointerdown={(e) => {
                    if (e.isPrimary) {
                        e.preventDefault();
                        e.currentTarget.setPointerCapture(e.pointerId);
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const convX = Math.max(
                            0,
                            Math.min(
                                outputImage.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const convY = Math.max(
                            0,
                            Math.min(
                                outputImage.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        const newX =
                            ((convX % noiseSource.size.x) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        const newY =
                            ((convY % noiseSource.size.y) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                        convCoord.x = convX;
                        convCoord.y = convY;
                    }
                }}
                onpointermove={(e) => {
                    if (
                        e.isPrimary &&
                        e.currentTarget.hasPointerCapture(e.pointerId)
                    ) {
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const convX = Math.max(
                            0,
                            Math.min(
                                outputImage.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const convY = Math.max(
                            0,
                            Math.min(
                                outputImage.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        const newX =
                            ((convX % noiseSource.size.x) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        const newY =
                            ((convY % noiseSource.size.y) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                        convCoord.x = convX;
                        convCoord.y = convY;
                    }
                }}
            >
                {#each Array(inputImage.size.y) as _, y}
                    {#each Array(inputImage.size.x) as _, x}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(
                                inputImage.values[
                                    x + y * inputImage.size.stride
                                ] * 100,
                            )}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}
                {#each [-1, 0, 1] as xx}
                    {#each [-1, 0, 1] as yy}
                        <rect
                            width={scale * kernel.size.x}
                            height={scale * kernel.size.y}
                            stroke-width={(2 * scale) / 10}
                            stroke="cyan"
                            fill="cyan"
                            fill-opacity="0.5"
                            opacity={0.9}
                            x={(convCoord.x -
                                Math.floor(kernel.size.x / 2) +
                                xx * inputImage.size.x) *
                                scale}
                            y={(convCoord.y -
                                Math.floor(kernel.size.y / 2) +
                                yy * inputImage.size.y) *
                                scale}
                        ></rect>
                    {/each}
                {/each}
                <rect
                    width={scale}
                    height={scale}
                    stroke-width={(2 * scale) / 10}
                    stroke="magenta"
                    fill="magenta"
                    fill-opacity="0.5"
                    opacity={0.9}
                    x={((convCoord.x -
                        1 -
                        Math.floor(kernel.size.x / 2) +
                        kernel.size.x -
                        (samplePosition % kernel.size.stride) +
                        inputImage.size.x) %
                        inputImage.size.x) *
                        scale}
                    y={((convCoord.y -
                        1 -
                        Math.floor(kernel.size.y / 2) +
                        kernel.size.y -
                        Math.floor(samplePosition / kernel.size.stride) +
                        inputImage.size.y) %
                        inputImage.size.y) *
                        scale}
                ></rect>
            </svg>
        </figure>
        <figure class="figure">
            <figcaption>Output Image</figcaption>

            <svg
                viewBox="{-5} {-5} {10 + outputImage.size.x * scale} {10 +
                    outputImage.size.y * scale}"
                onpointerdown={(e) => {
                    if (e.isPrimary) {
                        e.preventDefault();
                        e.currentTarget.setPointerCapture(e.pointerId);
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const convX = Math.max(
                            0,
                            Math.min(
                                outputImage.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const convY = Math.max(
                            0,
                            Math.min(
                                outputImage.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        const newX =
                            ((convX % noiseSource.size.x) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        const newY =
                            ((convY % noiseSource.size.y) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                        convCoord.x = convX;
                        convCoord.y = convY;
                    }
                }}
                onpointermove={(e) => {
                    if (
                        e.isPrimary &&
                        e.currentTarget.hasPointerCapture(e.pointerId)
                    ) {
                        const pt = e.currentTarget.createSVGPoint();
                        pt.x = e.clientX;
                        pt.y = e.clientY;
                        const svgGlobal = pt.matrixTransform(
                            e.currentTarget.getScreenCTM().inverse(),
                        );
                        const convX = Math.max(
                            0,
                            Math.min(
                                outputImage.size.x - 1,
                                Math.floor(svgGlobal.x / scale),
                            ),
                        );
                        const convY = Math.max(
                            0,
                            Math.min(
                                outputImage.size.y - 1,
                                Math.floor(svgGlobal.y / scale),
                            ),
                        );
                        const newX =
                            ((convX % noiseSource.size.x) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        const newY =
                            ((convY % noiseSource.size.y) +
                                noiseSource.size.x) %
                            noiseSource.size.x;
                        noiseCoord.x = newX;
                        noiseCoord.y = newY;
                        sampleValue =
                            noiseSource.values[
                                newX + newY * noiseSource.size.stride
                            ];
                        convCoord.x = convX;
                        convCoord.y = convY;
                    }
                }}
            >
                {#each Array(outputImage.size.y) as _, y}
                    {#each Array(outputImage.size.x) as _, x}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(
                                outputImage.values[
                                    x + y * outputImage.size.stride
                                ] * 100,
                            )}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}

                <rect
                    width={scale}
                    height={scale}
                    stroke-width={(2 * scale) / 10}
                    stroke="cyan"
                    fill="cyan"
                    fill-opacity="0.5"
                    opacity={0.9}
                    x={convCoord.x * scale}
                    y={convCoord.y * scale}
                ></rect>
            </svg>
        </figure>
    </div>
</main>

<style>
    main {
        max-width: 60em;
        margin: auto;
    }
    .galery {
        display: flex;
        gap: 1ex;
        margin: 1ex;
    }

    figcaption {
        text-align: center;
        padding: 1ex;
    }
    svg {
        display: block;
    }

    .figure {
        display: block;
        flex-grow: 1;
        flex-shrink: 1;
        margin: 0;
        flex-basis: 10em;
        background-color: #111;
        padding: 1ex;
        color: #fff;
        font-family: monospace;
    }
    .slider-control {
        user-select: none;
        display: flex;
        align-items: center;
        gap: 1ex;
    }
    .slider-control > label {
        display: flex;
        align-items: center;
        gap: 1ex;
    }
</style>
