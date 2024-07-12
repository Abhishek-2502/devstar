<script>
    import { onMount } from "svelte";

    let ctx;
    let drawing = false;
    let startX, startY;
    let currentTool = "line";
    let strokeColor = "#00bfff";
    let lineWidth = 2;
    let actions = [];
    let undoStack = [];
    let scale = 1;
    let panX = 0;
    let panY = 0;
    let fillColor = "#ff00ff";
    let fill = false;
    let dashedStroke = false;
    let roundCorners = false;
    let cornerRadius = 50;
    let showGrid = false; // Checkbox state for showing grid
    let showRulers = false;     

    let currentFreehandPoints = [];
    let controlPoint = null;

    function startDrawing(event) {
        drawing = true;
        const rect = event.target.getBoundingClientRect();
        startX = (event.clientX - rect.left - panX) / scale;
        startY = (event.clientY - rect.top - panY) / scale;

        if (currentTool === "freehand") {
            currentFreehandPoints = [{ x: startX, y: startY }];
        } else if (currentTool === "curve") {
            controlPoint = { x: startX, y: startY };
        }
    }

    function draw(event) {
        if (!drawing) return;
        const rect = event.target.getBoundingClientRect();
        const x = (event.clientX - rect.left - panX) / scale;
        const y = (event.clientY - rect.top - panY) / scale;

        redrawCanvas();
        ctx.lineWidth = lineWidth;
        ctx.strokeStyle = strokeColor;
        ctx.fillStyle = fillColor;
        ctx.lineCap = "round";

        if (dashedStroke) {
            ctx.setLineDash([5, 5]); // Array represents dash pattern: [dashLength, gapLength]
        } else {
            ctx.setLineDash([]); // Set to an empty array to reset to solid line
        }

        if (currentTool === "line") {
            ctx.beginPath();
            ctx.moveTo(startX, startY);
            ctx.lineTo(x, y);
            ctx.stroke();
        } else if (currentTool === "rectangle") {
            // ctx.strokeRect(startX, startY, x - startX, y - startY);
            // if (fill) {
            //     ctx.fillRect(startX, startY, x - startX, y - startY);
            // }
            // ctx.beginPath();
            // ctx.rect(startX, startY, x - startX, y - startY);
            // ctx.stroke();
            ctx.beginPath();
            if (roundCorners) {
                const width = x - startX;
                const height = y - startY;
                const radius = cornerRadius;
                ctx.moveTo(startX + radius, startY);
                ctx.arcTo(x, startY, x, startY + radius, radius);
                ctx.arcTo(x, y, x - radius, y, radius);
                ctx.arcTo(startX, y, startX, y - radius, radius);
                ctx.arcTo(startX, startY, startX + radius, startY, radius);
                ctx.closePath();
            } else {
                ctx.rect(startX, startY, x - startX, y - startY);
            }
            ctx.stroke();
            if (fill) {
                ctx.fill();
            }
        } else if (currentTool === "circle") {
            // ctx.beginPath();
            // const radius = Math.sqrt(
            //     Math.pow(x - startX, 2) + Math.pow(y - startY, 2),
            // );
            // ctx.arc(startX, startY, radius, 0, Math.PI * 2);
            // if (fill) {
            //     ctx.fill();
            // }
            // ctx.stroke();
            ctx.beginPath();
            const radius = Math.sqrt(Math.pow(x - startX, 2) + Math.pow(y - startY, 2));
            ctx.arc(startX, startY, radius, 0, Math.PI * 2);
            ctx.stroke();
        } else if (currentTool === "curve") {
            // controlPoint = { x, y };
            // ctx.beginPath();
            // ctx.moveTo(startX, startY);
            // ctx.quadraticCurveTo(controlPoint.x, controlPoint.y, x, y);
            // if (fill) {
            //     ctx.fill();
            // }
            // ctx.stroke();
            ctx.beginPath();
            ctx.moveTo(startX, startY);
            ctx.quadraticCurveTo(controlPoint.x, controlPoint.y, x, y);
            ctx.stroke();
        } else if (currentTool === "arrow") {
            const headlen = 10;
            const angle = Math.atan2(y - startY, x - startX);
            ctx.beginPath();
            ctx.moveTo(startX, startY);
            ctx.lineTo(x, y);
            ctx.lineTo(
                x - headlen * Math.cos(angle - Math.PI / 6),
                y - headlen * Math.sin(angle - Math.PI / 6),
            );
            ctx.moveTo(x, y);
            ctx.lineTo(
                x - headlen * Math.cos(angle + Math.PI / 6),
                y - headlen * Math.sin(angle + Math.PI / 6),
            );
            ctx.stroke();
        } else if (currentTool === "freehand") {
            currentFreehandPoints.push({ x, y });
            ctx.beginPath();
            for (let i = 0; i < currentFreehandPoints.length - 1; i++) {
                ctx.moveTo(
                    currentFreehandPoints[i].x,
                    currentFreehandPoints[i].y,
                );
                ctx.lineTo(
                    currentFreehandPoints[i + 1].x,
                    currentFreehandPoints[i + 1].y,
                );
            }
            // if (
            //     fill &&
            //     isShapeClosed({
            //         tool: "freehand",
            //         points: currentFreehandPoints,
            //     })
            // ) {
            //     ctx.fill();
            // } else {
            //     ctx.stroke();
            // }
            ctx.stroke();
        }
        if (showGrid) {
            drawGrid();

        }
    }

    function stopDrawing(event) {
        if (!drawing) return;
        drawing = false;
        const rect = event.target.getBoundingClientRect();
        const endX = (event.clientX - rect.left - panX) / scale;
        const endY = (event.clientY - rect.top - panY) / scale;

        if (currentTool === "freehand") {
            actions.push({
                tool: "freehand",
                points: currentFreehandPoints,
                strokeColor,
                fillColor,
                lineWidth,
                fill,
                dashedStroke,
                roundCorners,
            });
        } else if (currentTool === "text") {
            const text = prompt("Enter text:");
            if (text) {
                actions.push({
                    tool: "text",
                    startX,
                    startY,
                    text,
                    strokeColor,
                    fillColor,
                    lineWidth,fill,
                    dashedStroke,
                    roundCorners,
                });
            }
        } else if (currentTool === "curve") {
            actions.push({
                tool: "curve",
                startX,
                startY,
                controlX: controlPoint.x,
                controlY: controlPoint.y,
                endX,
                endY,
                strokeColor,
                fillColor,
                lineWidth,
                fill,
                dashedStroke,
                roundCorners,
            });
        } else if (currentTool === "line") {
            actions.push({
                tool: "line",
                startX,
                startY,
                endX,
                endY,
                strokeColor,
                fillColor,
                lineWidth,
                fill,
                dashedStroke,
                roundCorners,
            });
        } else {
            actions.push({
                tool: currentTool,
                startX,
                startY,
                endX,
                endY,
                strokeColor,
                fillColor,
                lineWidth,
                fill,dashedStroke,
                roundCorners,
            });
        }
        if (currentTool === "fill") {
            actions.push({ tool: "fill", startX, startY, color });
            fillShape(endX, endY);
        }

        undoStack = [];
        redrawCanvas();
    }

    function drawActions() {
        actions.forEach((action) => {
            ctx.lineWidth = action.lineWidth;
            ctx.strokeStyle = action.strokeColor;
            ctx.fillStyle = action.fillColor;
            ctx.beginPath();

            if (action.dashedStroke) {
                ctx.setLineDash([5, 5]); // Array represents dash pattern: [dashLength, gapLength]
            } else {
                ctx.setLineDash([]); // Set to an empty array to reset to solid line
            }

            if (action.tool === "line") {
                ctx.moveTo(action.startX, action.startY);
                ctx.lineTo(action.endX, action.endY);
            } else if (action.tool === "rectangle") {
                // ctx.strokeRect(
                //     action.startX,
                //     action.startY,
                //     action.endX - action.startX,
                //     action.endY - action.startY,
                // );
                // if (fill && isShapeClosed(action)) {
                //     ctx.fillRect(
                //         action.startX,
                //         action.startY,
                //         action.endX - action.startX,
                //         action.endY - action.startY,
                //     );
                // } else {
                //     ctx.strokeRect(
                //         action.startX,
                //         action.startY,
                //         action.endX - action.startX,
                //         action.endY - action.startY,
                //     );
                // }
                // ctx.rect(action.startX, action.startY, action.endX - action.startX, action.endY - action.startY);
                // ctx.stroke();
                // if (action.fill) {
                //     ctx.fill();
                // }
                ctx.beginPath();
                if (action.roundCorners) {
                    const width = action.endX - action.startX;
                    const height = action.endY - action.startY;
                    const radius = cornerRadius;
                    ctx.moveTo(action.startX + radius, action.startY);
                    ctx.arcTo(action.endX, action.startY, action.endX, action.startY + radius, radius);
                    ctx.arcTo(action.endX, action.endY, action.endX - radius, action.endY, radius);
                    ctx.arcTo(action.startX, action.endY, action.startX, action.endY - radius, radius);
                    ctx.arcTo(action.startX, action.startY, action.startX + radius, action.startY, radius);
                    ctx.closePath();
                } else {
                    ctx.rect(action.startX, action.startY, action.endX - action.startX, action.endY - action.startY);
                }
                ctx.stroke();
                if (action.fill) {
                    ctx.fill();
                }
            } else if (action.tool === "circle") {
                const radius = Math.sqrt(
                    Math.pow(action.endX - action.startX, 2) +
                        Math.pow(action.endY - action.startY, 2),
                );
                ctx.arc(action.startX, action.startY, radius, 0, Math.PI * 2);
                // if (fill && isShapeClosed(action)) {
                //     ctx.fill();
                // } else {
                //     ctx.stroke();
                // }
                if (action.fill) {
                    ctx.fill();
                }
            } else if (action.tool === "curve") {
                ctx.moveTo(action.startX, action.startY);
                ctx.quadraticCurveTo(
                    action.controlX,
                    action.controlY,
                    action.endX,
                    action.endY,
                );
                // if (fill && isShapeClosed(action)) {
                //     ctx.fill();
                // } else {
                //     ctx.stroke();
                // }
                if (action.fill) {
                    ctx.fill();
                }
            } else if (action.tool === "arrow") {
                const headlen = 10; // length of head in pixels
                const angle = Math.atan2(
                    action.endY - action.startY,
                    action.endX - action.startX,
                );
                ctx.moveTo(action.startX, action.startY);
                ctx.lineTo(action.endX, action.endY);
                ctx.lineTo(
                    action.endX - headlen * Math.cos(angle - Math.PI / 6),
                    action.endY - headlen * Math.sin(angle - Math.PI / 6),
                );
                ctx.moveTo(action.endX, action.endY);
                ctx.lineTo(
                    action.endX - headlen * Math.cos(angle + Math.PI / 6),
                    action.endY - headlen * Math.sin(angle + Math.PI / 6),
                );
                ctx.stroke();
            } else if (action.tool === "freehand") {
                for (let i = 0; i < action.points.length - 1; i++) {
                    ctx.moveTo(action.points[i].x, action.points[i].y);
                    ctx.lineTo(action.points[i + 1].x, action.points[i + 1].y);
                }
                // if (fill && isShapeClosed(action)) {
                //     ctx.fill();
                // } else {
                //     ctx.stroke();
                // }
                ctx.stroke();
            } else if (action.tool === "text") {
                ctx.font = `${action.lineWidth * 10}px Arial`;
                ctx.fillText(action.text, action.startX, action.startY);
            }

            ctx.stroke();
            ctx.closePath();
        });
        if (showGrid) {
            drawGrid();
        }
    }

    function isShapeClosed(action) {
        if (action.tool === "freehand") {
            const startPoint = action.points[0];
            const endPoint = action.points[action.points.length - 1];
            const distance = Math.sqrt(
                Math.pow(endPoint.x - startPoint.x, 2) +
                    Math.pow(endPoint.y - startPoint.y, 2),
            );
            return distance < 10; // A small threshold to consider the shape closed
        }
        if (
            action.tool === "line" ||
            action.tool === "rectangle" ||
            action.tool === "circle" ||
            action.tool === "curve"
        ) {
            return true; // Consider these shapes always closed
        }
        return false;
    }

    function redrawCanvas() {
        ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
        ctx.save();
        ctx.translate(panX, panY);
        ctx.scale(scale, scale);
        drawGrid(ctx);
        drawActions();
        ctx.restore();
    }

    onMount(() => {
        ctx = document.getElementById("canvas").getContext("2d");
        redrawCanvas();
        // canvas.addEventListener("mousedown", startDrawing);
        // canvas.addEventListener("mousemove", draw);
        // canvas.addEventListener("mouseup", stopDrawing);
    });

    function drawGrid(ctx) {
        const width = ctx.canvas.width / scale;
        const height = ctx.canvas.height / scale;
        const step = 20;

        ctx.strokeStyle = getComputedStyle(
            document.documentElement,
        ).getPropertyValue("--grid-color");
        ctx.lineWidth = 0.5;

        for (let x = step; x < width; x += step) {
            ctx.beginPath();
            ctx.moveTo(x, 0);
            ctx.lineTo(x, height);
            ctx.stroke();
        }

        for (let y = step; y < height; y += step) {
            ctx.beginPath();
            ctx.moveTo(0, y);
            ctx.lineTo(width, y);
            ctx.stroke();
        }
    }

    function fillShape(x, y) {
        const imageData = ctx.getImageData(
            0,
            0,
            ctx.canvas.width,
            ctx.canvas.height,
        );
        const data = imageData.data;
        const stack = [{ x, y }];
        const targetColor = getColorAtPixel(data, x, y);
        const fillColor = hexToRgb(color);

        if (colorsMatch(targetColor, fillColor)) {
            return;
        }

        while (stack.length) {
            const { x, y } = stack.pop();
            const currentColor = getColorAtPixel(data, x, y);

            if (colorsMatch(currentColor, targetColor)) {
                setColorAtPixel(data, x, y, fillColor);

                stack.push({ x: x + 1, y });
                stack.push({ x: x - 1, y });
                stack.push({ x, y: y + 1 });
                stack.push({ x, y: y - 1 });
            }
        }

        ctx.putImageData(imageData, 0, 0);
    }

    function getColorAtPixel(data, x, y) {
        const index = (y * ctx.canvas.width + x) * 4;
        return [data[index], data[index + 1], data[index + 2], data[index + 3]];
    }

    function setColorAtPixel(data, x, y, color) {
        const index = (y * ctx.canvas.width + x) * 4;
        data[index] = color[0];
        data[index + 1] = color[1];
        data[index + 2] = color[2];
        data[index + 3] = 255;
    }

    function colorsMatch(a, b) {
        return a[0] === b[0] && a[1] === b[1] && a[2] === b[2] && a[3] === b[3];
    }

    function hexToRgb(hex) {
        const bigint = parseInt(hex.slice(1), 16);
        const r = (bigint >> 16) & 255;
        const g = (bigint >> 8) & 255;
        const b = bigint & 255;
        return [r, g, b];
    }

    // Toolbar
    function undo() {
        const lastAction = actions.pop();
        if (lastAction) undoStack.push(lastAction);
        redrawCanvas();
    }

    function redo() {
        const lastUndo = undoStack.pop();
        if (lastUndo) actions.push(lastUndo);
        redrawCanvas();
    }

    function saveCanvas() {
        const link = document.createElement("a");
        link.download = "drawing.png";
        link.href = document.getElementById("canvas").toDataURL();
        link.click();
    }

    function printCanvas() {
        const dataUrl = document.getElementById("canvas").toDataURL();
        let windowContent = "<!DOCTYPE html>";
        windowContent += "<html>";
        windowContent += "<head><title>Print canvas</title>";
        windowContent += "<style>";
        windowContent +=
            "body { margin: 0; padding: 0; display: flex; justify-content: center; align-items: center; height: 100vh; }";
        windowContent += "img { max-width: 100%; max-height: 100%; }";
        windowContent += "</style>";
        windowContent += "</head>";
        windowContent += "<body>";
        windowContent += `<img src="${dataUrl}">`;
        windowContent += "</body>";
        windowContent += "</html>";
        const printWin = window.open("", "", "width=600,height=400");
        printWin.document.open();
        printWin.document.write(windowContent);
        printWin.document.close();
        printWin.focus();
        printWin.onload = function () {
            printWin.print();
            printWin.onafterprint = function () {
                printWin.close();
            };
        };
    }

    function newCanvas() {
        if (
            confirm(
                "Are you sure you want to create a new canvas? Unsaved changes will be lost.",
            )
        ) {
            actions = [];
            undoStack = [];
            redrawCanvas();
        }
    }
    

    function shareCanvas() {
        if (navigator.share) {
            document.getElementById("canvas").toBlob((blob) => {
                const file = new File([blob], "drawing.png", {
                    type: "image/png",
                });
                navigator
                    .share({
                        title: "My Drawing",
                        files: [file],
                    })
                    .catch(console.error);
            });
        } else {
            alert("Sharing is not supported in this browser.");
        }
    }

    function zoomIn() {
        scale *= 1.1;
        redrawCanvas();
    }

    function zoomOut() {
        scale /= 1.1;
        redrawCanvas();
    }

    function handleDashedStrokeToggle(event) {
        dashedStroke = event.target.checked;
    }
    function handleRoundCornersCheckbox(event) {
        roundCorners = event.target.checked;
    }

    // function drawGrid() {
    //     ctx.strokeStyle = "#ccc";
    //     ctx.lineWidth = 0.5;
    //     const gridSize = 50;

    //     // Vertical grid lines
    //     for (let i = gridSize; i < ctx.canvas.width; i += gridSize) {
    //         ctx.beginPath();
    //         ctx.moveTo(i, 0);
    //         ctx.lineTo(i, ctx.canvas.height);
    //         ctx.stroke();
    //     }

    //     // Horizontal grid lines
    //     for (let i = gridSize; i < ctx.canvas.height; i += gridSize) {
    //         ctx.beginPath();
    //         ctx.moveTo(0, i);
    //         ctx.lineTo(ctx.canvas.width, i);
    //         ctx.stroke();
    //     }
    // }

    function toggleGrid() {
        showGrid = !showGrid;
        redrawCanvas();
    }

    function toggleRulers() {
        showRulers = !showRulers;
        redrawCanvas();
    }
</script>

<div class="app">
    <div class="drawing-container">
        <div class="controls">
            <div>
                <label
                    >Color: <input
                        type="color"
                        bind:value={strokeColor}
                    /></label
                >
            </div>
            <div>
                <label
                    >Line Width: <input
                        type="number"
                        min="1"
                        max="10"
                        bind:value={lineWidth}
                    /></label
                >
            </div>
            <div>
                <label
                    >Color: <input type="color" bind:value={fillColor} /></label
                >
            </div>
            <div>
                <label
                    >Fill: <input type="checkbox" bind:checked={fill} /></label
                >
                <label>
                    <input type="checkbox" on:change="{handleDashedStrokeToggle}" /> Dashed Stroke
                </label>
                <label>
                    <input type="checkbox" id="round-corners-checkbox" checked={roundCorners} onchange={handleRoundCornersCheckbox} />
                    Round Corners
                </label>
            </div>
            <div>
                <div>
                    <label>
                        <input type="checkbox" bind:checked={showGrid} on:change={toggleGrid} />
                        Show Grid
                    </label>
                </div>
                
                <div>
                    <label>
                        <input type="checkbox" bind:checked={showRulers} on:change={toggleRulers} />
                        Show Rulers
                    </label>
                </div>
            </div>
        </div>

        <canvas
            id="canvas"
            width="1000"
            height="800"
            on:mousedown={startDrawing}
            on:mouseup={stopDrawing}
            on:mousemove={draw}
        ></canvas>
    </div>
    <div class="toolbar">
        <button on:click={newCanvas}>New</button>
        <button on:click={saveCanvas}>Save</button>
        <button on:click={printCanvas}>Print</button>
        <button on:click={shareCanvas}>Share</button>
        <button on:click={undo}>Undo</button>
        <button on:click={redo}>Redo</button>
        <button on:click={zoomIn}>Zoom In</button>
        <button on:click={zoomOut}>Zoom Out</button>
    </div>
    <div class="toolbar1">
        <button
            on:click={() => (currentTool = "line")}
            class:active={currentTool === "line"}>Line</button
        >
        <button
            on:click={() => (currentTool = "rectangle")}
            class:active={currentTool === "rectangle"}>Rectangle</button
        >
        <button
            on:click={() => (currentTool = "circle")}
            class:active={currentTool === "circle"}>Circle</button
        >
        <button
            on:click={() => (currentTool = "curve")}
            class:active={currentTool === "curve"}>Curve</button
        >
        <button
            on:click={() => (currentTool = "arrow")}
            class:active={currentTool === "arrow"}>Arrow</button
        >
        <button
            on:click={() => (currentTool = "freehand")}
            class:active={currentTool === "freehand"}>Freehand</button
        >
        <button
            on:click={() => (currentTool = "text")}
            class:active={currentTool === "text"}>Text</button
        >
    </div>
</div>

<style>
    :root {
        --background-color: #fff;
        --text-color: #000;
        --toolbar-color: #f0f0f0;
        --grid-color: #e0e0e0;
        --button-color: #fff;
        --active-button-color: #ddd;
        --border-color: #ddd;
    }

    body {
        background-color: var(--background-color);
        color: var(--text-color);
        font-family: Arial, sans-serif;
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
        margin: 0;
    }

    .toolbar1 button.active,
    .controls button.active {
        background-color: var(--active-button-color);
    }

    .dark-mode-toggle {
        position: absolute;
        top: 20px;
        right: 20px;
        background: var(--button-color);
        border: 1px solid var(--border-color);
        padding: 10px;
        cursor: pointer;
    }

    .app {
        display: flex;
        align-items: flex-start;
    }

    .toolbar,
    .toolbar1 {
        display: flex;
        flex-direction: column;
        background-color: var(--toolbar-color);
        padding: 10px;
        border-left: 1px solid var(--border-color);
        margin-left: 10px;
    }

    .toolbar button,
    .toolbar1 button {
        margin: 5px 0;
        padding: 10px;
        border: 1px solid var(--border-color);
        background-color: var(--button-color);
        cursor: pointer;
    }

    .toolbar button.active,
    .toolbar1 button.active {
        background-color: var(--active-button-color);
    }

    .drawing-container {
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .controls {
        display: flex;
        flex-direction: row;
        padding: 10px;
        background-color: var(--toolbar-color);
        border: 1px solid var(--border-color);
        margin-bottom: 20px;
    }

    canvas {
        border: 1px solid var(--text-color);
    }
</style>
