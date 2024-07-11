<script>
	import { onMount } from 'svelte';
  
	let canvas;
	let ctx;
	let drawing = false;
	let tool = 'line';
	let gridSize = 100;
	let fineGridSize = 20;
	let scale = 1;
	let snapToGrid = true;
	let startX, startY;
	let endX, endY;
	let controlX, controlY;
	let shapes = [];
  
	onMount(() => {
	  canvas.width = canvas.offsetWidth;
	  canvas.height = canvas.offsetHeight;
	  ctx = canvas.getContext('2d');
	  drawGrid();
	  canvas.style.cursor = 'crosshair';
	});
  
	function drawGrid() {
	  ctx.clearRect(0, 0, canvas.width, canvas.height);
	  ctx.save();
	  ctx.scale(scale, scale);

	  ctx.strokeStyle = '#add8e6';
	  for (let x = 0; x < canvas.width / scale; x += fineGridSize) {
		ctx.beginPath();
		ctx.moveTo(x, 0);
		ctx.lineTo(x, canvas.height / scale);
		ctx.stroke();
	  }
	  for (let y = 0; y < canvas.height / scale; y += fineGridSize) {
		ctx.beginPath();
		ctx.moveTo(0, y);
		ctx.lineTo(canvas.width / scale, y);
		ctx.stroke();
	  }
  
	
	  ctx.strokeStyle = '#2DCAEA';
	  for (let x = 0; x < canvas.width / scale; x += gridSize) {
		ctx.beginPath();
		ctx.moveTo(x, 0);
		ctx.lineTo(x, canvas.height / scale);
		ctx.stroke();
	  }
	  for (let y = 0; y < canvas.height / scale; y += gridSize) {
		ctx.beginPath();
		ctx.moveTo(0, y);
		ctx.lineTo(canvas.width / scale, y);
		ctx.stroke();
	  }
  
	  ctx.restore();
	}
  
	function startDrawing(event) {
	  drawing = true;
	  startX = snapToGrid ? Math.round(event.offsetX / gridSize) * gridSize : event.offsetX;
	  startY = snapToGrid ? Math.round(event.offsetY / gridSize) * gridSize : event.offsetY;
	  if (tool === 'line' || tool === 'curve') {
		ctx.beginPath();
		ctx.moveTo(startX, startY);
	  }
	}
  
	function draw(event) {
	  if (!drawing) return;
	  endX = snapToGrid ? Math.round(event.offsetX / gridSize) * gridSize : event.offsetX;
	  endY = snapToGrid ? Math.round(event.offsetY / gridSize) * gridSize : event.offsetY;
  
	  ctx.clearRect(0, 0, canvas.width, canvas.height);
	  drawGrid();
	  redrawShapes();
  
	  ctx.strokeStyle = 'black';
	  ctx.beginPath();
	  if (tool === 'line') {
		ctx.moveTo(startX, startY);
		ctx.lineTo(endX, endY);
	  } else if (tool === 'rectangle') {
		ctx.strokeRect(startX, startY, endX - startX, endY - startY);
	  } else if (tool === 'circle') {
		let radius = Math.sqrt(Math.pow(endX - startX, 2) + Math.pow(endY - startY, 2));
		ctx.beginPath();
		ctx.arc(startX, startY, radius, 0, 2 * Math.PI);
	  } else if (tool === 'curve') {
		controlX = (startX + endX) / 2;
		controlY = startY - 50;
		ctx.beginPath();
		ctx.moveTo(startX, startY);
		ctx.quadraticCurveTo(controlX, controlY, endX, endY);
	  }
	  ctx.stroke();
  
	  ctx.fillStyle = 'green';
	  ctx.beginPath();
	  ctx.arc(endX, endY, 3, 0, Math.PI * 2);
	  ctx.fill();
	}
  
	function stopDrawing(event) {
	  if (!drawing) return;
	  drawing = false;
	  endX = snapToGrid ? Math.round(event.offsetX / gridSize) * gridSize : event.offsetX;
	  endY = snapToGrid ? Math.round(event.offsetY / gridSize) * gridSize : event.offsetY;
	  shapes.push({ tool, startX, startY, endX, endY, controlX, controlY });
	  redrawShapes();
	  ctx.closePath();
	}
  
	function redrawShapes() {
	  drawGrid();
	  for (let shape of shapes) {
		ctx.strokeStyle = 'black';
		ctx.beginPath();
		if (shape.tool === 'line') {
		  ctx.moveTo(shape.startX, shape.startY);
		  ctx.lineTo(shape.endX, shape.endY);
		} else if (shape.tool === 'rectangle') {
		  ctx.strokeRect(shape.startX, shape.startY, shape.endX - shape.startX, shape.endY - shape.startY);
		} else if (shape.tool === 'circle') {
		  let radius = Math.sqrt(Math.pow(shape.endX - shape.startX, 2) + Math.pow(shape.endY - shape.startY, 2));
		  ctx.beginPath();
		  ctx.arc(shape.startX, shape.startY, radius, 0, 2 * Math.PI);
		} else if (shape.tool === 'curve') {
		  ctx.moveTo(shape.startX, shape.startY);
		  ctx.quadraticCurveTo(shape.controlX, shape.controlY, shape.endX, shape.endY);
		}
		ctx.stroke();
	  }
	}
  
	function selectTool(selectedTool) {
	  tool = selectedTool;
	}
  
	function toggleSnapToGrid() {
	  snapToGrid = !snapToGrid;
	}
</script>

<div class="relative bg-white border rounded-lg overflow-hidden" style="width: 100%; height: 550px;">
	<div class="absolute top-0 left-0 z-10 p-2">
	  <select on:change={(e) => selectTool(e.target.value)} class="px-4 py-2 bg-grey text-5B6669 rounded hover:bg-grey-700 m-1">
		<option value="null">Tools</option>
		<option value="line">Line</option>
		<option value="rectangle">Rectangle</option>
		<option value="circle">Circle</option>
		<option value="curve">Curve</option>
		<!-- Add other tool options here -->
	  </select>
	  <button on:click={toggleSnapToGrid} class="px-4 py-2 bg-E3EBED text-5B6669 rounded hover:bg-grey-700 m-1">
		Snap to Grid: {snapToGrid ? 'On' : 'Off'}
	  </button>
	</div>
	<canvas
	  bind:this={canvas}
	  on:mousedown={startDrawing}
	  on:mousemove={draw}
	  on:mouseup={stopDrawing}
	  on:mouseleave={stopDrawing}
	  class="w-full h-full"
	></canvas>
</div>

<style>
	canvas {
	  cursor: crosshair;
	}
</style>
