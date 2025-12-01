<script lang="ts">
	// ============================================================================
	// SLIPPYMUDWHEEL - Movie Wheel Spinner
	// ============================================================================

	import { onMount, onDestroy } from 'svelte';
	import arcticBg from '$lib/arctic_bg.jpg';
	import penguinImg from '$lib/penguin.jpg';

	// TODO: Import stores (once created)
	// import { movies } from '$lib/stores/movies';
	// import { wheelState } from '$lib/stores/wheelState';
	// import { spinHistory } from '$lib/stores/history';

	// TODO: Import utility functions (once created)
	// import { fetchMoviesFromGoogleDocs } from '$lib/utils/googleDocs';
	// import { getLetterboxdSearchUrl } from '$lib/utils/letterboxd';

	// TODO: Import components (once created)
	// import Wheel from '$lib/components/Wheel.svelte';
	// import ResultsModal from '$lib/components/ResultsModal.svelte';
	// import HistorySidebar from '$lib/components/HistorySidebar.svelte';

	// ============================================================================
	// STATE MANAGEMENT (Svelte 5 Runes)
	// ============================================================================

	// Google Docs link state
	let googleDocsLink = $state('');
	let rememberLink = $state(false);
	let isLinkLocked = $state(false);

	// Theme settings modal state
	let showThemeSettings = $state(false);
	let isClosingThemeSettings = $state(false);

	// Theme state - Arctic theme (defaults)
	let primaryColor = $state('#1E90FF'); // Dodger blue
	let secondaryColor = $state('#FFFFFF'); // White
	let wheelBackground = $state('#E0F2FF'); // Light ice blue
	let selectedFont = $state('Arial');
	let invertBW = $state(false);
	let wheelColors = $state(['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6']);
	let backgroundWallpaper = $state('arctic_bg.jpg'); // Arctic background
	let wheelCenterImage = $state('penguin.jpg'); // Penguin center image

	// Test data - numbers 1-10 written out
	let movies = $state(['one', 'two', 'three', 'four', 'five', 'six', 'seven', 'eight', 'nine', 'ten']);

	// Wheel state
	let wheelCanvas: HTMLCanvasElement;
	let rotation = $state(0);
	let isSpinning = $state(false);

	// Preload penguin image
	let penguinImage: HTMLImageElement | null = null;
	if (typeof window !== 'undefined') {
		penguinImage = new Image();
		penguinImage.src = penguinImg;
	}

	// Results modal state
	let showResultsModal = $state(false);
	let currentResults = $state<string[]>([]);
	let numberOfSpins = $state(1);
	let customSpinCount = $state('');

	// Spin history state
	type HistoryItem = {
		id: string;
		movie: string;
		timestamp: number;
	};
	let spinHistory = $state<HistoryItem[]>([]);

	// ============================================================================
	// GOOGLE DOCS LINK MANAGEMENT
	// ============================================================================

	function handleRememberLinkChange() {
		if (rememberLink && googleDocsLink.trim()) {
			// Save link to localStorage and lock the input
			localStorage.setItem('slippymud_google_docs_link', googleDocsLink);
			isLinkLocked = true;
		} else {
			// Unlock and clear saved link
			localStorage.removeItem('slippymud_google_docs_link');
			isLinkLocked = false;
		}
	}

	function handleRefresh() {
		// TODO: Implement actual refresh from Google Docs
		alert('Refresh functionality coming soon!');
		console.log('Refreshing from link:', googleDocsLink);
	}

	// ============================================================================
	// THEME SETTINGS MANAGEMENT
	// ============================================================================

	function toggleThemeSettings() {
		if (showThemeSettings) {
			// Start closing animation
			isClosingThemeSettings = true;
			// Wait for animation to complete before hiding
			setTimeout(() => {
				showThemeSettings = false;
				isClosingThemeSettings = false;
			}, 300); // Match animation duration
		} else {
			showThemeSettings = true;
		}
	}

	// ============================================================================
	// WHEEL DRAWING FUNCTIONS
	// ============================================================================

	function drawWheel() {
		if (!wheelCanvas) return;

		const ctx = wheelCanvas.getContext('2d');
		if (!ctx) return;

		const centerX = wheelCanvas.width / 2;
		const centerY = wheelCanvas.height / 2;
		const radius = Math.min(centerX, centerY) - 10;

		// Clear canvas
		ctx.clearRect(0, 0, wheelCanvas.width, wheelCanvas.height);

		// Save context state
		ctx.save();

		// Rotate canvas
		ctx.translate(centerX, centerY);
		ctx.rotate((rotation * Math.PI) / 180);
		ctx.translate(-centerX, -centerY);

		// Draw wheel slices
		const sliceAngle = (2 * Math.PI) / movies.length;

		for (let i = 0; i < movies.length; i++) {
			const startAngle = i * sliceAngle;
			const endAngle = startAngle + sliceAngle;

			// Draw slice
			ctx.beginPath();
			ctx.moveTo(centerX, centerY);
			ctx.arc(centerX, centerY, radius, startAngle, endAngle);
			ctx.closePath();

			// Fill with alternating colors
			ctx.fillStyle = wheelColors[i % wheelColors.length];
			ctx.fill();

			// Draw border
			ctx.strokeStyle = '#ffffff';
			ctx.lineWidth = 2;
			ctx.stroke();

			// Draw text
			ctx.save();
			ctx.translate(centerX, centerY);
			ctx.rotate(startAngle + sliceAngle / 2);
			ctx.textAlign = 'right';
			ctx.fillStyle = '#000000';
			ctx.font = 'bold 16px Arial';
			ctx.fillText(movies[i], radius - 20, 10);
			ctx.restore();
		}

		// Draw center circle with penguin image
		const centerRadius = 60; // Increased from 40
		ctx.beginPath();
		ctx.arc(centerX, centerY, centerRadius, 0, 2 * Math.PI);
		ctx.fillStyle = '#ffffff';
		ctx.fill();
		ctx.strokeStyle = primaryColor;
		ctx.lineWidth = 3;
		ctx.stroke();

		// Draw penguin image if selected
		if (wheelCenterImage === 'penguin.jpg' && penguinImage && penguinImage.complete) {
			// Draw image centered in the circle, clipped to circle shape
			const imgRadius = centerRadius - 2;
			ctx.save();
			ctx.beginPath();
			ctx.arc(centerX, centerY, imgRadius, 0, 2 * Math.PI);
			ctx.clip();
			ctx.drawImage(penguinImage, centerX - imgRadius, centerY - imgRadius, imgRadius * 2, imgRadius * 2);
			ctx.restore();
		}

		// Restore context state
		ctx.restore();
	}

	// Idle rotation animation
	let animationFrame: number;
	function animateIdleRotation() {
		if (!isSpinning) {
			rotation += 0.1; // Slow rotation speed
			if (rotation >= 360) rotation = 0;
			drawWheel();
		}
		animationFrame = requestAnimationFrame(animateIdleRotation);
	}

	// ============================================================================
	// SPIN FUNCTIONALITY
	// ============================================================================

	let spinAnimationFrame: number;
	let spinStartTime = 0;
	let spinStartRotation = 0;
	let targetRotation = 0;
	const spinDuration = 5000; // 5 seconds in milliseconds

	function spin(count: number) {
		if (isSpinning || movies.length === 0) return;

		numberOfSpins = count;
		isSpinning = true;

		// Pick a random target item for the wheel to land on
		const randomTargetIndex = Math.floor(Math.random() * movies.length);

		// Calculate the rotation needed to land on this item at the ticker (top center)
		const sliceAngle = 360 / movies.length;

		// The ticker is at the top (12 o'clock), we need to rotate so the chosen slice is under it
		// Items are drawn starting from 0 degrees and going clockwise
		// To have an item at the top, we need to account for the slice position
		const targetSliceRotation = randomTargetIndex * sliceAngle;

		// Add multiple full spins for effect (5-8 rotations)
		const fullSpins = 5 + Math.random() * 3;

		// Calculate final target rotation
		// We want to align the CENTER of the target slice with the ticker
		const sliceCenterOffset = sliceAngle / 2;
		spinStartRotation = rotation;
		targetRotation = rotation + (fullSpins * 360) + (360 - targetSliceRotation - sliceCenterOffset + 90);

		spinStartTime = Date.now();

		animateSpin();
	}

	function getItemAtTicker(): string {
		// The ticker is at the top (12 o'clock = 270 degrees in canvas coords, or -90 from right)
		// Slices start at 0 radians (3 o'clock) and go counter-clockwise in canvas
		// But we rotate the canvas, so we need to account for the current rotation

		const sliceAngle = 360 / movies.length;
		const normalizedRotation = ((rotation % 360) + 360) % 360; // Ensure positive

		// The ticker is at 270 degrees (top of circle in standard canvas coords)
		// We need to find which slice is at 270 degrees considering the wheel's rotation
		// Since the wheel rotates clockwise (positive rotation), we subtract
		const tickerAngleInWheel = (270 - normalizedRotation + 360) % 360;

		const sliceIndex = Math.floor(tickerAngleInWheel / sliceAngle) % movies.length;

		return movies[sliceIndex];
	}

	function animateSpin() {
		if (!isSpinning) return;

		const now = Date.now();
		const elapsed = now - spinStartTime;
		const progress = Math.min(elapsed / spinDuration, 1); // 0 to 1

		// Easing function - ease out cubic for smooth deceleration
		const eased = 1 - Math.pow(1 - progress, 3);

		// Calculate current rotation based on eased progress
		rotation = spinStartRotation + (targetRotation - spinStartRotation) * eased;

		// Check if spin is complete (5 seconds have passed)
		if (progress >= 1) {
			rotation = targetRotation % 360;
			isSpinning = false;
			drawWheel();

			// Collect results based on actual ticker position
			const results: string[] = [];

			// ALWAYS get the first result from the ticker
			const tickerResult = getItemAtTicker();
			results.push(tickerResult);

			// If multi-spin, add additional UNIQUE random items
			if (numberOfSpins > 1) {
				const remainingMovies = movies.filter(m => m !== tickerResult);
				const additionalCount = numberOfSpins - 1;

				// Shuffle and pick unique items
				const shuffled = [...remainingMovies].sort(() => Math.random() - 0.5);
				for (let i = 0; i < Math.min(additionalCount, shuffled.length); i++) {
					results.push(shuffled[i]);
				}
			}

			currentResults = results;

			// Show results modal after a brief delay
			setTimeout(() => {
				showResultsModal = true;
			}, 300);
			return;
		}

		drawWheel();
		spinAnimationFrame = requestAnimationFrame(animateSpin);
	}

	// ============================================================================
	// THEME PERSISTENCE
	// ============================================================================

	function saveTheme() {
		const theme = {
			primaryColor,
			secondaryColor,
			wheelBackground,
			selectedFont,
			invertBW,
			wheelColors,
			backgroundWallpaper,
			wheelCenterImage
		};
		localStorage.setItem('slippymud_theme', JSON.stringify(theme));
	}

	function loadTheme() {
		const saved = localStorage.getItem('slippymud_theme');
		if (saved) {
			try {
				const theme = JSON.parse(saved);
				primaryColor = theme.primaryColor || primaryColor;
				secondaryColor = theme.secondaryColor || secondaryColor;
				wheelBackground = theme.wheelBackground || wheelBackground;
				selectedFont = theme.selectedFont || selectedFont;
				invertBW = theme.invertBW || invertBW;
				wheelColors = theme.wheelColors || wheelColors;
				backgroundWallpaper = theme.backgroundWallpaper || backgroundWallpaper;
				wheelCenterImage = theme.wheelCenterImage || wheelCenterImage;
			} catch (e) {
				console.error('Failed to load theme:', e);
			}
		}
	}

	// ============================================================================
	// LIFECYCLE - Load saved data from localStorage
	// ============================================================================

	onMount(() => {
		// Load saved theme
		loadTheme();

		// Start idle rotation animation
		animateIdleRotation();

		// Load saved Google Docs link if exists
		const savedLink = localStorage.getItem('slippymud_google_docs_link');
		if (savedLink) {
			googleDocsLink = savedLink;
			rememberLink = true;
			isLinkLocked = true;
		}
	});

	onDestroy(() => {
		// Clean up animation frame
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
		}
	});

	// ============================================================================
	// GOOGLE DOCS INTEGRATION
	// ============================================================================

	// TODO: Function to fetch movies from Google Docs
	// async function handleFetchMovies() {
	//   if (!googleDocsLink.trim()) {
	//     alert('Please enter a Google Docs link');
	//     return;
	//   }
	//
	//   isLoadingMovies = true;
	//
	//   try {
	//     // Fetch movies from Google Docs (line-separated, trim whitespace)
	//     const movieList = await fetchMoviesFromGoogleDocs(googleDocsLink);
	//     movies.set(movieList);
	//
	//     // Save link to localStorage if "remember" is checked
	//     if (rememberLink) {
	//       saveToStorage('slippymud_google_docs_link', googleDocsLink);
	//     } else {
	//       // Remove saved link if unchecked
	//       saveToStorage('slippymud_google_docs_link', null);
	//     }
	//   } catch (error) {
	//     console.error('Failed to fetch movies:', error);
	//     alert('Failed to load movies from Google Docs. Make sure the link is public and correct.');
	//   } finally {
	//     isLoadingMovies = false;
	//   }
	// }

	// ============================================================================
	// WHEEL SPINNING LOGIC
	// ============================================================================

	// TODO: Function to handle wheel spin
	// function handleSpin(spins: number) {
	//   if (isSpinning) return; // Prevent multiple spins at once
	//   if ($movies.length === 0) {
	//     alert('Please load movies first!');
	//     return;
	//   }
	//
	//   isSpinning = true;
	//   numberOfSpins = spins;
	//
	//   // Generate N random movie results
	//   const results: string[] = [];
	//   for (let i = 0; i < spins; i++) {
	//     const randomIndex = Math.floor(Math.random() * $movies.length);
	//     results.push($movies[randomIndex]);
	//   }
	//
	//   // Trigger wheel animation (Canvas component handles visual spin)
	//   wheelState.spin(spins);
	//
	//   // Wait for spin animation to complete (e.g., 3-5 seconds)
	//   setTimeout(() => {
	//     isSpinning = false;
	//     currentResults = results;
	//     showResultsModal = true;
	//
	//     // Play success sound (handle missing file gracefully)
	//     playSound('spin-complete');
	//
	//     // Add results to history (arbitrary order for multiple spins)
	//     results.forEach((movie) => {
	//       spinHistory.addSpin(movie);
	//     });
	//
	//     // Save history to localStorage
	//     saveToStorage('slippymud_spin_history', $spinHistory);
	//   }, 4000); // 4 second spin animation
	// }

	// TODO: Quick spin buttons (1, 2, 3, 6)
	// function spin1() { handleSpin(1); }
	// function spin2() { handleSpin(2); }
	// function spin3() { handleSpin(3); }
	// function spin6() { handleSpin(6); }

	// ============================================================================
	// AUDIO HANDLING
	// ============================================================================

	// TODO: Function to play sounds (handle missing files)
	// function playSound(soundName: string) {
	//   try {
	//     const soundFile = $theme.soundFile || `sounds/${soundName}.mp3`;
	//     const audio = new Audio(soundFile);
	//     audio.play().catch((err) => {
	//       console.warn('Sound file not found or cannot play:', err);
	//     });
	//   } catch (error) {
	//     console.warn('Audio playback failed:', error);
	//   }
	// }

	// ============================================================================
	// RESULTS MODAL HANDLERS
	// ============================================================================

	// TODO: Close results modal
	// function closeResultsModal() {
	//   showResultsModal = false;
	//   currentResults = [];
	// }

	// TODO: Open Letterboxd for a specific movie
	// function openLetterboxd(movieName: string) {
	//   const url = getLetterboxdSearchUrl(movieName);
	//   window.open(url, '_blank');
	// }

	// ============================================================================
	// THEME SETTINGS HANDLERS
	// ============================================================================

	// TODO: Toggle theme settings modal
	// function toggleThemeSettings() {
	//   showThemeSettings = !showThemeSettings;
	// }

	// TODO: Save theme settings to localStorage
	// function saveThemeSettings(newTheme: ThemeSettings) {
	//   theme.set(newTheme);
	//   saveToStorage('slippymud_theme', newTheme);
	// }
</script>

<!-- ============================================================================ -->
<!-- MAIN LAYOUT -->
<!-- ============================================================================ -->

<div class="app-container" style="font-family: {selectedFont}; background-color: {wheelBackground}; {backgroundWallpaper === 'arctic_bg.jpg' ? `background-image: url('${arcticBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : ''}">

	<header style="background-color: {primaryColor}; border-bottom-color: {primaryColor};">
		<h1 style="color: {secondaryColor};">SlippyMovieWheel</h1>
		<button class="theme-button" style="border-color: {secondaryColor}; color: {secondaryColor};" onclick={toggleThemeSettings}>Theme Settings</button>
	</header>

	<main>
		<!-- ==================================================================== -->
		<!-- GOOGLE DOCS INPUT SECTION -->
		<!-- ==================================================================== -->
		<section class="google-docs-input" style="border-color: {primaryColor};">
			<div class="link-input-container">
				<button class="refresh-button" style="border-color: {primaryColor}; color: {primaryColor};" onclick={handleRefresh} disabled={!googleDocsLink.trim()}>
					Refresh
				</button>
				<input
					type="text"
					class="link-input"
					style="border-color: {primaryColor};"
					bind:value={googleDocsLink}
					placeholder="Paste Google Docs sharing link here..."
					disabled={isLinkLocked}
				/>
			</div>

			<label class="checkbox-label">
				<input type="checkbox" bind:checked={rememberLink} onchange={handleRememberLinkChange} />
				Remember this link
			</label>
		</section>

		<!-- ==================================================================== -->
		<!-- WHEEL CONTAINER -->
		<!-- ==================================================================== -->
		<section class="wheel-section" style="background-color: {wheelBackground};">
			<!-- Canvas-based Wheel -->
			<!-- Features:
				- Slow idle rotation ✓
				- Click to spin (fast -> decelerate -> stop) - TODO
				- Multiple ticker marks for multi-spin (comedic effect) - TODO
				- Virtual rendering for thousands of items - TODO
				- Apply wheel colors from theme settings ✓
			-->
			<!-- Ticker at top -->
			<div class="ticker-container" style="border-color: {primaryColor}; background-color: {primaryColor};">
				<div class="ticker-arrow" style="border-bottom-color: {primaryColor};"></div>
				<div class="ticker-text" style="color: {secondaryColor};">▼</div>
			</div>

			<canvas
				bind:this={wheelCanvas}
				width="500"
				height="500"
				class="wheel-canvas"
				onclick={() => spin(1)}
			></canvas>
		</section>

		<!-- ==================================================================== -->
		<!-- SPIN CONTROLS -->
		<!-- ==================================================================== -->
		<section class="spin-controls" style="border-color: {primaryColor};">
			<div class="multi-spin-header">
				<h3 style="color: {primaryColor};">Multi Spin</h3>
				<div class="multi-spin-buttons">
					<button style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => spin(2)} disabled={isSpinning}>2</button>
					<button style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => spin(3)} disabled={isSpinning}>3</button>
					<button style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => spin(6)} disabled={isSpinning}>6</button>
					<input
						type="number"
						bind:value={customSpinCount}
						placeholder="_"
						min="1"
						style="border-color: {primaryColor}; color: {primaryColor};"
						class="custom-spin-input"
					/>
					<button
						style="border-color: {primaryColor}; color: {primaryColor};"
						onclick={() => {
							const count = parseInt(customSpinCount);
							if (count > 0) spin(count);
						}}
						disabled={isSpinning || !customSpinCount || parseInt(customSpinCount) <= 0}
						class="custom-spin-btn"
					>Go</button>
				</div>
			</div>
		</section>
	</main>

	<!-- ======================================================================== -->
	<!-- HISTORY SIDEBAR -->
	<!-- ======================================================================== -->
	<aside class="history-sidebar" style="border-left-color: {primaryColor};">
		<h2 style="color: {primaryColor};">Spin History</h2>

		<!-- TODO: History component -->
		<!-- Features:
			- Display recent spins (from localStorage)
			- Click to open Letterboxd search
			- Persist across sessions
		-->
		<!-- <HistorySidebar
			history={$spinHistory}
			onMovieClick={openLetterboxd}
		/> -->
	</aside>

	<footer style="background-color: {secondaryColor}; border-top-color: {primaryColor};">
		<p class="footer-main" style="color: {primaryColor};">made for Dawn</p>
		<p class="footer-small" style="color: {primaryColor};">alicevoid - make sure to be nice to yourself &lt;3</p>
	</footer>
</div>

<!-- ============================================================================ -->
<!-- THEME SETTINGS MODAL -->
<!-- ============================================================================ -->

{#if showThemeSettings}
	<div class="theme-settings-overlay" class:closing={isClosingThemeSettings} onclick={toggleThemeSettings}></div>
	<div class="theme-settings-panel" class:closing={isClosingThemeSettings} style="background-color: {secondaryColor};">
		<h2 style="color: {primaryColor}; border-bottom-color: {primaryColor};">Theme Settings</h2>

		<!-- Action Buttons at Top -->
		<div class="settings-actions-top" style="border-bottom-color: {primaryColor};">
			<button class="save-button" style="background-color: {primaryColor}; border-color: {primaryColor}; color: {secondaryColor};" onclick={() => {
				saveTheme();
				toggleThemeSettings();
			}}>Save Changes</button>
			<button class="cancel-button" style="border-color: {primaryColor}; color: {primaryColor};" onclick={toggleThemeSettings}>Cancel</button>
		</div>

		<!-- Import/Export Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Import/Export Theme</h3>
			<div class="button-group">
				<button style="border-color: {primaryColor}; color: {primaryColor};">Import Theme JSON</button>
				<button style="border-color: {primaryColor}; color: {primaryColor};">Export Theme JSON</button>
			</div>
		</section>

		<!-- Site Theme Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Site Theme</h3>

			<div class="color-setting">
				<label>Primary Color</label>
				<input type="color" bind:value={primaryColor} style="border-color: {primaryColor};" />
			</div>

			<div class="color-setting">
				<label>Secondary Color</label>
				<input type="color" bind:value={secondaryColor} style="border-color: {primaryColor};" />
			</div>

			<div class="color-setting">
				<label>Wheel Background</label>
				<input type="color" bind:value={wheelBackground} style="border-color: {primaryColor};" />
			</div>

			<div class="font-setting">
				<label>Font</label>
				<select bind:value={selectedFont} style="border-color: {primaryColor};">
					<option>Arial</option>
					<option>Times New Roman</option>
					<option>Georgia</option>
					<option>Courier New</option>
					<option>Verdana</option>
					<option>Comic Sans MS</option>
					<option>Impact</option>
					<option>Trebuchet MS</option>
				</select>
			</div>
		</section>

		<!-- Wheel Colors Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Wheel Slice Colors</h3>

			<div class="wheel-colors-list">
				{#each wheelColors as color, i}
					<div class="wheel-color-item">
						<input type="color" bind:value={wheelColors[i]} style="border-color: {primaryColor};" />
						<button class="remove-color-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
							if (wheelColors.length > 1) {
								wheelColors = wheelColors.filter((_, index) => index !== i);
							}
						}}>Remove</button>
					</div>
				{/each}
			</div>

			<button class="add-color-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
				wheelColors = [...wheelColors, '#000000'];
			}}>Add Color</button>
		</section>

		<!-- Images Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Images</h3>

			<div class="upload-setting-compact">
				<label>Site Wallpaper</label>
				<div class="upload-row">
					<input type="file" accept="image/*" style="border-color: {primaryColor};" />
					<select bind:value={backgroundWallpaper} style="border-color: {primaryColor};">
						<option value="">None</option>
						<option value="arctic_bg.jpg">Arctic</option>
					</select>
				</div>
			</div>

			<div class="upload-setting-compact">
				<label>Wheel Center Image</label>
				<div class="upload-row">
					<input type="file" accept="image/*" style="border-color: {primaryColor};" />
					<select bind:value={wheelCenterImage} style="border-color: {primaryColor};">
						<option value="">None</option>
						<option value="penguin.jpg">Penguin</option>
					</select>
				</div>
			</div>
		</section>

		<!-- Sounds Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Sound Effects</h3>

			<div class="upload-setting-compact">
				<label>Spin SFX</label>
				<div class="upload-row">
					<input type="file" accept="audio/*" style="border-color: {primaryColor};" />
					<select style="border-color: {primaryColor};">
						<option>None</option>
						<option>Default 1</option>
						<option>Default 2</option>
					</select>
				</div>
			</div>

			<div class="upload-setting-compact">
				<label>Result SFX</label>
				<div class="upload-row">
					<input type="file" accept="audio/*" style="border-color: {primaryColor};" />
					<select style="border-color: {primaryColor};">
						<option>None</option>
						<option>Default 1</option>
						<option>Default 2</option>
					</select>
				</div>
			</div>
		</section>
	</div>
{/if}

<!-- ============================================================================ -->
<!-- RESULTS MODAL -->
<!-- ============================================================================ -->

{#if showResultsModal}
	<div class="modal-overlay" onclick={() => showResultsModal = false}></div>
	<div class="results-modal" style="border-color: {primaryColor};">
		<button class="modal-close-btn" style="color: {primaryColor};" onclick={() => showResultsModal = false}>✕</button>

		<h2 style="color: {primaryColor};">Results!</h2>

		<div class="results-list">
			{#each currentResults as result, i}
				<div class="result-item" style="border-color: {wheelColors[i % wheelColors.length]}; background-color: {wheelColors[i % wheelColors.length]}20;">
					<div class="result-content">
						<h3 style="color: {primaryColor};">{result}</h3>
						<button class="letterboxd-btn" style="background-color: {primaryColor}; color: {secondaryColor};">
							Open in Letterboxd
						</button>
					</div>
				</div>
			{/each}
		</div>
	</div>
{/if}

<!-- ============================================================================ -->
<!-- THEME SETTINGS MODAL -->
<!-- ============================================================================ -->

<!-- TODO: Theme settings modal component -->
<!-- Features:
	- Background wallpaper upload
	- Wheel color customization
	- Sound file selection (with default naming)
	- Save to localStorage
-->
<!-- {#if showThemeSettings}
	<ThemeSettings
		currentTheme={$theme}
		onSave={saveThemeSettings}
		onClose={toggleThemeSettings}
	/>
{/if} -->

<!-- ============================================================================ -->
<!-- STYLES -->
<!-- ============================================================================ -->

<style>
	/* TODO: Add Tailwind CSS or custom styles */
	/* For now, basic layout structure */

	:global(*) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:global(body) {
		font-family: Arial, sans-serif;
		overflow: hidden;
	}

	.app-container {
		display: grid;
		grid-template-columns: 1fr 300px;
		grid-template-rows: auto 1fr auto;
		height: 100vh;
		overflow: hidden;
	}

	header {
		grid-column: 1 / -1;
		text-align: center;
		padding: 1rem;
		border-bottom: 2px solid #ccc;
		position: relative;
	}

	.theme-button {
		position: absolute;
		top: 1rem;
		right: 1rem;
		padding: 0.5rem 1rem;
		font-size: 1rem;
		cursor: pointer;
		border: 2px solid;
		border-radius: 8px;
		background: transparent;
		transition: all 0.2s;
	}

	.theme-button:hover {
		background: rgba(255, 255, 255, 0.2);
	}

	main {
		display: flex;
		flex-direction: column;
		gap: 2rem;
		padding: 1rem;
		overflow-y: auto;
	}

	.google-docs-input {
		padding: 1rem;
		border: 1px solid #ccc;
		border-radius: 8px;
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.link-input-container {
		display: flex;
		gap: 0.5rem;
		align-items: center;
	}

	.refresh-button {
		padding: 0.75rem 1rem;
		font-size: 1rem;
		cursor: pointer;
		border: 2px solid;
		border-radius: 8px;
		background: #fff;
		transition: all 0.2s;
		flex-shrink: 0;
		height: 100%;
	}

	.refresh-button:hover:not(:disabled) {
		background: rgba(114, 47, 55, 0.1);
	}

	.refresh-button:disabled {
		opacity: 0.3;
		cursor: not-allowed;
	}

	.link-input {
		flex: 1;
		padding: 0.75rem;
		font-size: 1rem;
		border: 2px solid;
		border-radius: 8px;
		font-family: Arial, sans-serif;
	}

	.link-input:disabled {
		background: #f0f0f0;
		cursor: not-allowed;
		color: #666;
	}

	.link-input:focus {
		outline: none;
		box-shadow: 0 0 0 3px rgba(114, 47, 55, 0.2);
	}

	.checkbox-label {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 1rem;
		cursor: pointer;
	}

	.checkbox-label input[type="checkbox"] {
		width: 1.2rem;
		height: 1.2rem;
		cursor: pointer;
	}

	.wheel-section {
		position: relative;
		display: flex;
		justify-content: center;
		align-items: center;
		min-height: 500px;
		padding: 2rem;
		border-radius: 12px;
		background-color: rgba(224, 242, 255, 0.3) !important; /* Translucent light blue */
		backdrop-filter: blur(10px);
	}

	.wheel-canvas {
		display: block;
		cursor: pointer;
		filter: drop-shadow(0 4px 8px rgba(0, 0, 0, 0.2));
	}

	.spin-controls {
		padding: 1rem;
		border: 1px solid;
		border-radius: 8px;
		background-color: rgba(224, 242, 255, 0.3); /* Translucent light blue */
		backdrop-filter: blur(10px);
	}

	.multi-spin-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 1rem;
	}

	.multi-spin-header h3 {
		margin: 0;
	}

	.multi-spin-buttons {
		display: flex;
		gap: 0.5rem;
		align-items: center;
	}

	.multi-spin-buttons button {
		padding: 0.5rem 1rem;
		font-size: 1rem;
		cursor: pointer;
		border: 2px solid;
		border-radius: 8px;
		background: #fff;
		transition: all 0.2s;
		min-width: 50px;
	}

	.multi-spin-buttons button:hover:not(:disabled) {
		background: rgba(114, 47, 55, 0.1);
	}

	.multi-spin-buttons button:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}

	.custom-spin-input {
		width: 60px;
		padding: 0.5rem;
		font-size: 1rem;
		border: 2px solid;
		border-radius: 8px;
		text-align: center;
		background: #fff;
	}

	.custom-spin-input:focus {
		outline: none;
		box-shadow: 0 0 0 2px rgba(30, 144, 255, 0.2);
	}

	/* Remove spinner arrows from number input */
	.custom-spin-input::-webkit-inner-spin-button,
	.custom-spin-input::-webkit-outer-spin-button {
		-webkit-appearance: none;
		margin: 0;
	}

	.custom-spin-input[type=number] {
		-moz-appearance: textfield;
		appearance: textfield;
	}

	.ticker-container {
		position: absolute;
		top: -20px;
		left: 50%;
		transform: translateX(-50%);
		z-index: 10;
		padding: 0.5rem 1rem;
		border-radius: 8px;
		border: 2px solid;
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
	}

	.ticker-text {
		font-size: 1.5rem;
		font-weight: bold;
		text-align: center;
	}

	.history-sidebar {
		padding: 1rem;
		border-left: 2px solid #ccc;
		overflow-y: auto;
		background-color: rgba(224, 242, 255, 0.3); /* Translucent light blue */
		backdrop-filter: blur(10px);
	}

	.placeholder-text {
		color: #999;
		font-style: italic;
	}

	h1 {
		margin: 0;
		font-size: 2.5rem;
	}

	h2 {
		margin-top: 0;
		font-size: 1.5rem;
	}

	h3 {
		margin-bottom: 0.5rem;
	}

	footer {
		grid-column: 1 / -1;
		text-align: center;
		padding: 1.5rem 1rem;
		border-top: 2px solid #ccc;
		background: #f9f9f9;
	}

	.footer-main {
		margin: 0;
		font-size: 1.2rem;
		font-weight: bold;
	}

	.footer-small {
		margin: 0.5rem 0 0 0;
		font-size: 0.9rem;
		color: #666;
	}

	/* Theme Settings Panel */
	.theme-settings-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: rgba(0, 0, 0, 0.5);
		z-index: 999;
		animation: fadeIn 0.2s ease-out;
	}

	.theme-settings-overlay.closing {
		animation: fadeOut 0.3s ease-out forwards;
	}

	@keyframes fadeIn {
		from {
			opacity: 0;
		}
		to {
			opacity: 1;
		}
	}

	@keyframes fadeOut {
		from {
			opacity: 1;
		}
		to {
			opacity: 0;
		}
	}

	.theme-settings-panel {
		position: fixed;
		top: 0;
		right: 0;
		width: 500px;
		height: 100vh;
		background: #fff;
		box-shadow: -2px 0 10px rgba(0, 0, 0, 0.2);
		z-index: 1000;
		padding: 1.5rem;
		padding-bottom: 3rem;
		overflow-y: auto;
		animation: slideIn 0.3s ease-out;
	}

	.theme-settings-panel.closing {
		animation: slideOut 0.3s ease-out forwards;
	}

	@keyframes slideIn {
		from {
			transform: translateX(100%);
		}
		to {
			transform: translateX(0);
		}
	}

	@keyframes slideOut {
		from {
			transform: translateX(0);
		}
		to {
			transform: translateX(100%);
		}
	}

	.theme-settings-panel h2 {
		margin-top: 0;
		margin-bottom: 1rem;
		font-size: 1.5rem;
		border-bottom: 2px solid #333;
		padding-bottom: 0.4rem;
	}

	.settings-section {
		margin-bottom: 1.5rem;
		padding-bottom: 1rem;
		border-bottom: 1px solid #ddd;
	}

	.settings-section h3 {
		margin-top: 0;
		margin-bottom: 0.75rem;
		font-size: 1.2rem;
		color: #555;
	}

	.button-group {
		display: flex;
		gap: 0.5rem;
	}

	.button-group button {
		padding: 0.5rem 1rem;
		border: 2px solid;
		border-radius: 8px;
		background: #fff;
		cursor: pointer;
		transition: all 0.2s;
	}

	.button-group button:hover {
		background: rgba(114, 47, 55, 0.1);
	}

	.color-setting {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.5rem;
	}

	.color-setting label {
		font-size: 1rem;
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.color-setting input[type="color"] {
		width: 60px;
		height: 40px;
		border: 2px solid;
		border-radius: 4px;
		cursor: pointer;
	}

	.color-setting-full {
		margin-bottom: 1.5rem;
	}

	.color-setting-full label {
		display: block;
		font-weight: bold;
		margin-bottom: 0.5rem;
		font-size: 1rem;
	}


	.font-setting {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.5rem;
	}

	.font-setting label {
		font-size: 1rem;
	}

	.font-setting select {
		width: 200px;
		padding: 0.5rem;
		border: 2px solid;
		border-radius: 6px;
		font-family: Arial, sans-serif;
		cursor: pointer;
	}

	.wheel-colors-list {
		display: flex;
		flex-direction: column;
		gap: 0.4rem;
		margin-bottom: 0.75rem;
	}

	.wheel-color-item {
		display: flex;
		gap: 0.5rem;
		align-items: center;
	}

	.wheel-color-item input[type="color"] {
		width: 80px;
		height: 40px;
		border: 2px solid;
		border-radius: 4px;
		cursor: pointer;
		flex-shrink: 0;
	}

	.wheel-color-item-full {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		margin-bottom: 1rem;
		padding: 1rem;
		border: 1px solid #ddd;
		border-radius: 8px;
		background: #f9f9f9;
	}

	.wheel-color-item-full button {
		align-self: flex-start;
	}


	.remove-color-btn {
		padding: 0.5rem 1rem;
		border: 2px solid;
		border-radius: 6px;
		background: #fff;
		cursor: pointer;
		transition: all 0.2s;
		font-size: 0.9rem;
	}

	.remove-color-btn:hover {
		background: rgba(114, 47, 55, 0.1);
	}

	.add-color-btn {
		width: 100%;
		padding: 0.75rem;
		border: 2px dashed;
		border-radius: 6px;
		background: #fff;
		cursor: pointer;
		transition: all 0.2s;
		font-size: 1rem;
	}

	.add-color-btn:hover {
		background: rgba(114, 47, 55, 0.05);
		border-style: solid;
	}

	.upload-setting {
		margin-bottom: 1.5rem;
	}

	.upload-setting label {
		display: block;
		font-weight: bold;
		margin-bottom: 0.5rem;
	}

	.upload-setting input[type="file"],
	.upload-setting select {
		width: 100%;
		padding: 0.5rem;
		margin-bottom: 0.5rem;
		border: 2px solid;
		border-radius: 8px;
		font-family: Arial, sans-serif;
	}

	.upload-setting-compact {
		margin-bottom: 0.75rem;
	}

	.upload-setting-compact label {
		display: block;
		font-weight: bold;
		margin-bottom: 0.25rem;
		font-size: 0.9rem;
	}

	.upload-row {
		display: flex;
		gap: 0.5rem;
	}

	.upload-row input[type="file"] {
		flex: 1;
		padding: 0.4rem;
		border: 2px solid;
		border-radius: 6px;
		font-family: Arial, sans-serif;
		font-size: 0.85rem;
	}

	.upload-row select {
		width: 120px;
		padding: 0.4rem;
		border: 2px solid;
		border-radius: 6px;
		font-family: Arial, sans-serif;
		font-size: 0.85rem;
	}

	.settings-actions {
		display: flex;
		gap: 1rem;
		margin-top: 1.5rem;
		padding-top: 1rem;
		border-top: 2px solid #333;
	}

	.settings-actions-top {
		display: flex;
		gap: 1rem;
		margin-bottom: 1.5rem;
		padding-bottom: 1rem;
		border-bottom: 2px solid;
	}

	.save-button,
	.cancel-button {
		flex: 1;
		padding: 0.75rem 1.5rem;
		font-size: 1rem;
		border: 2px solid;
		border-radius: 8px;
		cursor: pointer;
		transition: all 0.2s;
	}

	.save-button:hover {
		opacity: 0.9;
	}

	.cancel-button {
		background: #fff;
	}

	.cancel-button:hover {
		background: rgba(114, 47, 55, 0.1);
	}

	/* ========================================================================== */
	/* RESULTS MODAL */
	/* ========================================================================== */

	.modal-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: rgba(0, 0, 0, 0.6);
		z-index: 1000;
		animation: fadeIn 0.3s ease-out;
	}

	.results-modal {
		position: fixed;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
		background: white;
		border-radius: 16px;
		border: 3px solid;
		padding: 2rem;
		max-width: 600px;
		width: 90%;
		max-height: 80vh;
		overflow-y: auto;
		z-index: 1001;
		animation: modalSlideIn 0.3s ease-out;
		box-shadow: 0 10px 40px rgba(0, 0, 0, 0.3);
	}

	.modal-close-btn {
		position: absolute;
		top: 1rem;
		right: 1rem;
		background: none;
		border: none;
		font-size: 2rem;
		cursor: pointer;
		padding: 0.5rem;
		line-height: 1;
		transition: transform 0.2s;
	}

	.modal-close-btn:hover {
		transform: scale(1.2);
	}

	.results-modal h2 {
		text-align: center;
		margin-bottom: 2rem;
		font-size: 2rem;
	}

	.results-list {
		display: flex;
		flex-direction: column;
		gap: 1.5rem;
		margin-bottom: 2rem;
	}

	.result-item {
		border: 2px solid;
		border-radius: 12px;
		padding: 1.5rem;
		animation: bounceIn 0.5s ease-out;
	}

	.result-content {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 1rem;
	}

	.result-item h3 {
		font-size: 1.5rem;
		margin: 0;
		flex: 1;
		word-wrap: break-word;
	}

	.letterboxd-btn {
		padding: 0.75rem 1.5rem;
		border-radius: 8px;
		font-size: 1rem;
		cursor: pointer;
		transition: all 0.2s;
		border: none;
		white-space: nowrap;
		flex-shrink: 0;
	}

	.letterboxd-btn:hover {
		opacity: 0.9;
		transform: translateY(-2px);
	}

	@keyframes modalSlideIn {
		from {
			opacity: 0;
			transform: translate(-50%, -60%);
		}
		to {
			opacity: 1;
			transform: translate(-50%, -50%);
		}
	}

	@keyframes bounceIn {
		0% {
			opacity: 0;
			transform: scale(0.8);
		}
		50% {
			transform: scale(1.05);
		}
		100% {
			opacity: 1;
			transform: scale(1);
		}
	}
</style>
