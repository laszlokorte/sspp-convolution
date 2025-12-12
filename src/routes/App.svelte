<script>
    import { bluenoise16 } from "./bluenoise";
    import exampleImage from "./exampleImage";
    import { uniformnoise16 } from "./uniformnoise";
    import { whitenoise16 } from "./whitenoise";
    import favicon from "$lib/assets/favicon.svg";

    const numf = new Intl.NumberFormat("us", {
        maximumFractionDigits: 2,
        minimumFractionDigits: 2,
    });
    let sampleValue = $state(0.5);
    let variance = $state(1);
    let noiseCoord = $state({ x: 0, y: 0, selected: false });
    let convCoord = $state({ x: 0, y: 0 });
    let name = $state("world");
    const stride = 11;
    let kernel = $derived({
        size: { x: stride, y: stride, stride: stride },
        max: 1 / Math.sqrt(2),
        values: Array(stride * stride)
            .fill(variance)
            .map((v, i) => {
                if (v == 0) {
                    return i == (stride * stride - 1) / 2 ? 1 : 0;
                }
                const x = i % stride;
                const y = (i - x) / stride;

                const dx = x - (stride - 1) / 2;
                const dy = y - (stride - 1) / 2;

                const d = Math.hypot(dx, dy);
                return (
                    (Math.exp((-d * d) / Math.pow(v, 2)) * Math.PI) /
                    Math.sqrt(2) /
                    v
                );
            }),
    });

    let noiseIndex = $state(0);
    const noises = [
        { data: bluenoise16, label: "blue" },
        { data: whitenoise16, label: "white" },
        { data: uniformnoise16, label: "constant" },
    ];
    let noiseSource = $derived(noises[noiseIndex].data);

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
                for (let l = 0; l <= i; l++) {
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
                    (noisePos % kernel.size.stride);
                const offsetY =
                    Math.floor(kernel.size.y / 2) -
                    Math.floor(noisePos / kernel.size.stride);
                const sampleX =
                    (x + offsetX + inputImage.size.x) % inputImage.size.x;
                const sampleY =
                    (y + offsetY + inputImage.size.y) % inputImage.size.y;
                return inputImage.values[sampleX + 64 * sampleY];
            }),
    });
    const singleSampleNoisePos = $derived(
        kernelPrefix.findIndex(
            (v, i, a) => v >= sampleValue || i + 1 == a.length,
        ),
    );
    let outputImageSingleSample = $derived.by(() => {
        const x = convCoord.x;
        const y = convCoord.y;
        const noisePos = singleSampleNoisePos;

        const offsetX =
            Math.floor(kernel.size.x / 2) - (noisePos % kernel.size.stride);
        const offsetY =
            Math.floor(kernel.size.y / 2) -
            Math.floor(noisePos / kernel.size.stride);
        const sampleX = (x + offsetX + inputImage.size.x) % inputImage.size.x;
        const sampleY = (y + offsetY + inputImage.size.y) % inputImage.size.y;
        return inputImage.values[sampleX + 64 * sampleY];
    });

    const scale = 30;
</script>

<title>Single Sample per Pixel Convolution</title>

<main>
    <h1>
        <img width="32" height="32" src={favicon} alt="Icon" />
        <span>Single sample per pixel convolution (SSPP/1SSP)</span>
    </h1>
    <p>
        <em>Single Sample per Pixel (1SPP)</em> is a stochastic trick to improve the
        performance of otherwise expensive convolution operations by sampling only
        a single pixel from the input image for calculating each output pixel.
    </p>
    <details>
        <summary>Explanation</summary>
        <div>
            <p>
                In classic convolution, a filter kernel is shifted across an
                image and at each location each kernel pixel is multiplied with
                each overlapping image pixel, to the sum the products. For large
                filter kernels this calculation is expensive. With an
                11&cross;11 kernel for example 121 multiplications are needed
                for each pixel in the output image to be calculated.
            </p>
            <p>
                With <em>SSPP</em> instead of shifting the kernel itself across the
                image, the kernel is interpreted as probability distribution. It can
                be summed up into the corresponding cumulative distribution. This
                distribution can then be sampled to select a single position in the
                kernel each each step. High values in the original filter kernel correspond
                to positions that are likely to be picked.
            </p>
            <p>
                Then while sliding the window across the input image at each
                window position a single coordinate is sampled from the
                distribution to determine which pixel from the input image
                should be used as output pixel. This decreases the cost from
                being linear in kernel area size to being constant time per
                input image pixel.
            </p>
            <p>
                For sampling from the cumulative distribution a precalulated
                source of randomness (of proper size) can be used. The source of
                randomness has an influence on the result as well.
            </p>
            <p>
                Below you can see a Gaussian filter kernel, its corresponding
                cummulative distribution, different kinds of noise sources you
                can choose from, the input image and the resulting output image.
            </p>
            <p>
                Click on a pixel in the output image to understand how its value
                is calculated. The green square in the input image shows the
                window from which a pixel is picked. The small magenta square
                inside the gree window shows the single actually picked pixel.
                The location of this picked value corresponds to the location of
                the pixel sampled from the cumulated filter kernel. This sample
                in turn is picked by choosing a sample value from the noise
                source and the finding the first entry in the cumulative filter
                kernel, that is greater or equal to the picked random value.
            </p>
            <p>
                You can also pick a single sample value by hand, using the <em
                    >Sample Value</em
                > slider and see how it effects the resulting location in the cumulated
                filter kernel. Alternatively you can click on a pixel in the cumulated
                kernel to retreive the corresponding sample value.
            </p>
        </div>
    </details>
    <p>
        Try to play around with the variance of the filter and with the types of
        noise sources to see how it effects the resulting output image.
    </p>

    <div class="galery">
        <figure class="figure">
            <figcaption>Filter Kernel (Gaussian)</figcaption>
            <div class="radio-list">
                <span class="slider-control">
                    <label>
                        Variance
                        <input
                            type="range"
                            min="0"
                            max="6"
                            bind:value={variance}
                            step={numf.format(1 / 2)}
                        /></label
                    >
                    <output>{numf.format(variance)}</output>
                </span>
            </div>
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
                        noiseCoord.selected = false;
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
                        noiseCoord.selected = false;
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
                    fill="deeppink"
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
            <div class="radio-list">
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
            </div>
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

                        noiseCoord.selected = false;
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

                        noiseCoord.selected = false;
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
                    fill="deeppink"
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
            <div class="radio-list">
                {#each noises as n, ni}
                    <label class="radio-control">
                        <input
                            type="radio"
                            bind:group={noiseIndex}
                            value={ni}
                        />{n.label}</label
                    >
                {/each}
            </div>
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
                        noiseCoord.selected = true;
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
                        noiseCoord.selected = true;
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
                            noiseSource.values[
                                x + y * noiseSource.size.stride
                            ] / noiseSource.max}
                        <rect
                            x={scale * x}
                            y={scale * y}
                            fill="hsl(0,0%,{numf.format(intensity * 100)}%)"
                            width={scale}
                            height={scale}
                        ></rect>
                    {/each}
                {/each}

                {#if noiseCoord.selected}
                    <rect
                        width={scale}
                        height={scale}
                        stroke-width={(2 * scale) / 10}
                        stroke="cyan"
                        fill="turquoise"
                        fill-opacity="0.6"
                        opacity={0.9}
                        x={noiseCoord.x * scale}
                        y={noiseCoord.y * scale}
                    ></rect>
                {/if}
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
                        noiseCoord.selected = true;
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
                        noiseCoord.selected = true;
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
                            stroke="lime"
                            fill="mediumspringgreen"
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
                    fill="deeppink"
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
                        noiseCoord.selected = true;
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
                        noiseCoord.selected = true;
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
                    fill="hsl(0,0%,{numf.format(
                        outputImageSingleSample * 100,
                    )}%)"
                    opacity={0.9}
                    x={convCoord.x * scale}
                    y={convCoord.y * scale}
                ></rect>

                <rect
                    width={scale}
                    height={scale}
                    stroke-width={(2 * scale) / 10}
                    stroke="lime"
                    fill="mediumspringgreen"
                    fill-opacity="0.5"
                    opacity={0.9}
                    x={convCoord.x * scale}
                    y={convCoord.y * scale}
                ></rect>
            </svg>
        </figure>
    </div>
    <footer>
        <a href="//tools.laszlokorte.de">more educational tools</a>
    </footer>
</main>

<style>
    main {
        max-width: 60em;
        margin: auto;
    }
    .galery {
        display: flex;
        flex-wrap: wrap-reverse;
        gap: 1ex;
        margin: 1ex;
    }

    figcaption {
        text-align: center;
        padding: 1ex;
    }
    svg {
        display: block;
        min-width: 10em;
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
        accent-color: hotpink;
    }
    .slider-control > label {
        display: flex;
        align-items: center;
        gap: 1ex;
        white-space: nowrap;
    }
    .radio-list {
        display: flex;
        gap: 1em;
        justify-content: center;
        align-items: baseline;
        padding: 0.5ex;
    }
    .radio-control {
        display: flex;
        gap: 1ex;
        align-items: center;
        justify-content: center;
    }
    .radio-control:has(:checked) {
        color: cyan;
        accent-color: turquoise;
    }
    input[type="radio"] {
        margin: 0;
    }
    summary {
        cursor: pointer;
        padding: 1em;
        background-color: #111;
        color: #fff;
        font-family: monospace;
        margin: 1ex;
    }
    summary + div {
        margin: 1em;
        padding-bottom: 1ex;
    }
    details {
        background-color: #eee;
    }
</style>
