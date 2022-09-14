<script>
    import { onMount } from "svelte";
    import { debounce } from "./utils.js";
    import { presets } from "./presets.js";
    import { fade, fly } from "svelte/transition";

    // global variables

    let width = window.innerWidth;
    let height = window.innerHeight;
    let pathString = "";
    let pathColor = "#ff0080";
    const MAX_PATH_LENGTH = 200000;
    let showCircles = true;
    let paused = false;
    let t = 0;
    let presetIndex = 3;
    let circles = [];

    // main loop

    function loop() {
        if (paused) return;
        calculateCoordinates();
        calculatePath();
        t++;
        requestAnimationFrame(loop);
    }

    // execute loop and set some preset

    onMount(() => {
        setPreset();
        loop();
    });

    // calculates the circle coordinates

    function calculateCoordinates() {
        for (let i = 1; i < circles.length; i++) {
            const x1 = circles[i - 1].x;
            const y1 = circles[i - 1].y;
            const r1 = circles[i - 1].radius;
            const r2 = circles[i].radius;
            const angle =
                t * circles[i].speed * ((2 * Math.PI) / 180) -
                (90 * Math.PI) / 180;
            circles[i].x = x1 + (r1 + r2) * Math.cos(angle);
            circles[i].y = y1 + (r1 + r2) * Math.sin(angle);
        }
    }

    // calculates the path string

    function calculatePath() {
        if (pathString.length >= MAX_PATH_LENGTH) return;

        const { x, y } = circles.at(-1);
        if (pathString.length == 0) {
            pathString = `M ${x},${y}`;
        } else {
            pathString += ` L ${x},${y}`;
        }
    }

    // toggles the circles

    function toggleCircles() {
        showCircles = !showCircles;
    }

    // adds a circle

    function addCircle() {
        const lastCircle = circles.at(-1);
        const newCircle = {
            radius: lastCircle.radius / 2,
            speed: Math.max(1, lastCircle.speed * 4),
            x: 0,
            y: 0,
        };
        circles = [...circles, newCircle];
        clearPath();
        if (paused) calculateCoordinates();
    }

    // removes the last circle

    function removeLastCircle() {
        if (circles.length <= 1) return;
        circles = circles.slice(0, circles.length - 1);
        clearPath();
    }

    // clears the path

    function clearPath() {
        t = 0;
        pathString = "";
    }

    // toggles pause

    function togglePause() {
        paused = !paused;
        if (!paused) loop();
    }

    // sets a preset

    function setPreset() {
        clearPath();
        circles = presets[presetIndex].data.map(
            ([radius, speed], index) => {
                return {
                    radius,
                    speed,
                    x: index == 0 ? width / 2 : 0,
                    y: index == 0 ? height / 2 : 0,
                };
            }
        );
        if (paused) calculateCoordinates();
    }

    // window resize handler

    window.addEventListener(
        "resize",
        debounce(() => {
            width = window.innerWidth;
            height = window.innerHeight;
            circles[0].x = width / 2;
            circles[1].x = height / 2;
            clearPath();
            if (paused) calculateCoordinates();
        }, 150)
    );
</script>

<!-- menu -->

<menu>
    <div class="controls">
        <button on:click={addCircle}>Add circle</button>
        <button
            disabled={circles.length == 1}
            on:click={removeLastCircle}
        >
            Remove last circle
        </button>
        <button on:click={toggleCircles}>Toggle circles</button>
        <button on:click={togglePause}>Toggle Pause</button>
        <label class="colorLabel">
            <span>Path color</span>
            <input type="color" bind:value={pathColor} />
        </label>
        <label
            ><span>Preset</span>
            <select
                bind:value={presetIndex}
                on:change={() => setPreset()}
            >
                {#each presets as preset, index}
                    <option value={index}>{preset.name}</option>
                {/each}
            </select>
        </label>
    </div>
    <table>
        <thead>
            <tr>
                <td>circle</td>
                <td>radius</td>
                <td>speed</td>
            </tr>
        </thead>
        <tbody>
            {#each circles as circle, index}
                <tr transition:fly={{ x: -50, duration: 200 }}>
                    <td>
                        {index + 1}
                    </td>
                    <td>
                        <input
                            type="number"
                            bind:value={circle.radius}
                            step="1"
                            min="1"
                            on:input={clearPath}
                        />
                    </td>
                    <td>
                        <input
                            type="number"
                            bind:value={circle.speed}
                            step="0.1"
                            min="0"
                            disabled={index == 0}
                            on:input={clearPath}
                        />
                    </td>
                </tr>
            {/each}
        </tbody>
    </table>
</menu>

<!-- drawing -->

<svg {width} {height}>
    {#if showCircles && circles.length > 0}
        {#each circles as circle, index (index)}
            <circle
                transition:fade={{ duration: 500 }}
                cx={circle.x}
                cy={circle.y}
                r={circle.radius}
                stroke="black"
                fill="none"
                stroke-width="2"
            />
        {/each}
        <circle
            transition:fade={{ duration: 500 }}
            cx={circles.at(-1).x}
            cy={circles.at(-1).y}
            r={3}
            stroke="none"
            fill="#888"
        />
    {/if}
    <path
        d={pathString}
        fill="none"
        stroke={pathColor}
        stroke-width="2"
    />
</svg>

<!-- styles -->
<style>
    svg {
        display: block;
    }

    menu {
        position: absolute;
        top: 0;
        left: 0;
        z-index: 1;
        padding: 20px;
    }

    .controls {
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        gap: 10px;
        margin-bottom: 20px;
    }

    button {
        font-family: inherit;
        font-size: inherit;
        background: #777;
        color: white;
        padding: 5px 12px;
        border: none;
        border-radius: 3px;
        cursor: pointer;
        white-space: nowrap;
    }

    button:hover,
    button:focus-visible {
        background-color: #555 !important;
    }

    button:disabled {
        opacity: 0.4;
        pointer-events: none;
    }

    input[type="number"] {
        width: 70px;
        font-family: inherit;
        outline: none;
        padding: 5px;
        border: 1px solid #777;
        border-radius: 3px;
    }

    input[type="number"]:focus {
        background-color: #eee;
    }

    input[type="color"] {
        width: 20px;
        height: 20px;
        display: inline-block;
        border: none;
        cursor: pointer;
    }

    .colorLabel {
        display: flex;
        align-items: center;
        gap: 5px;
        cursor: pointer;
    }

    select {
        font-family: inherit;
        font-size: inherit;
        border: 1px solid #777;
        border-radius: 3px;
        padding: 3px;
    }

    table {
        border-spacing: 5px;
        background-color: #fff7;
    }

    thead td {
        text-align: center;
        font-size: 14px;
    }

    td:first-child {
        text-align: center;
    }
</style>
