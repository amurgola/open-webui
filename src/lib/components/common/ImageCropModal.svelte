<script lang="ts">
	import { onMount, createEventDispatcher } from 'svelte';
	import { fade } from 'svelte/transition';
	import { flyAndScale } from '$lib/utils/transitions';
	import * as FocusTrap from 'focus-trap';
	
	const dispatch = createEventDispatcher();

	export let show = false;
	export let imageUrl = '';
	export let fileName = '';

	let modalElement: HTMLElement | null = null;
	let canvasElement: HTMLCanvasElement | null = null;
	let imageElement: HTMLImageElement | null = null;
	let focusTrap: FocusTrap.FocusTrap | null = null;

	// Crop state
	let isDragging = false;
	let isResizing = false;
	let cropX = 50;
	let cropY = 50;
	let cropWidth = 200;
	let cropHeight = 200;
	let rotation = 0;
	let startX = 0;
	let startY = 0;
	let originalCropX = 0;
	let originalCropY = 0;
	let originalCropWidth = 0;
	let originalCropHeight = 0;

	// Image dimensions
	let imageWidth = 0;
	let imageHeight = 0;
	let displayWidth = 0;
	let displayHeight = 0;
	let scale = 1;

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Escape') {
			cancel();
		}
	};

	const loadImage = () => {
		if (!imageElement || !imageUrl) return;
		
		imageElement.onload = () => {
			imageWidth = imageElement.naturalWidth;
			imageHeight = imageElement.naturalHeight;
			
			// Calculate display size (max 600px width/height)
			const maxSize = 600;
			scale = Math.min(maxSize / imageWidth, maxSize / imageHeight, 1);
			displayWidth = imageWidth * scale;
			displayHeight = imageHeight * scale;
			
			// Initialize crop area in center (25% of image)
			cropWidth = Math.min(displayWidth * 0.5, 200);
			cropHeight = Math.min(displayHeight * 0.5, 200);
			cropX = (displayWidth - cropWidth) / 2;
			cropY = (displayHeight - cropHeight) / 2;
			
			redrawCanvas();
		};
		imageElement.src = imageUrl;
	};

	const redrawCanvas = () => {
		if (!canvasElement || !imageElement) return;
		
		const ctx = canvasElement.getContext('2d');
		canvasElement.width = displayWidth;
		canvasElement.height = displayHeight;
		
		ctx.clearRect(0, 0, displayWidth, displayHeight);
		
		// Save context for rotation
		ctx.save();
		
		// Apply rotation around center
		ctx.translate(displayWidth / 2, displayHeight / 2);
		ctx.rotate((rotation * Math.PI) / 180);
		ctx.translate(-displayWidth / 2, -displayHeight / 2);
		
		// Draw image
		ctx.drawImage(imageElement, 0, 0, displayWidth, displayHeight);
		
		// Restore context
		ctx.restore();
		
		// Draw crop overlay
		ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
		ctx.fillRect(0, 0, displayWidth, displayHeight);
		
		// Clear crop area
		ctx.clearRect(cropX, cropY, cropWidth, cropHeight);
		
		// Draw crop border
		ctx.strokeStyle = '#3b82f6';
		ctx.lineWidth = 2;
		ctx.setLineDash([]);
		ctx.strokeRect(cropX, cropY, cropWidth, cropHeight);
		
		// Draw resize handles
		const handleSize = 8;
		ctx.fillStyle = '#3b82f6';
		
		// Corner handles
		ctx.fillRect(cropX - handleSize/2, cropY - handleSize/2, handleSize, handleSize);
		ctx.fillRect(cropX + cropWidth - handleSize/2, cropY - handleSize/2, handleSize, handleSize);
		ctx.fillRect(cropX - handleSize/2, cropY + cropHeight - handleSize/2, handleSize, handleSize);
		ctx.fillRect(cropX + cropWidth - handleSize/2, cropY + cropHeight - handleSize/2, handleSize, handleSize);
	};

	const getEventPos = (e: MouseEvent | TouchEvent) => {
		const rect = canvasElement!.getBoundingClientRect();
		let clientX: number;
		let clientY: number;
		
		if (e instanceof MouseEvent) {
			clientX = e.clientX;
			clientY = e.clientY;
		} else {
			// TouchEvent
			const touch = e.touches[0] || e.changedTouches[0];
			clientX = touch.clientX;
			clientY = touch.clientY;
		}
		
		return {
			x: clientX - rect.left,
			y: clientY - rect.top
		};
	};

	const isInResizeHandle = (mouseX: number, mouseY: number) => {
		const handleSize = 8;
		const handles = [
			{ x: cropX, y: cropY },
			{ x: cropX + cropWidth, y: cropY },
			{ x: cropX, y: cropY + cropHeight },
			{ x: cropX + cropWidth, y: cropY + cropHeight }
		];
		
		return handles.some(handle => 
			mouseX >= handle.x - handleSize/2 && mouseX <= handle.x + handleSize/2 &&
			mouseY >= handle.y - handleSize/2 && mouseY <= handle.y + handleSize/2
		);
	};

	const isInCropArea = (mouseX: number, mouseY: number) => {
		return mouseX >= cropX && mouseX <= cropX + cropWidth &&
			   mouseY >= cropY && mouseY <= cropY + cropHeight;
	};

	const onPointerDown = (e: MouseEvent | TouchEvent) => {
		e.preventDefault();
		const { x, y } = getEventPos(e);
		startX = x;
		startY = y;
		originalCropX = cropX;
		originalCropY = cropY;
		originalCropWidth = cropWidth;
		originalCropHeight = cropHeight;
		
		if (isInResizeHandle(x, y)) {
			isResizing = true;
		} else if (isInCropArea(x, y)) {
			isDragging = true;
		}
	};

	const onPointerMove = (e: MouseEvent | TouchEvent) => {
		if (!isDragging && !isResizing) return;
		
		e.preventDefault();
		const { x, y } = getEventPos(e);
		const deltaX = x - startX;
		const deltaY = y - startY;
		
		if (isDragging) {
			cropX = Math.max(0, Math.min(displayWidth - cropWidth, originalCropX + deltaX));
			cropY = Math.max(0, Math.min(displayHeight - cropHeight, originalCropY + deltaY));
		} else if (isResizing) {
			// Simple resize from bottom-right corner
			const newWidth = Math.max(50, originalCropWidth + deltaX);
			const newHeight = Math.max(50, originalCropHeight + deltaY);
			
			cropWidth = Math.min(newWidth, displayWidth - cropX);
			cropHeight = Math.min(newHeight, displayHeight - cropY);
		}
		
		redrawCanvas();
	};

	const onPointerUp = (e: MouseEvent | TouchEvent) => {
		e.preventDefault();
		isDragging = false;
		isResizing = false;
	};

	const rotate90 = () => {
		rotation = (rotation + 90) % 360;
		redrawCanvas();
	};

	const setRotation = (newRotation: number) => {
		rotation = ((newRotation % 360) + 360) % 360; // Ensure positive angle between 0-359
		redrawCanvas();
	};

	const getCroppedImage = (): string | null => {
		if (!imageElement) return null;
		
		// Create a new canvas for the final cropped image
		const outputCanvas = document.createElement('canvas');
		const outputCtx = outputCanvas.getContext('2d');
		
		if (!outputCtx) return null;
		
		// Calculate actual crop coordinates on original image
		const actualCropX = (cropX / scale);
		const actualCropY = (cropY / scale);
		const actualCropWidth = (cropWidth / scale);
		const actualCropHeight = (cropHeight / scale);
		
		outputCanvas.width = actualCropWidth;
		outputCanvas.height = actualCropHeight;
		
		// Apply rotation if needed
		if (rotation !== 0) {
			outputCtx.save();
			outputCtx.translate(actualCropWidth / 2, actualCropHeight / 2);
			outputCtx.rotate((rotation * Math.PI) / 180);
			outputCtx.translate(-actualCropWidth / 2, -actualCropHeight / 2);
		}
		
		// Draw the cropped portion
		outputCtx.drawImage(
			imageElement,
			actualCropX, actualCropY, actualCropWidth, actualCropHeight,
			0, 0, actualCropWidth, actualCropHeight
		);
		
		if (rotation !== 0) {
			outputCtx.restore();
		}
		
		return outputCanvas.toDataURL('image/png');
	};

	const accept = () => {
		const croppedImageUrl = getCroppedImage();
		if (croppedImageUrl) {
			dispatch('accept', { 
				imageUrl: croppedImageUrl,
				fileName: fileName || 'cropped-image.png'
			});
		}
		show = false;
	};

	const cancel = () => {
		dispatch('cancel');
		show = false;
	};

	$: if (show && modalElement) {
		document.body.appendChild(modalElement);
		focusTrap = FocusTrap.createFocusTrap(modalElement);
		focusTrap.activate();
		window.addEventListener('keydown', handleKeyDown);
		document.body.style.overflow = 'hidden';
		loadImage();
	} else if (modalElement) {
		if (focusTrap) {
			focusTrap.deactivate();
		}
		window.removeEventListener('keydown', handleKeyDown);
		document.body.removeChild(modalElement);
		document.body.style.overflow = 'unset';
	}

	onMount(() => {
		// Add global pointer event listeners for better mobile support
		const addPointerListeners = () => {
			document.addEventListener('mousemove', onPointerMove);
			document.addEventListener('mouseup', onPointerUp);
			document.addEventListener('touchmove', onPointerMove, { passive: false });
			document.addEventListener('touchend', onPointerUp);
		};
		
		const removePointerListeners = () => {
			document.removeEventListener('mousemove', onPointerMove);
			document.removeEventListener('mouseup', onPointerUp);
			document.removeEventListener('touchmove', onPointerMove);
			document.removeEventListener('touchend', onPointerUp);
		};
		
		addPointerListeners();
		
		return removePointerListeners;
	});
</script>

{#if show}
	<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
	<div
		bind:this={modalElement}
		aria-modal="true"
		role="dialog"
		class="modal fixed top-0 right-0 left-0 bottom-0 bg-black/60 w-full h-screen max-h-[100dvh] flex justify-center z-9999 overflow-y-auto overscroll-contain"
		in:fade={{ duration: 10 }}
		on:mousedown={(e) => {
			if (e.target === modalElement) {
				cancel();
			}
		}}
	>
		<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
		<div
			class="m-auto max-w-full w-[50rem] mx-2 shadow-3xl min-h-fit bg-white dark:bg-gray-900 rounded-2xl"
			in:flyAndScale
			role="dialog"
			on:mousedown={(e) => {
				e.stopPropagation();
			}}
		>
			<div class="px-6 py-4 border-b border-gray-200 dark:border-gray-700">
				<div class="flex justify-between items-center">
					<h2 class="text-lg font-semibold dark:text-gray-200">Crop & Rotate Image</h2>
					<button
						on:click={cancel}
						class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200"
					>
						<svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
							<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
						</svg>
					</button>
				</div>
			</div>

			<div class="p-6">
				<div class="flex flex-col items-center space-y-4">
					<div class="relative">
						<canvas
							bind:this={canvasElement}
							on:mousedown={onPointerDown}
							on:touchstart={onPointerDown}
							class="border border-gray-300 dark:border-gray-600 cursor-crosshair max-w-full touch-none"
							style="touch-action: none;"
						/>
						<img bind:this={imageElement} class="hidden" alt="Source" />
					</div>

					<div class="flex flex-col space-y-3">
						<div class="flex space-x-3">
							<button
								on:click={rotate90}
								class="px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-800 dark:hover:bg-gray-700 dark:text-white rounded-lg transition flex items-center space-x-2"
							>
								<svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"/>
								</svg>
								<span>Rotate 90°</span>
							</button>
						</div>
						
						<div class="flex items-center space-x-3">
							<label for="rotation-input" class="text-sm font-medium text-gray-700 dark:text-gray-300">Fine Rotation:</label>
							<input
								type="range"
								min="0"
								max="359"
								bind:value={rotation}
								on:input={(e) => setRotation(Number(e.target.value))}
								class="flex-1 h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer dark:bg-gray-700"
								aria-label="Rotation slider"
							/>
							<input
								id="rotation-input"
								type="number"
								min="0"
								max="359"
								bind:value={rotation}
								on:input={(e) => setRotation(Number(e.target.value))}
								class="w-16 px-2 py-1 text-sm border border-gray-300 dark:border-gray-600 rounded bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100"
							/>
							<span class="text-sm text-gray-600 dark:text-gray-400">°</span>
						</div>
					</div>

					<div class="text-sm text-gray-600 dark:text-gray-400 text-center">
						Drag to move crop area • Drag corners to resize • Use controls to rotate as needed
					</div>
				</div>
			</div>

			<div class="px-6 py-4 border-t border-gray-200 dark:border-gray-700">
				<div class="flex justify-end space-x-3">
					<button
						on:click={cancel}
						class="px-4 py-2 bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-850 dark:hover:bg-gray-800 dark:text-white font-medium rounded-lg transition"
					>
						Cancel
					</button>
					<button
						on:click={accept}
						class="px-4 py-2 bg-gray-900 hover:bg-gray-850 text-gray-100 dark:bg-gray-100 dark:hover:bg-white dark:text-gray-800 font-medium rounded-lg transition"
					>
						Accept
					</button>
				</div>
			</div>
		</div>
	</div>
{/if}