<script lang="ts">
	// ============================================================================
	// SLIPPYMUDWHEEL - Movie Wheel Spinner
	// ============================================================================

	import { onMount, onDestroy } from 'svelte';
	import arcticBg from '$lib/arctic_bg.jpg';
	import creepyBg from '$lib/creepy.jpg';
	import penguinImg from '$lib/penguin.jpg';
	import handsImg from '$lib/hands.jpg';
	import gasCanImg from '$lib/gas_can.png';

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
	let showCustomSettings = $state(false);

	// Theme preset state
	let selectedThemePreset = $state('penguin'); // 'penguin' or 'haunted'

	// Theme state - Penguin theme (defaults)
	let primaryColor = $state('#1E90FF'); // Dodger blue
	let secondaryColor = $state('#FFFFFF'); // White
	let wheelBackground = $state('#E0F2FF'); // Light ice blue
	let selectedFont = $state('Arial');
	let invertBW = $state(false);
	let wheelColors = $state(['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6']);
	let backgroundWallpaper = $state('arctic_bg.jpg'); // Arctic background
	let wheelCenterImage = $state('penguin.jpg'); // Penguin center image

	// Movie data - default to 1-100 for testing (will be replaced by Google Docs data later)
	let movies = $state(Array.from({ length: 100 }, (_, i) => (i + 1).toString()));

	// Wheel state
	let wheelCanvas: HTMLCanvasElement;
	let rotation = $state(0);
	let isSpinning = $state(false);

	// Lantern cursor effect for Haunted Mode
	let cursorX = $state(0);
	let cursorY = $state(0);

	// Fuel system for Haunted Mode
	let fuelLevel = $state(100); // 0-100, starts at full
	let fuelDrainInterval: number;
	let gasCanX = $state(0);
	let gasCanY = $state(0);

	function handleMouseMove(e: MouseEvent) {
		cursorX = e.clientX;
		cursorY = e.clientY;
	}

	function startFuelDrain() {
		// Drain fuel over 60 seconds (100% / 60 seconds = ~1.67% per second)
		// Update every 100ms for smooth animation
		fuelDrainInterval = setInterval(() => {
			if (fuelLevel > 0) {
				fuelLevel = Math.max(0, fuelLevel - (100 / 600)); // 600 intervals in 60 seconds
			}
		}, 100);
	}

	function stopFuelDrain() {
		if (fuelDrainInterval) {
			clearInterval(fuelDrainInterval);
		}
	}

	function refuelLantern() {
		// Gradually refill to 100%
		const refillInterval = setInterval(() => {
			if (fuelLevel < 100) {
				fuelLevel = Math.min(100, fuelLevel + 5); // Refill faster than drain
			} else {
				clearInterval(refillInterval);
				repositionGasCan(); // Move gas can to new location after refuel
			}
		}, 50);
	}

	function repositionGasCan() {
		// Position gas can randomly on the page, avoiding edges
		const margin = 100;
		gasCanX = margin + Math.random() * (typeof window !== 'undefined' ? window.innerWidth - margin * 2 : 800);
		gasCanY = margin + Math.random() * (typeof window !== 'undefined' ? window.innerHeight - margin * 2 : 600);
	}

	// Preload center images
	let penguinImage: HTMLImageElement | null = null;
	let handsImage: HTMLImageElement | null = null;
	if (typeof window !== 'undefined') {
		penguinImage = new Image();
		penguinImage.src = penguinImg;
		handsImage = new Image();
		handsImage.src = handsImg;
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
	// SPIN HISTORY MANAGEMENT
	// ============================================================================

	function saveHistoryToLocalStorage() {
		localStorage.setItem('slippymud_spin_history', JSON.stringify(spinHistory));
	}

	function loadHistoryFromLocalStorage() {
		const saved = localStorage.getItem('slippymud_spin_history');
		if (saved) {
			try {
				spinHistory = JSON.parse(saved);
			} catch (e) {
				console.error('Failed to load spin history:', e);
				spinHistory = [];
			}
		}
	}

	function addToHistory(movie: string) {
		const historyItem: HistoryItem = {
			id: Date.now().toString() + Math.random().toString(36).substring(2, 9),
			movie,
			timestamp: Date.now()
		};
		// Add to beginning of array (most recent first)
		spinHistory = [historyItem, ...spinHistory];
		saveHistoryToLocalStorage();
	}

	function openLetterboxd(movieName: string) {
		const searchUrl = `https://letterboxd.com/search/${encodeURIComponent(movieName)}/`;
		window.open(searchUrl, '_blank');
	}

	function clearHistory() {
		if (confirm('Are you sure you want to clear all spin history?')) {
			spinHistory = [];
			saveHistoryToLocalStorage();
		}
	}

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

	async function handleRefresh() {
		if (!googleDocsLink.trim()) {
			alert('Please enter a Google Docs link first');
			return;
		}

		try {
			// Convert sharing link to export format
			// Google Docs sharing link format: https://docs.google.com/document/d/{DOC_ID}/edit...
			// Export as plain text format: https://docs.google.com/document/d/{DOC_ID}/export?format=txt

			let exportUrl = googleDocsLink;

			// Extract document ID and convert to export URL
			const docIdMatch = googleDocsLink.match(/\/d\/([a-zA-Z0-9-_]+)/);
			if (docIdMatch) {
				const docId = docIdMatch[1];
				exportUrl = `https://docs.google.com/document/d/${docId}/export?format=txt`;
			}

			// Fetch the document
			const response = await fetch(exportUrl);

			if (!response.ok) {
				throw new Error(`Failed to fetch document: ${response.statusText}`);
			}

			const text = await response.text();

			// Split by lines and filter out empty lines
			const lines = text
				.split('\n')
				.map(line => line.trim())
				.filter(line => line.length > 0);

			if (lines.length === 0) {
				alert('No content found in the document. Make sure the document has text and is publicly accessible.');
				return;
			}

			// Update movies array with the lines from the document
			movies = lines;

			alert(`Successfully loaded ${lines.length} items from Google Docs!`);

		} catch (error) {
			console.error('Error fetching Google Docs:', error);
			alert('Failed to load document. Make sure:\n1. The link is correct\n2. The document is set to "Anyone with the link can view"\n3. You have internet connection');
		}
	}

	// ============================================================================
	// THEME SETTINGS MANAGEMENT
	// ============================================================================

	function applyThemePreset(preset: 'penguin' | 'haunted') {
		selectedThemePreset = preset;

		if (preset === 'penguin') {
			// Penguin Theme (Arctic)
			primaryColor = '#1E90FF'; // Dodger blue
			secondaryColor = '#FFFFFF'; // White
			wheelBackground = '#E0F2FF'; // Light ice blue
			selectedFont = 'Arial';
			wheelColors = ['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6'];
			backgroundWallpaper = 'arctic_bg.jpg';
			wheelCenterImage = 'penguin.jpg';
		} else if (preset === 'haunted') {
			// Haunted Mode (Black & Red)
			primaryColor = '#8B0000'; // Dark red
			secondaryColor = '#000000'; // Black
			wheelBackground = '#1a0000'; // Very dark red
			selectedFont = 'Georgia';
			wheelColors = ['#8B0000', '#FF0000', '#DC143C', '#B22222', '#A52A2A'];
			backgroundWallpaper = 'creepy.jpg';
			wheelCenterImage = 'hands.jpg';
		}
	}

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
		const shouldShowText = movies.length <= 100;
		const shouldShowBorders = movies.length <= 100;

		for (let i = 0; i < movies.length; i++) {
			const startAngle = i * sliceAngle;
			const endAngle = startAngle + sliceAngle;

			// Draw slice
			ctx.beginPath();
			ctx.moveTo(centerX, centerY);
			ctx.arc(centerX, centerY, radius, startAngle, endAngle);
			ctx.closePath();

			// Fill with theme colors
			ctx.fillStyle = wheelColors[i % wheelColors.length];
			ctx.fill();

			// CONDITIONAL BORDER: Only show for datasets <= 100 items
			if (shouldShowBorders) {
				ctx.strokeStyle = '#ffffff';
				ctx.lineWidth = 2;
				ctx.stroke();
			}

			// CONDITIONAL TEXT: Only show for datasets <= 100 items (performance optimization)
			if (shouldShowText) {
				ctx.save();
				ctx.translate(centerX, centerY);
				ctx.rotate(startAngle + sliceAngle / 2);
				ctx.textAlign = 'right';
				ctx.fillStyle = '#000000';
				ctx.font = 'bold 16px Arial';
				ctx.fillText(movies[i], radius - 20, 10);
				ctx.restore();
			}
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

		// Draw center image if selected
		if (wheelCenterImage === 'penguin.jpg' && penguinImage && penguinImage.complete) {
			// Draw image centered in the circle, clipped to circle shape
			const imgRadius = centerRadius - 2;
			ctx.save();
			ctx.beginPath();
			ctx.arc(centerX, centerY, imgRadius, 0, 2 * Math.PI);
			ctx.clip();
			ctx.drawImage(penguinImage, centerX - imgRadius, centerY - imgRadius, imgRadius * 2, imgRadius * 2);
			ctx.restore();
		} else if (wheelCenterImage === 'hands.jpg' && handsImage && handsImage.complete) {
			// Draw hands image centered in the circle, clipped to circle shape
			const imgRadius = centerRadius - 2;
			ctx.save();
			ctx.beginPath();
			ctx.arc(centerX, centerY, imgRadius, 0, 2 * Math.PI);
			ctx.clip();
			ctx.drawImage(handsImage, centerX - imgRadius, centerY - imgRadius, imgRadius * 2, imgRadius * 2);
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

			// Add each result to history
			results.forEach(movie => addToHistory(movie));

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
			selectedThemePreset,
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
				selectedThemePreset = theme.selectedThemePreset || selectedThemePreset;
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

		// Load spin history
		loadHistoryFromLocalStorage();

		// Start idle rotation animation
		animateIdleRotation();

		// Load saved Google Docs link if exists
		const savedLink = localStorage.getItem('slippymud_google_docs_link');
		if (savedLink) {
			googleDocsLink = savedLink;
			rememberLink = true;
			isLinkLocked = true;
		}

		// Initialize gas can position
		repositionGasCan();

		// Start fuel drain if in Haunted Mode
		if (selectedThemePreset === 'haunted') {
			startFuelDrain();
		}
	});

	onDestroy(() => {
		// Clean up animation frame
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
		}
		// Clean up fuel drain interval
		stopFuelDrain();
	});

	// Watch for theme changes to start/stop fuel drain
	$effect(() => {
		if (selectedThemePreset === 'haunted') {
			startFuelDrain();
		} else {
			stopFuelDrain();
			fuelLevel = 100; // Reset fuel when leaving Haunted Mode
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

<div class="app-container" style="font-family: {selectedFont}; background-color: {wheelBackground}; {backgroundWallpaper === 'arctic_bg.jpg' ? `background-image: url('${arcticBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : backgroundWallpaper === 'creepy.jpg' ? `background-image: url('${creepyBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : ''}" onmousemove={handleMouseMove}>

	<!-- Lantern cursor effect for Haunted Mode -->
	{#if selectedThemePreset === 'haunted'}
	<div class="lantern-overlay" style="--cursor-x: {cursorX}px; --cursor-y: {cursorY}px; --fuel-level: {fuelLevel};"></div>

	<!-- Hidden Gas Can -->
	<img
		src={gasCanImg}
		alt="Gas Can"
		class="gas-can"
		style="left: {gasCanX}px; top: {gasCanY}px;"
		onmouseenter={refuelLantern}
	/>
	{/if}

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
		<div class="history-header">
			<h2 style="color: {primaryColor};">Spin History</h2>
			{#if spinHistory.length > 0}
				<button class="clear-history-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={clearHistory}>
					Clear
				</button>
			{/if}
		</div>

		{#if spinHistory.length === 0}
			<p class="placeholder-text">No spins yet! Spin the wheel to start building your history.</p>
		{:else}
			<div class="history-list">
				{#each spinHistory as item (item.id)}
					<div class="history-item" style="border-color: {primaryColor}; background-color: {primaryColor}10;">
						<div class="history-item-content">
							<p class="history-movie-name" style="color: {primaryColor};">{item.movie}</p>
							<button
								class="history-letterboxd-btn"
								style="background-color: {primaryColor}; color: {secondaryColor};"
								onclick={() => openLetterboxd(item.movie)}
							>
								Letterboxd
							</button>
						</div>
						<p class="history-timestamp">{new Date(item.timestamp).toLocaleString()}</p>
					</div>
				{/each}
			</div>
		{/if}
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

		<!-- Theme Presets Section -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<h3 style="color: {primaryColor};">Theme Presets</h3>
			<div class="theme-preset-buttons">
				<button
					class="preset-button"
					class:active={selectedThemePreset === 'penguin'}
					style="border-color: {primaryColor}; {selectedThemePreset === 'penguin' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
					onclick={() => applyThemePreset('penguin')}
				>
					🐧 Penguin (Default)
				</button>
				<button
					class="preset-button"
					class:active={selectedThemePreset === 'haunted'}
					style="border-color: {primaryColor}; {selectedThemePreset === 'haunted' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
					onclick={() => applyThemePreset('haunted')}
				>
					👻 HAUNTED MODE
				</button>
			</div>
		</section>

		<!-- Custom Settings Toggle -->
		<section class="settings-section" style="border-bottom-color: {primaryColor};">
			<label class="checkbox-label">
				<input type="checkbox" bind:checked={showCustomSettings} />
				<span style="color: {primaryColor};">Show Custom Theme Settings</span>
			</label>
		</section>

		{#if showCustomSettings}
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
						<option value="creepy.jpg">Creepy</option>
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
						<option value="hands.jpg">Hands</option>
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
		{/if}
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
						<button class="letterboxd-btn" style="background-color: {primaryColor}; color: {secondaryColor};" onclick={() => openLetterboxd(result)}>
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

	.test-mode-toggle {
		position: absolute;
		top: 1rem;
		right: 1rem;
		z-index: 5;
		background: rgba(255, 255, 255, 0.9);
		padding: 0.5rem 0.75rem;
		border-radius: 8px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
	}

	.test-mode-label {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 0.9rem;
		cursor: pointer;
		margin: 0;
	}

	.test-mode-label input[type="checkbox"] {
		width: 1.1rem;
		height: 1.1rem;
		cursor: pointer;
	}

	.test-mode-label span {
		font-weight: 500;
		user-select: none;
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

	.history-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1rem;
	}

	.history-header h2 {
		margin: 0;
	}

	.clear-history-btn {
		padding: 0.5rem 1rem;
		font-size: 0.9rem;
		cursor: pointer;
		border: 2px solid;
		border-radius: 8px;
		background: #fff;
		transition: all 0.2s;
	}

	.clear-history-btn:hover {
		background: rgba(30, 144, 255, 0.1);
	}

	.history-list {
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
	}

	.history-item {
		padding: 0.75rem;
		border: 1px solid;
		border-radius: 8px;
		transition: all 0.2s;
	}

	.history-item:hover {
		transform: translateX(-4px);
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
	}

	.history-item-content {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 0.5rem;
		margin-bottom: 0.5rem;
	}

	.history-movie-name {
		margin: 0;
		font-weight: bold;
		font-size: 1rem;
		flex: 1;
		word-wrap: break-word;
	}

	.history-letterboxd-btn {
		padding: 0.4rem 0.75rem;
		font-size: 0.85rem;
		cursor: pointer;
		border: none;
		border-radius: 6px;
		transition: all 0.2s;
		white-space: nowrap;
		flex-shrink: 0;
	}

	.history-letterboxd-btn:hover {
		opacity: 0.9;
		transform: translateY(-2px);
	}

	.history-timestamp {
		margin: 0;
		font-size: 0.75rem;
		color: #666;
		font-style: italic;
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

	.theme-preset-buttons {
		display: flex;
		gap: 1rem;
		flex-direction: column;
	}

	.preset-button {
		padding: 1rem 1.5rem;
		font-size: 1.1rem;
		font-weight: bold;
		border: 3px solid;
		border-radius: 12px;
		background: #fff;
		cursor: pointer;
		transition: all 0.3s;
		text-align: left;
	}

	.preset-button:hover {
		transform: translateX(8px);
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
	}

	.preset-button.active {
		transform: translateX(8px);
		box-shadow: 0 6px 16px rgba(0, 0, 0, 0.3);
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

	/* ========================================================================== */
	/* LANTERN CURSOR EFFECT (HAUNTED MODE) */
	/* ========================================================================== */

	.lantern-overlay {
		position: fixed;
		top: 0;
		left: 0;
		width: 100vw;
		height: 100vh;
		pointer-events: none;
		z-index: 99999; /* Increased to cover theme settings modal */
		/* Light radius and brightness based on fuel level */
		/* At 100% fuel: bright warm light with 250px radius */
		/* At 0% fuel: dim cool blue light with 50px radius */
		--light-radius: calc(50px + (var(--fuel-level) * 2px)); /* More dramatic radius change */
		--light-brightness: calc(var(--fuel-level) / 100);

		/* Logarithmic color temperature transition (warm yellow/orange -> cool blue) */
		/* Only starts getting cool around 20% fuel remaining */
		/* Using exponential decay: more warm above 20%, rapidly cool below 20% */
		--coolness-factor: calc(
			max(0, 1 - pow(var(--fuel-level) / 20, 2))
		); /* 0 at 100%, barely changes until ~20%, then rapidly increases to 1 at 0% */

		--warm-red: calc(255 - (200 * var(--coolness-factor))); /* 255 -> 55 */
		--warm-orange: calc(200 - (150 * var(--coolness-factor))); /* 200 -> 50 */
		--cool-blue: calc(100 * var(--coolness-factor)); /* 0 -> 100 */

		/* Fuzzy flashlight effect with more gradient stops for softer edges */
		background: radial-gradient(
			circle var(--light-radius) at var(--cursor-x) var(--cursor-y),
			transparent 0%,
			transparent calc(var(--light-radius) * 0.2),
			rgba(
				var(--warm-red),
				var(--warm-orange),
				var(--cool-blue),
				calc(0.3 * (1 - var(--light-brightness) * 0.5))
			) calc(var(--light-radius) * 0.4),
			rgba(
				var(--warm-red),
				var(--warm-orange),
				var(--cool-blue),
				calc(0.5 * (1 - var(--light-brightness) * 0.5))
			) calc(var(--light-radius) * 0.6),
			rgba(
				var(--warm-red),
				var(--warm-orange),
				var(--cool-blue),
				calc(0.7 + (0.15 * (1 - var(--light-brightness))))
			) calc(var(--light-radius) * 0.8),
			rgba(
				var(--warm-red),
				var(--warm-orange),
				var(--cool-blue),
				calc(0.85 + (0.1 * (1 - var(--light-brightness))))
			) var(--light-radius),
			rgba(
				calc(var(--warm-red) * 0.8),
				calc(var(--warm-orange) * 0.8),
				calc(var(--cool-blue) * 1.2),
				calc(0.92 + (0.05 * (1 - var(--light-brightness))))
			) calc(var(--light-radius) * 1.3),
			rgba(
				calc(var(--warm-red) * 0.6),
				calc(var(--warm-orange) * 0.6),
				calc(var(--cool-blue) * 1.5),
				calc(0.96 + (0.04 * (1 - var(--light-brightness))))
			) 100%
		);
		transition: background 0.1s ease-out;
	}

	.gas-can {
		position: fixed;
		width: 100px; /* Normal size for debugging */
		height: auto;
		z-index: 10000; /* Above lantern overlay but still visible through it */
		cursor: pointer;
		transition: transform 0.2s, filter 0.2s;
		opacity: 0.8; /* More visible for debugging */
		filter: drop-shadow(0 0 10px rgba(255, 200, 0, 0.5));
		pointer-events: auto; /* Ensure it can be interacted with */
	}

	.gas-can:hover {
		transform: scale(1.2);
		opacity: 1;
		filter: drop-shadow(0 0 20px rgba(255, 200, 0, 0.8));
	}
</style>
