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
	import wheelClickSfx from '$lib/wheel_click.ogg';
	import rimClickSfx from '$lib/rim_click.ogg';
	import victorySfx from '$lib/victory.ogg';

	// Horror Mode Audio
	import refillSfx from '$lib/refill.ogg';
	import horrorBgSfx from '$lib/horror_bg.ogg';
	import fnafHallwaySfx from '$lib/fnaf2_hallway_sfx.ogg';
	import fnafJumpscareSfx from '$lib/fnaf2_puppet_js.ogg';
	import fnafJumpscareGif from '$lib/fnaf_jumpscare.gif';
	import horrorStartSfx from '$lib/horror_start.ogg';
	import horrorResultSfx from '$lib/horror_result.ogg';
	import woodWheelSfx from '$lib/wood_wheel.ogg';

	// Marionette scream SFX (random one plays before reset)
	import mar01Sfx from '$lib/Mar01.ogg';
	import mar02Sfx from '$lib/Mar02.ogg';
	import mar03Sfx from '$lib/Mar03.ogg';
	import mar04Sfx from '$lib/Mar04.ogg';
	import mar05Sfx from '$lib/Mar05.ogg';

	// Ghoul encounter assets
	import ghoulImg from '$lib/ghoul.png';
	import ghoulGif from '$lib/ghoul.gif';
	import ghoulSfx from '$lib/ghoul.ogg';
	import heartbeatSfx from '$lib/heartbeat.ogg';

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
	const defaultGoogleDocsLink = 'https://docs.google.com/document/d/1-jO8mpKGzuZTN_JkLC-NhOJ1hAucKX_O3ETe_3ZXTHM/edit?usp=sharing';
	let googleDocsLink = $state(defaultGoogleDocsLink);
	let rememberLink = $state(false);
	let isLinkLocked = $state(false);
	let userHasChangedLink = $state(false); // Track if user has modified the link

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

	// ========================================================================
	// SFX CONFIGURATION (Modular sound effects system)
	// ========================================================================
	let sfxEnabled = $state(true); // Master SFX toggle
	let wheelClickEnabled = $state(true); // Wheel click sound
	let wheelClickSound = $state('rimshot'); // 'wheel_click' or 'rimshot'
	let victoryEnabled = $state(true); // Victory sound
	let sfxVolume = $state(0.5); // Master volume (0-1)

	// Audio instances (created on mount to avoid SSR issues)
	let wheelClickAudioPool: HTMLAudioElement[] = []; // Pool of 10 instances for overlapping
	let wheelClickPoolIndex = 0; // Round-robin index
	let victoryAudio: HTMLAudioElement;

	// Movie data - default to 1-100 for testing (will be replaced by Google Docs data later)
	let movies = $state(Array.from({ length: 100 }, (_, i) => (i + 1).toString()));

	// Wheel state
	let wheelCanvas: HTMLCanvasElement;
	let rotation = $state(0);
	let isSpinning = $state(false);

	// ========================================================================
	// HAUNTED MODE CONFIGURATION (Modular for future expansion)
	// ========================================================================
	const hauntedConfig = {
		// Fuel system
		fuelDrainTime: 60000, // 60 seconds total
		fuelDrainInterval: 200, // Update every 200ms (slower, smoother)
		refuelRate: 3, // +3% per tick (slower refill)
		refuelInterval: 100, // 100ms per tick

		// Light properties
		lightRadiusMax: 300, // Full fuel radius (wider spread)
		lightRadiusMin: 40, // Empty fuel radius (tight beam)

		// Gas can properties
		gasCanSizeInitial: 100, // Starting size in px
		gasCanSizeMin: 20, // Minimum size in px
		gasCanShrinkRate: 0.94, // 6% smaller each refuel (was 8%)
		gasCanMinSizeThreshold: 10, // Stop shrinking after 10 refuels
		gasCanOpacity: 0.6, // Base opacity

		// Visual effects
		flickerIntensity: 0.008, // Light flicker amount (much subtler)
		flickerSpeed: 0.002, // How fast it flickers (much slower)
		vignetteIntensity: 0.95, // How dark the edges get (0-1)

		// Color temperature (warm → cool transition)
		warmR: 255, warmG: 220, warmB: 180, // Warm flashlight color
		coolR: 180, coolG: 200, coolB: 255, // Cool dying light color
		colorShiftThreshold: 20, // % fuel when color shift accelerates

		// Audio settings (increased by 6dB from original)
		bgMusicVolumeMin: 0.063, // -24dB ≈ 0.063 linear
		bgMusicVolumeMax: 0.126, // -18dB ≈ 0.126 linear
		warningVolume: 0.126, // -18dB ≈ 0.126 linear
		refillVolume: 0.126, // -18dB ≈ 0.126 linear
		tempoVariationAmount: 0.05, // ±5%
		tempoChangeInterval: 10000, // Change tempo every 10 seconds

		// Jumpscare timing
		jumpscareCountdownTime: Math.floor(10 + Math.random() * 11), // Random 10-20 seconds
		jumpscarePopDuration: 100, // Fast pop-in (0.1 seconds)
		jumpscareSitDuration: 1000, // Sit on screen (1 second)
		fadeoutDuration: 1500, // Fade to black (1.5 seconds)

		// Ghoul encounter settings
		ghoulSpawnCheckInterval: 3000, // Check every 3 seconds
		ghoulSpawnChance: 0.0167, // ~1.67% chance (1 per 3 minutes avg)
		ghoulFindTime: 3000, // 3 seconds to hover
		ghoulCountdownMin: 30, // 30 seconds min
		ghoulCountdownMax: 40, // 40 seconds max
		ghoulSizeInitial: 60, // Starting size (px) - smaller than before
		ghoulSizeMin: 25, // Minimum size
		ghoulShrinkRate: 0.93, // 7% smaller each encounter
		ghoulOpacity: 0.5, // Less visible than gas can
	};

	// Haunted Mode state variables
	let hauntedInit = false;

	// Lantern cursor effect for Haunted Mode
	let cursorX = $state(0);
	let cursorY = $state(0);

	// Enhanced fuel system for Haunted Mode
	let fuelLevel = $state(100); // 0-100, starts at full
	let fuelDrainInterval: number;
	let gasCanX = $state(0);
	let gasCanY = $state(0);
	let gasCanSize = $state(hauntedConfig.gasCanSizeInitial); // Dynamic size
	let gasCanRefuelCount = $state(0); // Track refuels for difficulty
	let lightFlicker = $state(0); // Flicker offset for realism
	let flickerAnimationFrame: number; // Animation frame ID

	// Horror Mode Audio
	let horrorBgAudio: HTMLAudioElement;
	let warningAudio: HTMLAudioElement;
	let jumpscareAudio: HTMLAudioElement;
	let refillAudio: HTMLAudioElement;
	let marionetteScreamAudios: HTMLAudioElement[] = []; // Array of marionette screams

	// Horror Game State
	let horrorGamePaused = $state(false);
	let jumpscareCountdown = $state(-1); // -1 = inactive, 0-15 = countdown seconds
	let jumpscareCountdownInterval: number;
	let showJumpscare = $state(false);
	let jumpscareAnimationProgress = $state(0); // 0-1 for animation
	let jumpscarePhase = $state<'idle' | 'jumpscare' | 'scream' | 'fadeout'>('idle'); // Track jumpscare phases

	// Audio State
	let bgMusicTempo = $state(1.0); // Playback rate for tempo variation
	let tempoVariationInterval: number;

	// Ghoul encounter state
	let ghoulActive = $state(false); // Is ghoul currently spawned?
	let ghoulX = $state(0); // X position
	let ghoulY = $state(0); // Y position
	let ghoulSize = $state(hauntedConfig.ghoulSizeInitial);
	let ghoulCountdown = $state(-1); // -1 = inactive
	let ghoulHoverDuration = $state(0); // ms hovering
	let ghoulHoverStartTime = 0; // timestamp
	let isHoveringGhoul = $state(false); // hover state

	let ghoulSpawnInterval: number; // spawn check interval
	let ghoulCountdownInterval: number; // countdown timer
	let ghoulHoverCheckInterval: number; // hover duration tracker

	let heartbeatAudio: HTMLAudioElement; // heartbeat sfx
	let ghoulJumpscareAudio: HTMLAudioElement; // ghoul jumpscare sfx

	// Jumpscare type tracking
	let currentJumpscareType = $state<'default' | 'ghoul'>('default');

	// Horror intro state
	let showHorrorIntro = $state(false);
	let lanternFadeProgress = $state(0); // 0 to 1

	let lanternFadeAnimationFrame: number;

	function handleMouseMove(e: MouseEvent) {
		cursorX = e.clientX;
		cursorY = e.clientY;
	}

	function startHorrorIntro() {
		console.log('[HORROR INTRO] Starting horror intro');
		showHorrorIntro = true;
		lanternFadeProgress = 0;
		const fadeDuration = 1500; // ms
		const startTime = Date.now();

		function animateFade() {
			const elapsed = Date.now() - startTime;
			lanternFadeProgress = Math.min(elapsed / fadeDuration, 1);
			if (lanternFadeProgress < 1) {
				lanternFadeAnimationFrame = requestAnimationFrame(animateFade);
			} else {
				console.log('[HORROR INTRO] Fade-in complete, starting game');
				showHorrorIntro = false;
				// Start horror game systems
				startFuelDrain();
				startFlickerAnimation();
				if (horrorBgAudio) {
					horrorBgAudio.currentTime = 0;
					horrorBgAudio.play().catch(() => {});
				}
				startTempoVariation();
			}
		}
		animateFade();
	}

	function stopHorrorIntroAnimation() {
		if (lanternFadeAnimationFrame) {
			cancelAnimationFrame(lanternFadeAnimationFrame);
		}
	}

	// ========================================================================
	// LIGHT FLICKER ANIMATION (Realistic flashlight wobble)
	// ========================================================================
	function startFlickerAnimation() {
		console.log('Starting flicker animation');

		function updateFlicker() {
			if (horrorGamePaused) {
				flickerAnimationFrame = requestAnimationFrame(updateFlicker);
				return; // Skip flicker updates when paused
			}

			// Subtle sine wave + random noise for realistic flicker
			const baseFlicker = Math.sin(Date.now() * hauntedConfig.flickerSpeed) * 0.5;
			const randomNoise = (Math.random() - 0.5) * 0.5;

			// Intensify flicker at low fuel (more desperation)
			const lowFuelMultiplier = fuelLevel < 20 ? (1 - fuelLevel / 20) * 2 : 0;

			lightFlicker = (baseFlicker + randomNoise) * hauntedConfig.flickerIntensity * (1 + lowFuelMultiplier);

			flickerAnimationFrame = requestAnimationFrame(updateFlicker);
		}
		updateFlicker();
	}

	function stopFlickerAnimation() {
		console.log('Stopping flicker animation');

		if (flickerAnimationFrame) {
			cancelAnimationFrame(flickerAnimationFrame);
		}
	}

	// ========================================================================
	// HORROR MODE AUDIO SYSTEM
	// ========================================================================
	function updateBgMusicVolume() {
		console.log('Updating background music volume based on fuel level');

		if (!horrorBgAudio || horrorGamePaused) return;

		// Logarithmic volume increase as fuel depletes
		// At 100% fuel: min volume, At 0% fuel: max volume
		const fuelRatio = fuelLevel / 100;
		const volumeRange = hauntedConfig.bgMusicVolumeMax - hauntedConfig.bgMusicVolumeMin;
		horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMax - (fuelRatio * volumeRange);
	}

	function startTempoVariation() {
		console.log('Starting tempo variation for background music');

		function randomizeTempo() {
			const variation = (Math.random() * 2 - 1) * hauntedConfig.tempoVariationAmount;
			bgMusicTempo = 1.0 + variation; // 0.95 to 1.05
			if (horrorBgAudio) {
				horrorBgAudio.playbackRate = bgMusicTempo;
			}
		}

		randomizeTempo(); // Initial tempo
		tempoVariationInterval = setInterval(randomizeTempo, hauntedConfig.tempoChangeInterval);
	}

	function stopTempoVariation() {
		console.log('Stopping tempo variation for background music');

		if (tempoVariationInterval) {
			clearInterval(tempoVariationInterval);
		}
	}

	// ========================================================================
	// JUMPSCARE SYSTEM
	// ========================================================================
	function startJumpscareCountdown() {
		console.log('Starting jumpscare countdown');

		jumpscareCountdown = hauntedConfig.jumpscareCountdownTime;

		// Play warning audio in a loop while countdown is active
		if (warningAudio && !horrorGamePaused) {
			warningAudio.loop = true;
			warningAudio.currentTime = 0;
			warningAudio.play().catch(() => {});
		}

		jumpscareCountdownInterval = setInterval(() => {
			if (horrorGamePaused) return; // Pause countdown

			jumpscareCountdown--;

			if (jumpscareCountdown <= 0) {
				clearInterval(jumpscareCountdownInterval);
				// Stop warning loop before jumpscare
				if (warningAudio) {
					warningAudio.loop = false;
					warningAudio.pause();
					warningAudio.currentTime = 0;
				}
				triggerJumpscare();
			}
		}, 1000);
	}

	// Add a ref for the jumpscare gif
	let jumpscareGifEl: HTMLImageElement | null = null;

	function triggerJumpscare() {
		console.log('Triggering jumpscare sequence');

		// Stop all horror audio
		if (horrorBgAudio) horrorBgAudio.pause();
		if (warningAudio) {
			warningAudio.loop = false;
			warningAudio.pause();
			warningAudio.currentTime = 0;
		}

		// Play jumpscare sound
		if (jumpscareAudio) {
			jumpscareAudio.currentTime = 0;
			jumpscareAudio.play().catch(() => {});
		}

		// Phase 1: Fast pop-in (200ms)
		showJumpscare = true;
		jumpscarePhase = 'jumpscare';
		jumpscareAnimationProgress = 0;

		// Reset gif by changing src (forces restart)
		if (jumpscareGifEl) {
			const src = jumpscareGifEl.src;
			jumpscareGifEl.src = '';
			jumpscareGifEl.src = src;
		}

		const popStartTime = Date.now();
		const popInterval = setInterval(() => {
			const elapsed = Date.now() - popStartTime;
			jumpscareAnimationProgress = Math.min(elapsed / hauntedConfig.jumpscarePopDuration, 1);

			if (jumpscareAnimationProgress >= 1) {
				clearInterval(popInterval);
				// Phase 2: Sit on screen for the exact gif duration
				// If you know the gif duration (e.g. 1.2s), set it here
				const gifDuration = 1200; // ms (adjust to match your gif length)
				setTimeout(() => {
					// Optionally, freeze the gif on last frame by replacing with a static image
					// If you have a static PNG/JPG of the last frame, swap here:
					// if (jumpscareGifEl) jumpscareGifEl.src = lastFrameImg;

					// Phase 3: Play random marionette scream
					jumpscarePhase = 'scream';
					const randomScream = marionetteScreamAudios[Math.floor(Math.random() * marionetteScreamAudios.length)];
					if (randomScream) {
						randomScream.currentTime = 0;
						randomScream.play().catch(() => {});
					}

					// Phase 4: Fade to black
					jumpscarePhase = 'fadeout';
					const fadeStartTime = Date.now();
					const fadeInterval = setInterval(() => {
						const fadeElapsed = Date.now() - fadeStartTime;
						const fadeProgress = fadeElapsed / hauntedConfig.fadeoutDuration;

						if (fadeProgress >= 1) {
							clearInterval(fadeInterval);
							// Reset after fade completes
							setTimeout(() => {
								resetHorrorMode();
							}, 300);
						}
					}, 16);
				}, gifDuration);
			}
		}, 16); // ~60fps
	}

	function resetHorrorMode() {
		console.log('Resetting horror mode after jumpscare');

		// Reset all horror state
		showJumpscare = false;
		jumpscareAnimationProgress = 0;
		jumpscarePhase = 'idle';
		jumpscareCountdown = -1;
		fuelLevel = 100;
		gasCanSize = hauntedConfig.gasCanSizeInitial;
		gasCanRefuelCount = 0;

		// Restart haunted intro (delays game logic)
		startHauntedIntro();
	}

	// Play horror start sound when entering haunted mode
	function playHorrorStart() {
		if (horrorStartAudio) {
			horrorStartAudio.currentTime = 0;
			horrorStartAudio.play().catch(() => {});
		}
	}

	// ========================================================================
	// GHOUL ENCOUNTER SYSTEM
	// ========================================================================

	// Start checking for ghoul spawns (called when haunted mode starts)
	function startGhoulSpawnSystem() {
		console.log('[GHOUL] Starting spawn system');

		ghoulSpawnInterval = setInterval(() => {
			if (horrorGamePaused || ghoulActive || showJumpscare) return;

			// Random spawn chance (~1 per 3 minutes)
			const roll = Math.random();
			if (roll < hauntedConfig.ghoulSpawnChance) {
				spawnGhoul();
			}
		}, hauntedConfig.ghoulSpawnCheckInterval);
	}

	function stopGhoulSpawnSystem() {
		if (ghoulSpawnInterval) clearInterval(ghoulSpawnInterval);
		despawnGhoul(); // Clean up any active ghoul
	}

	function spawnGhoul() {
		console.log('[GHOUL] Spawning ghoul');

		ghoulActive = true;
		repositionGhoul();

		// Play heartbeat sound
		if (heartbeatAudio) {
			heartbeatAudio.currentTime = 0;
			heartbeatAudio.play().catch(() => {});
		}

		// Start countdown timer (30-40 seconds)
		const countdownTime = Math.floor(
			hauntedConfig.ghoulCountdownMin +
			Math.random() * (hauntedConfig.ghoulCountdownMax - hauntedConfig.ghoulCountdownMin)
		);
		ghoulCountdown = countdownTime;

		ghoulCountdownInterval = setInterval(() => {
			if (horrorGamePaused) return;

			ghoulCountdown--;

			if (ghoulCountdown <= 0) {
				clearInterval(ghoulCountdownInterval);
				// Player failed to find ghoul - trigger jumpscare
				triggerGhoulJumpscare();
			}
		}, 1000);
	}

	function repositionGhoul() {
		// Similar to repositionGasCan - place away from cursor
		const width = typeof window !== 'undefined' ? window.innerWidth : 800;
		const height = typeof window !== 'undefined' ? window.innerHeight : 600;

		const xRange = width * 0.15;  // Bigger zone than gas can (15%)
		const yRange = height * 0.15;

		const mouseX = Math.max(0, Math.min(cursorX, width));
		const mouseY = Math.max(0, Math.min(cursorY, height));

		const margin = 100;
		const nearMinX = Math.max(margin, mouseX - xRange);
		const nearMaxX = Math.min(width - margin, mouseX + xRange);
		const nearMinY = Math.max(margin, mouseY - yRange);
		const nearMaxY = Math.min(height - margin, mouseY + yRange);

		let x, y;
		let attempts = 0;
		do {
			x = margin + Math.random() * (width - margin * 2);
			y = margin + Math.random() * (height - margin * 2);
			attempts++;
			if (attempts > 20) break;
		} while (
			x >= nearMinX && x <= nearMaxX &&
			y >= nearMinY && y <= nearMaxY
		);

		ghoulX = x;
		ghoulY = y;
	}

	function despawnGhoul() {
		console.log('[GHOUL] Despawning ghoul');

		ghoulActive = false;
		ghoulCountdown = -1;
		ghoulHoverDuration = 0;
		isHoveringGhoul = false;

		if (ghoulCountdownInterval) clearInterval(ghoulCountdownInterval);
		if (ghoulHoverCheckInterval) clearInterval(ghoulHoverCheckInterval);
		if (heartbeatAudio) heartbeatAudio.pause();
	}

	function onGhoulMouseEnter() {
		console.log('[GHOUL] Mouse enter - starting hover timer');

		isHoveringGhoul = true;
		ghoulHoverStartTime = Date.now();

		// Check hover duration every 100ms
		ghoulHoverCheckInterval = setInterval(() => {
			if (horrorGamePaused) return;

			const elapsed = Date.now() - ghoulHoverStartTime;
			ghoulHoverDuration = elapsed;

			if (ghoulHoverDuration >= hauntedConfig.ghoulFindTime) {
				// Success! Player found the ghoul
				clearInterval(ghoulHoverCheckInterval);
				ghoulFound();
			}
		}, 100);
	}

	function onGhoulMouseLeave() {
		console.log('[GHOUL] Mouse leave - resetting hover timer');

		isHoveringGhoul = false;
		ghoulHoverDuration = 0;

		if (ghoulHoverCheckInterval) {
			clearInterval(ghoulHoverCheckInterval);
		}
	}

	function ghoulFound() {
		console.log('[GHOUL] Player successfully found ghoul!');

		// Stop heartbeat
		if (heartbeatAudio) heartbeatAudio.pause();

		// Shrink ghoul for next encounter (progressive difficulty)
		ghoulSize = Math.max(
			hauntedConfig.ghoulSizeMin,
			ghoulSize * hauntedConfig.ghoulShrinkRate
		);

		// Despawn and allow next spawn
		despawnGhoul();
	}

	function triggerGhoulJumpscare() {
		console.log('[GHOUL] Triggering ghoul jumpscare');

		// Stop all horror audio
		if (horrorBgAudio) horrorBgAudio.pause();
		if (heartbeatAudio) heartbeatAudio.pause();
		if (warningAudio) warningAudio.pause();

		// Despawn ghoul
		despawnGhoul();

		// Set jumpscare type BEFORE showing overlay
		currentJumpscareType = 'ghoul';

		// Play ghoul jumpscare audio
		if (ghoulJumpscareAudio) {
			ghoulJumpscareAudio.currentTime = 0;
			ghoulJumpscareAudio.play().catch(() => {});

			// Cut audio after 4 seconds
			setTimeout(() => {
				if (ghoulJumpscareAudio) {
					ghoulJumpscareAudio.pause();
					ghoulJumpscareAudio.currentTime = 0;
				}
			}, 4000);
		}

		// Show jumpscare overlay
		showJumpscare = true;
		jumpscarePhase = 'jumpscare';
		jumpscareAnimationProgress = 0;

		// Reset GIF to restart animation
		if (jumpscareGifEl) {
			const src = jumpscareGifEl.src;
			jumpscareGifEl.src = '';
			jumpscareGifEl.src = src;
		}

		// Phase 1: Fast pop-in (100ms)
		const popStartTime = Date.now();
		const popInterval = setInterval(() => {
			const elapsed = Date.now() - popStartTime;
			jumpscareAnimationProgress = Math.min(
				elapsed / hauntedConfig.jumpscarePopDuration, 1
			);

			if (jumpscareAnimationProgress >= 1) {
				clearInterval(popInterval);

				// Phase 2: Display for 4 seconds
				const ghoulDisplayDuration = 4000;
				setTimeout(() => {
					// Phase 3: Fade to black
					jumpscarePhase = 'fadeout';
					const fadeStartTime = Date.now();
					const fadeInterval = setInterval(() => {
						const fadeElapsed = Date.now() - fadeStartTime;
						const fadeProgress = fadeElapsed / hauntedConfig.fadeoutDuration;

						if (fadeProgress >= 1) {
							clearInterval(fadeInterval);

							// Reset horror mode
							setTimeout(() => {
								// Reset to default jumpscare type
								currentJumpscareType = 'default';
								resetHorrorMode();
							}, 300);
						}
					}, 16);
				}, ghoulDisplayDuration);
			}
		}, 16); // ~60fps
	}

	// ========================================================================
	// PAUSE/RESUME SYSTEM
	// ========================================================================
	function handleVisibilityChange() {
		console.log('Handling visibility change for pause/resume');

		if (typeof document === 'undefined') return; // SSR guard
		if (selectedThemePreset !== 'haunted') return;

		if (document.hidden) {
			pauseHorrorMode();
		} else {
			resumeHorrorMode();
		}
	}

	function pauseHorrorMode() {
		console.log('Pausing horror mode');

		horrorGamePaused = true;

		// Mute/pause all audio
		if (horrorBgAudio) horrorBgAudio.volume = 0;
		if (warningAudio) warningAudio.pause();
		// All intervals check horrorGamePaused flag, so they'll freeze automatically
	}

	function resumeHorrorMode() {
		console.log('Resuming horror mode');

		horrorGamePaused = false;

		// Restore audio
		updateBgMusicVolume();
		if (horrorBgAudio && horrorBgAudio.paused) {
			horrorBgAudio.play().catch(() => {});
		}
		if (jumpscareCountdown >= 0 && warningAudio && warningAudio.paused) {
			warningAudio.play().catch(() => {});
		}
	}

	// ========================================================================
	// SFX SYSTEM (Sound Effects)
	// ========================================================================
	function playWheelClick() {

		if (!sfxEnabled || !wheelClickEnabled || wheelClickAudioPool.length === 0) return;

		// Use round-robin to grab next available audio instance from pool
		// This allows multiple clicks to play simultaneously without cutting each other off
		const audio = wheelClickAudioPool[wheelClickPoolIndex];
		wheelClickPoolIndex = (wheelClickPoolIndex + 1) % wheelClickAudioPool.length;

		// Random pitch detune: ±5 cents (0.05 semitones)
		// playbackRate formula: 2^(cents/1200)
		// ±5 cents = ±0.00289 playback rate adjustment
		const detuneCents = (Math.random() * 10) - 5; // -5 to +5 cents
		const detuneRate = Math.pow(2, detuneCents / 1200);
		audio.playbackRate = detuneRate;

		// Set volume and play (no currentTime reset - allows overlapping)
		audio.volume = sfxVolume;
		audio.currentTime = 0; // Reset this instance only
		audio.play().catch(() => {
			// Silently handle autoplay restrictions
		});
	}

	function playVictory() {
		console.log('Playing victory sound effect');

		if (!sfxEnabled || !victoryEnabled) return;

		// Choose appropriate victory sound based on theme
		const audioToPlay = selectedThemePreset === 'haunted'
			? horrorResultAudio
			: victoryAudio;

		if (!audioToPlay) return;

		audioToPlay.currentTime = 0;
		audioToPlay.volume = sfxVolume;
		audioToPlay.play().catch(() => {
			// Silently handle autoplay restrictions
		});
	}

	// ========================================================================
	// FUEL SYSTEM
	// ========================================================================
	function startFuelDrain() {
		console.log('Starting fuel drain');

		// Stop any existing drain first to prevent multiple intervals
		stopFuelDrain();

		// Drain fuel over 60 seconds
		// 60000ms / 200ms interval = 300 intervals
		// 100% / 300 = ~0.333% per interval
		fuelDrainInterval = setInterval(() => {
			if (horrorGamePaused) return; // Skip if paused

			if (fuelLevel > 0) {
				fuelLevel = Math.max(0, fuelLevel - (100 / 300)); // 300 intervals in 60 seconds
				updateBgMusicVolume(); // Dynamic volume adjustment
			} else if (fuelLevel === 0 && jumpscareCountdown === -1) {
				// Trigger jumpscare countdown when fuel runs out
				startJumpscareCountdown();
			}
		}, hauntedConfig.fuelDrainInterval);
	}

	function stopFuelDrain() {
		console.log('Stopping fuel drain');

		if (fuelDrainInterval) {
			clearInterval(fuelDrainInterval);
		}
	}

	function refuelLantern() {
		console.log('Refueling lantern from gas can');

		// Play refill sound
		if (refillAudio && sfxEnabled) {
			refillAudio.currentTime = 0;
			refillAudio.play().catch(() => {});
		}

		// Track refuel count for progressive difficulty
		gasCanRefuelCount++;

		// Shrink gas can progressively (gets harder to find over time)
		// Stop shrinking after 10 refuels to maintain minimum viable size
		if (gasCanRefuelCount < hauntedConfig.gasCanMinSizeThreshold) {
			gasCanSize = Math.max(
				hauntedConfig.gasCanSizeMin,
				gasCanSize * hauntedConfig.gasCanShrinkRate
			);
		}

		// Gradually refill to 100%
		const refillInterval = setInterval(() => {
			if (fuelLevel < 100) {
				fuelLevel = Math.min(100, fuelLevel + hauntedConfig.refuelRate);
			} else {
				clearInterval(refillInterval);
				repositionGasCan(); // Move gas can to new location after refuel
			}
		}, hauntedConfig.refuelInterval);
	}

	function repositionGasCan() {
		console.log('Repositioning gas can away from mouse');

		// Get screen dimensions
		const width = typeof window !== 'undefined' ? window.innerWidth : 800;
		const height = typeof window !== 'undefined' ? window.innerHeight : 600;

		// 10% of screen dimensions
		const xRange = width * 0.1;
		const yRange = height * 0.1;

		// Clamp mouse position to screen
		const mouseX = Math.max(0, Math.min(cursorX, width));
		const mouseY = Math.max(0, Math.min(cursorY, height));

		const margin = 100;

		// Define "near mouse" region
		const nearMinX = Math.max(margin, mouseX - xRange);
		const nearMaxX = Math.min(width - margin, mouseX + xRange);
		const nearMinY = Math.max(margin, mouseY - yRange);
		const nearMaxY = Math.min(height - margin, mouseY + yRange);

		// Pick a random position OUTSIDE the "near mouse" region
		let x, y;
		let attempts = 0;
		do {
			x = margin + Math.random() * (width - margin * 2);
			y = margin + Math.random() * (height - margin * 2);
			attempts++;
			// Avoid infinite loop if screen is very small
			if (attempts > 20) break;
		} while (
			x >= nearMinX && x <= nearMaxX &&
			y >= nearMinY && y <= nearMaxY
		);

		gasCanX = x;
		gasCanY = y;
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

	// Add state for item count notification
	let showItemCountNotif = $state(false);
	let itemCountNotifText = $state('');
	let itemCountNotifTimeout: number;

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
			userHasChangedLink = true;
		} else {
			// Unlock and clear saved link
			localStorage.removeItem('slippymud_google_docs_link');
			isLinkLocked = false;
		}
	}

	// Handle when user types in the link input
	function handleLinkInput(event: Event) {
		const newLink = (event.target as HTMLInputElement).value;
		// Mark that user has changed the link if it differs from default
		if (newLink !== defaultGoogleDocsLink) {
			userHasChangedLink = true;
			// Auto-check remember link when user changes from default
			if (!rememberLink) {
				rememberLink = true;
			}
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

			// Show notification instead of alert
			itemCountNotifText = `+${lines.length} items`;
			showItemCountNotif = true;
			if (itemCountNotifTimeout) clearTimeout(itemCountNotifTimeout);
			itemCountNotifTimeout = setTimeout(() => {
				showItemCountNotif = false;
			}, 2200);

			// Remove ALL alert calls for item count
			// (No alert for manual refresh or auto-load)
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

		console.log(`Applying theme preset: ${preset}`);

		if (preset === 'penguin') {

			// Penguin Theme (Arctic)
			primaryColor = '#1E90FF'; // Dodger blue
			secondaryColor = '#FFFFFF'; // White
			wheelBackground = '#E0F2FF'; // Light ice blue
			selectedFont = 'Arial';
			wheelColors = ['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6'];
			backgroundWallpaper = 'arctic_bg.jpg';
			wheelCenterImage = 'penguin.jpg';
			wheelClickSound = 'rimshot'; // Ensure penguin theme uses rimshot or default

		} else if (preset === 'haunted') {

			// Haunted Mode (Black & Red)
			primaryColor = '#8B0000'; // Dark red
			secondaryColor = '#000000'; // Black
			wheelBackground = '#1a0000'; // Very dark red
			selectedFont = 'Georgia';
			wheelColors = ['#8B0000', '#FF0000', '#DC143C', '#B22222', '#A52A2A'];
			backgroundWallpaper = 'creepy.jpg';
			wheelCenterImage = 'hands.jpg';
			wheelClickSound = 'wood_wheel'; // Auto set to wood creak for haunted theme

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

	// Track last click time for wheel click spacing
	let lastClickRotation = 0;

	function animateSpin() {
		if (!isSpinning) return;

		const now = Date.now();
		const elapsed = now - spinStartTime;
		const progress = Math.min(elapsed / spinDuration, 1); // 0 to 1

		// Easing function - ease out cubic for smooth deceleration
		const eased = 1 - Math.pow(1 - progress, 3);

		// Calculate current rotation based on eased progress
		rotation = spinStartRotation + (targetRotation - spinStartRotation) * eased;

		// Wheel click SFX: Play clicks as wheel passes segments, slowing down over time
		// Click interval increases as wheel decelerates (mimics ratchet/clicker slowing)
		const rotationDelta = Math.abs(rotation - lastClickRotation);

		// Dynamic click threshold: starts small (frequent clicks), increases as wheel slows
		// At start (progress ~0): click every ~10 degrees (fast)
		// At end (progress ~1): click every ~60 degrees (slow)
		const clickThreshold = 10 + (progress * 50); // 10° → 60° threshold

		if (rotationDelta >= clickThreshold) {
			playWheelClick();
			lastClickRotation = rotation;
		}

		// Check if spin is complete (5 seconds have passed)
		if (progress >= 1) {
			rotation = targetRotation % 360;
			isSpinning = false;
			drawWheel();

			// Play victory sound when wheel stops
			// (playVictory now handles horror mode automatically)
			playVictory();

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

	// Declare horrorStartAudio and horrorResultAudio at the top
	let horrorStartAudio: HTMLAudioElement;
	let horrorResultAudio: HTMLAudioElement;

	// Helper to load wheel click sounds into pool
	function loadWheelClickPool() {
		wheelClickAudioPool = [];
		let soundFile;
		if (wheelClickSound === 'rimshot') {
			soundFile = rimClickSfx;
		} else if (wheelClickSound === 'wood_wheel') {
			soundFile = woodWheelSfx;
		} else {
			soundFile = wheelClickSfx;
		}
		for (let i = 0; i < 10; i++) {
			wheelClickAudioPool.push(new Audio(soundFile));
		}
		wheelClickPoolIndex = 0;
	}

	onMount(() => {
		// Initialize audio instances (must be done client-side to avoid SSR issues)
		loadWheelClickPool();
		victoryAudio = new Audio(victorySfx);

		// Initialize horror mode audio
		horrorBgAudio = new Audio(horrorBgSfx);
		horrorBgAudio.loop = true;
		horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMin;

		warningAudio = new Audio(fnafHallwaySfx);
		warningAudio.loop = false;
		warningAudio.volume = hauntedConfig.warningVolume;

		jumpscareAudio = new Audio(fnafJumpscareSfx);

		refillAudio = new Audio(refillSfx);
		refillAudio.volume = hauntedConfig.refillVolume;

		// Initialize marionette scream audio pool
		marionetteScreamAudios = [
			new Audio(mar01Sfx),
			new Audio(mar02Sfx),
			new Audio(mar03Sfx),
			new Audio(mar04Sfx),
			new Audio(mar05Sfx)
		];

		// Initialize horror start/result audio
		horrorStartAudio = new Audio(horrorStartSfx);
		horrorResultAudio = new Audio(horrorResultSfx);

		// Initialize ghoul encounter audio
		heartbeatAudio = new Audio(heartbeatSfx);
		heartbeatAudio.volume = hauntedConfig.warningVolume; // Same as warning
		heartbeatAudio.loop = true;

		ghoulJumpscareAudio = new Audio(ghoulSfx);

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
			userHasChangedLink = true; // User has a saved link
		}

		// Auto-load movies from whatever link is in the input field (default or saved)
		if (googleDocsLink && googleDocsLink.trim()) {
			console.log('[INIT] Auto-loading movies from Google Docs link:', googleDocsLink);
			handleRefresh();
		} else {
			console.log('[INIT] No link provided, using fallback 1-100');
		}

		// Initialize gas can position
		repositionGasCan();

		// Add visibility change listener for pause/resume
		document.addEventListener('visibilitychange', handleVisibilityChange);

		// Note: Haunted Mode systems are started by the $effect() below, not here
		// This prevents duplicate intervals from running
	});

	onDestroy(() => {
		// Clean up animation frame
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
		}
		// Clean up fuel drain interval
		stopFuelDrain();
		// Clean up flicker animation
		stopFlickerAnimation();
		// Clean up horror mode systems
		stopTempoVariation();
		stopGhoulSpawnSystem(); // Clean up ghoul system
		if (horrorBgAudio) horrorBgAudio.pause();
		if (warningAudio) warningAudio.pause();
		if (heartbeatAudio) heartbeatAudio.pause();
		if (jumpscareCountdownInterval) clearInterval(jumpscareCountdownInterval);
		// Remove visibility change listener (browser only)
		if (typeof document !== 'undefined') {
			document.removeEventListener('visibilitychange', handleVisibilityChange);
		}
		stopHorrorIntroAnimation();
		stopLanternIntroAnimation();
		if (itemCountNotifTimeout) clearTimeout(itemCountNotifTimeout);
	});

	// Watch for theme changes to start/stop Haunted Mode systems
	$effect(() => {
		if (selectedThemePreset === 'haunted' && hauntedInit == false) {
			hauntedInit = true;
			if (showThemeSettings) showThemeSettings = false;
			// Start bg music and tempo variation ONCE, immediately
			if (horrorBgAudio) {
				horrorBgAudio.currentTime = 0;
				horrorBgAudio.play().catch(() => {});
			}
			startTempoVariation();
			// Start haunted intro animation (delays game logic)
			startHauntedIntro();
			console.log(`[HAUNTED INTRO] Haunted Mode systems will start after intro`);
		} else if (selectedThemePreset !== 'haunted' && hauntedInit == true){
			console.log('Haunted Mode deactivated - stopping horror systems');
			hauntedInit = false;
			stopFuelDrain();
			stopFlickerAnimation();
			stopTempoVariation();
			stopLanternIntroAnimation();
			stopGhoulSpawnSystem(); // Stop ghoul system
			if (horrorBgAudio) horrorBgAudio.pause();
			if (warningAudio) warningAudio.pause();
			if (jumpscareCountdownInterval) clearInterval(jumpscareCountdownInterval);
			fuelLevel = 100;
			gasCanSize = hauntedConfig.gasCanSizeInitial;
			gasCanRefuelCount = 0;
			lightFlicker = 0;
			showJumpscare = false;
			jumpscareCountdown = -1;
			horrorGamePaused = false;
			lanternIntroProgress = 1;
		}
	});

	// Watch for wheel click sound changes and reload audio pool
	$effect(() => {
		// Track wheelClickSound to trigger reload
		wheelClickSound;

		if (wheelClickAudioPool.length > 0) {
			// Only reload if audio is already initialized (skip on initial load)
			loadWheelClickPool();
		}
	});

	// Lantern intro progress for haunted mode (0 = dark, 1 = normal lantern)
	let lanternIntroProgress = $state(1); // 1 = normal, 0 = dark
	let lanternIntroAnimationFrame: number;

		// Helper to start haunted intro animation
		function startHauntedIntro() {
			console.log('[HAUNTED INTRO] Starting haunted intro');
			playHorrorStart();
			lanternIntroProgress = 0;
			const fadeDuration = 1500; // ms
			const startTime = Date.now();
	
			function animateLanternIntro() {
				const elapsed = Date.now() - startTime;
				lanternIntroProgress = Math.min(elapsed / fadeDuration, 1);
				if (lanternIntroProgress < 1) {
					lanternIntroAnimationFrame = requestAnimationFrame(animateLanternIntro);
				} else {
					console.log('[HAUNTED INTRO] Lantern fade-in complete, starting game');
					// Start game logic only (no bg music here)
					startFuelDrain();
					startFlickerAnimation();
					repositionGasCan();
					startGhoulSpawnSystem(); // Start ghoul spawn system
				}
			}
			animateLanternIntro();
		}
	
		function stopLanternIntroAnimation() {
			if (lanternIntroAnimationFrame) {
				cancelAnimationFrame(lanternIntroAnimationFrame);
			}
		}
	</script>
	
		<!-- ============================================================================
		<!-- MAIN LAYOUT
		// ============================================================================ -->

	<div class="app-container" role="application" style="font-family: {selectedFont}; background-color: {wheelBackground}; {backgroundWallpaper === 'arctic_bg.jpg' ? `background-image: url('${arcticBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : backgroundWallpaper === 'creepy.jpg' ? `background-image: url('${creepyBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : ''}" onmousemove={handleMouseMove}>

		{#if selectedThemePreset === 'haunted'}
		<!-- Use only one lantern-overlay for haunted mode, always rendered -->
		<div
			class="lantern-overlay haunted-intro"
			style="
				--cursor-x: {cursorX}px;
				--cursor-y: {cursorY}px;
				--fuel-level: {fuelLevel};
				--flicker: {lightFlicker};
				--lantern-intro: {lanternIntroProgress};
			"
		></div>
		<!-- Show gas can only after intro is done -->
		{#if lanternIntroProgress === 1}
			<img
				src={gasCanImg}
				alt="Gas Can"
				class="gas-can"
				style="left: {gasCanX}px; top: {gasCanY}px; width: {gasCanSize}px;"
				onmouseenter={refuelLantern}
			/>
		{/if}

		<!-- Ghoul Encounter (hidden in darkness until found) -->
		{#if ghoulActive && lanternIntroProgress === 1}
			<img
				src={ghoulImg}
				alt="Ghoul"
				class="ghoul"
				class:hovering={isHoveringGhoul}
				style="left: {ghoulX}px; top: {ghoulY}px; width: {ghoulSize}px; opacity: {hauntedConfig.ghoulOpacity};"
				onmouseenter={onGhoulMouseEnter}
				onmouseleave={onGhoulMouseLeave}
			/>
		{/if}
		{/if}

		<!-- Jumpscare Overlay -->
		{#if showJumpscare}
		<div class="jumpscare-overlay" class:fadeout={jumpscarePhase === 'fadeout'}>
			{#if jumpscarePhase === 'jumpscare' || jumpscarePhase === 'scream'}
				<img
					src={currentJumpscareType === 'ghoul' ? ghoulGif : fnafJumpscareGif}
					alt="Jumpscare"
					class="jumpscare-gif"
					style="opacity: {jumpscareAnimationProgress};"
					bind:this={jumpscareGifEl}
					{...{loop: false}}
				/>
			{/if}
		</div>
		{/if}

		<!-- Header :3 -->
		<header style="background-color: {primaryColor}; border-bottom-color: {primaryColor};">
			<h1 style="color: {secondaryColor};">SlippyMudWheel</h1>
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
						oninput={handleLinkInput}
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
				{#if showItemCountNotif}
					<div class="item-count-notif" style="color: {secondaryColor};">
						{itemCountNotifText}
					</div>
				{/if}
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
						<div class="history-item" style="border-color: {primaryColor}; background-color: {secondaryColor};">
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
		<p class="footer-small" style="color: {primaryColor};">
			alicevoid - {selectedThemePreset === 'haunted'
				? 'make sure to be EVIL to yourself </3'
				: 'make sure to be nice to yourself <3'}
		</p>
	</footer>
	</div>

	<!-- ============================================================================ -->
	<!-- THEME SETTINGS MODAL -->
	<!-- ============================================================================ -->

	{#if showThemeSettings}
		<!-- Backdrop that closes modal when clicked -->
		<div class="theme-settings-backdrop" class:closing={isClosingThemeSettings} onclick={toggleThemeSettings}></div>
		<div class="theme-settings-panel" class:closing={isClosingThemeSettings} style="background-color: {secondaryColor};" onclick={(e) => e.stopPropagation()}>
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

					<!-- ARCTIC -->
					<button
						class="preset-button"
						class:active={selectedThemePreset === 'penguin'}
						style="border-color: {primaryColor}; {selectedThemePreset === 'penguin' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
						onclick={() => applyThemePreset('penguin')}
					>
						🐧 Penguin (Default)
					</button>

					<!-- HAUNTED -->
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

			<!-- Sound Effects Section -->
			<section class="settings-section" style="border-bottom-color: {primaryColor};">
				<h3 style="color: {primaryColor};">Sound Effects</h3>

				<!-- Master SFX Toggle -->
				<label class="checkbox-label">
					<input type="checkbox" bind:checked={sfxEnabled} />
					<span style="color: {primaryColor};">Enable Sound Effects</span>
				</label>

				<!-- Volume Control -->
				<div class="volume-setting">
					<label for="sfx-volume" style="color: {primaryColor};">Master Volume</label>
					<input
						id="sfx-volume"
						type="range"
						min="0"
						max="1"
						step="0.05"
						bind:value={sfxVolume}
						disabled={!sfxEnabled}
						style="--track-color: {primaryColor};"
					/>
					<span style="color: {primaryColor};">{Math.round(sfxVolume * 100)}%</span>
				</div>

				<!-- Individual SFX Toggles -->
				<div class="sfx-toggles">
					<label class="checkbox-label">
						<input type="checkbox" bind:checked={wheelClickEnabled} disabled={!sfxEnabled} />
						<span style="color: {primaryColor};">Wheel Click</span>
					</label>

					<!-- Wheel Click Sound Selector -->
					<div class="sound-selector">
						<label for="wheel-click-sound" style="color: {primaryColor};">Click Sound:</label>
						<select
							id="wheel-click-sound"
							bind:value={wheelClickSound}
							disabled={!sfxEnabled || !wheelClickEnabled}
							style="border-color: {primaryColor};"
						>
							<option value="wheel_click">Wheel Click</option>
							<option value="rimshot">Rimshot</option>
							<option value="wood_wheel">Wood Creak</option>
						</select>
					</div>

					<label class="checkbox-label">
						<input type="checkbox" bind:checked={victoryEnabled} disabled={!sfxEnabled} />
						<span style="color: {primaryColor};">Victory</span>
					</label>
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

		/* SFX Controls */
		.volume-setting {
			display: flex;
			align-items: center;
			gap: 1rem;
			margin-top: 1rem;
		}

		.volume-setting label {
			min-width: 120px;
			font-weight: 500;
		}

		.volume-setting input[type="range"] {
			flex: 1;
			height: 6px;
			border-radius: 3px;
			outline: none;
			-webkit-appearance: none;
			appearance: none;
			background: #ddd;
		}

		.volume-setting input[type="range"]::-webkit-slider-thumb {
			-webkit-appearance: none;
			appearance: none;
			width: 18px;
			height: 18px;
			border-radius: 50%;
			background: var(--track-color, #1E90FF);
			cursor: pointer;
		}

		.volume-setting input[type="range"]::-moz-range-thumb {
			width: 18px;
			height: 18px;
			border-radius: 50%;
			background: var(--track-color, #1E90FF);
			cursor: pointer;
			border: none;
		}

		.volume-setting input[type="range"]:disabled {
			opacity: 0.5;
			cursor: not-allowed;
		}

		.volume-setting span {
			min-width: 45px;
			text-align: right;
			font-weight: 500;
		}

		.sfx-toggles {
			display: flex;
			flex-direction: column;
			gap: 0.5rem;
			margin-top: 1rem;
		}

		.sound-selector {
			display: flex;
			align-items: center;
			gap: 0.75rem;
			margin-left: 2rem;
			margin-top: 0.25rem;
			margin-bottom: 0.5rem;
		}

		.sound-selector label {
			font-size: 0.9rem;
			font-weight: 500;
		}

		.sound-selector select {
			padding: 0.4rem 0.6rem;
			border-radius: 4px;
			border: 2px solid;
			background: white;
			font-size: 0.9rem;
			cursor: pointer;
			min-width: 140px;
		}

		.sound-selector select:disabled {
			opacity: 0.5;
			cursor: not-allowed;
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

		/* Backdrop for closing modal on outside click */
		.theme-settings-backdrop {
			position: fixed;
			top: 0;
			left: 0;
			width: 100vw;
			height: 100vh;
			background: rgba(0, 0, 0, 0.3);
			z-index: 999;
			animation: fadeIn 0.3s ease-out;
			cursor: pointer;
		}

		.theme-settings-backdrop.closing {
			animation: fadeOut 0.3s ease-out;
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
			pointer-events: auto; /* Allow clicks on the panel */
		}

		.theme-settings-panel * {
			pointer-events: auto; /* But allow interaction with children */
		}

		.theme-settings-panel.closing {
			animation: slideOut 0.3s ease-out forwards;
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
			z-index: 99999; /* Cover everything including theme settings modal */

			/* ====================================================================
			   DYNAMIC LIGHT PROPERTIES (with flicker)
			   ==================================================================== */
			--light-radius: calc(
				(40px + (var(--fuel-level) * 2.6px)) * (1 + var(--flicker))
			); /* 40px → 300px with flicker wobble */
			--light-brightness: calc(var(--fuel-level) / 100);

			/* Logarithmic color temperature (warm → cool transition)
			   Stays warm until ~20% fuel, then rapidly shifts to cool blue */
			--coolness-factor: calc(max(0, 1 - pow(var(--fuel-level) / 20, 2)));

			--warm-red: calc(255 - (75 * var(--coolness-factor)));    /* 255 → 180 */
			--warm-green: calc(220 - (20 * var(--coolness-factor)));  /* 220 → 200 */
			--warm-blue: calc(180 + (75 * var(--coolness-factor)));   /* 180 → 255 */

			/* Vignette: edges get darker as fuel depletes (desperation effect) */
			/* More gradual darkness increase - starts at 85% opaque, goes to 95% */
			--vignette: calc(0.85 + (0.10 * (1 - var(--light-brightness))));

			/* ====================================================================
			   OPPRESSIVE DARKNESS WITH REALISTIC FLASHLIGHT FALLOFF
			   ==================================================================== */
			background: radial-gradient(
				circle var(--light-radius) at var(--cursor-x) var(--cursor-y),
				/* Center: Nearly transparent (flashlight core) */
				rgba(var(--warm-red), var(--warm-green), var(--warm-blue), 0.05) 0%,
				rgba(var(--warm-red), var(--warm-green), var(--warm-blue), 0.08) 10%,

				/* Inner cone: Visible light (what you can actually see by) */
				rgba(var(--warm-red), var(--warm-green), var(--warm-blue), 0.15) 20%,
				rgba(var(--warm-red), var(--warm-green), var(--warm-blue), 0.25) 35%,

				/* Mid falloff: More gradual transition from light to shadow */
				rgba(0, 0, 0, calc(0.35 * (1 - var(--light-brightness) * 0.4))) 50%,
				rgba(0, 0, 0, calc(0.55 * (1 - var(--light-brightness) * 0.3))) 65%,

				/* Outer falloff: Less aggressive darkness */
				rgba(0, 0, 0, 0.75) 80%,
				rgba(0, 0, 0, 0.85) 90%,
				rgba(0, 0, 0, var(--vignette)) 97%,

				/* Edge vignette: Still dark but less oppressive */
				rgba(0, 0, 0, 0.95) 100%
			);
			transition: background 0.3s ease-out; /* Slower, smoother transitions */
		}

		.lantern-overlay.haunted-intro {
			--light-radius: calc(
				(40px + (var(--fuel-level) * 2.6px)) * (1 + var(--flicker)) * var(--lantern-intro)
			); /* 0px → normal radius */
			--light-brightness: calc((var(--fuel-level) / 100) * var(--lantern-intro));
			transition: background 0.3s, --light-radius 1.5s, --light-brightness 1.5s;
		}

		.gas-can {
			position: fixed;
			/* Width is set dynamically via inline style (shrinks each refuel) */
			height: auto;
			z-index: 10000; /* Below lantern overlay (99999) so it's hidden in darkness */
			cursor: pointer;
			transition: transform 0.3s ease, filter 0.3s ease, opacity 0.3s ease;
			opacity: 0.7; /* Visible but not too obvious */
			filter: drop-shadow(0 0 12px rgba(255, 200, 0, 0.6));
			pointer-events: auto; /* Ensure it can be interacted with */
		}

		.gas-can:hover {
			transform: scale(1.15);
			opacity: 1;
			filter: drop-shadow(0 0 25px rgba(255, 200, 0, 0.9)) drop-shadow(0 0 10px rgba(255, 150, 0, 0.7));
		}

		/* ====================================================================
		   GHOUL ENCOUNTER
		   ==================================================================== */
		.ghoul {
			position: fixed;
			height: auto;
			z-index: 10000; /* Same as gas can - hidden in darkness */
			cursor: crosshair; /* Different cursor to indicate danger */
			filter: drop-shadow(0 0 8px rgba(255, 0, 0, 0.4));
			pointer-events: auto;
		}

		.ghoul:hover {
			filter: drop-shadow(0 0 15px rgba(255, 0, 0, 0.8));
		}

		/* Shake effect when hovering over ghoul */
		.ghoul.hovering {
			animation: ghoulShake 0.15s ease-in-out infinite;
		}

		@keyframes ghoulShake {
			0%, 100% {
				transform: translate(0, 0) rotate(0deg);
			}
			25% {
				transform: translate(-2px, 1px) rotate(-1deg);
			}
			50% {
				transform: translate(2px, -1px) rotate(1deg);
			}
			75% {
				transform: translate(-1px, 2px) rotate(-0.5deg);
			}
		}

		/* ====================================================================
		   JUMPSCARE OVERLAY
		   ==================================================================== */
		.jumpscare-overlay {
			position: fixed;
			top: 0;
			left: 0;
			width: 100vw;
			height: 100vh;
			z-index: 100000; /* Above everything */
			display: flex;
			align-items: center;
			justify-content: center;
			background: black;
			pointer-events: none;
			transition: none;
		}

		.jumpscare-overlay.fadeout {
			animation: fadeToBlack 1.5s ease-out forwards;
		}

		.jumpscare-gif {
			width: 100%;
			height: 100%;
			object-fit: cover;
			transition: opacity 0.2s ease-out;
		}

		@keyframes fadeToBlack {
			from {
				background: black;
			}
			to {
				background: black;
				opacity: 0;
			}
		}

		.item-count-notif {
			position: absolute;
			top: 18px;
			left: 18px;
			font-size: 1.3rem;
			font-weight: bold;
			color: var(--secondary-color, #fff);
			background: rgba(0,0,0,0.08);
			padding: 0.35em 1em;
			border-radius: 8px;
			pointer-events: none;
			opacity: 1;
			animation: fadeItemNotif 2.2s ease-out forwards;
			z-index: 20;
		}

		@keyframes fadeItemNotif {
			0% { opacity: 0; transform: translateY(-10px);}
			10% { opacity: 1; transform: translateY(0);}
			80% { opacity: 1; }
			100% { opacity: 0; transform: translateY(-10px);}
		}

	</style>
