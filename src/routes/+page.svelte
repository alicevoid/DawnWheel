<script lang="ts">
	// ============================================================================
	// SLIPPYMUDWHEEL - Movie Wheel Spinner
	// ============================================================================

	import { onMount, onDestroy, tick } from 'svelte';
	import arcticBg from '$lib/arctic_bg.jpg';
	// import creepyBg from '$lib/creepy.jpg'; // Lazy loaded
	import penguinImg from '$lib/penguin.jpg';
	// import handsImg from '$lib/hands.jpg'; // Lazy loaded
	import gasCanImg from '$lib/gas_can.png';
	import blackTag from '$lib/black.png';
	import black2Tag from '$lib/black2.png';
	import wheelClickSfx from '$lib/wheel_click.ogg';
	import rimClickSfx from '$lib/rim_click.ogg';
	import victorySfx from '$lib/victory.ogg';

	// Horror Mode Audio - Lazy loaded (except wood wheel which is used for haunted theme wheel click)
	// import refillSfx from '$lib/refill.ogg'; // Lazy loaded
	// import horrorBgSfx from '$lib/horror_bg.ogg'; // Lazy loaded
	// import fnafHallwaySfx from '$lib/fnaf2_hallway_sfx.ogg'; // Lazy loaded
	// import fnafJumpscareSfx from '$lib/fnaf2_puppet_js.ogg'; // Lazy loaded
	// import fnafJumpscareGif from '$lib/fnaf_jumpscare.gif'; // Lazy loaded
	// import horrorStartSfx from '$lib/horror_start.ogg'; // Lazy loaded
	// import horrorResultSfx from '$lib/horror_result.ogg'; // Lazy loaded
	import woodWheelSfx from '$lib/wood_wheel.ogg';
	import KICK13Sfx from '$lib/KICK_13.ogg';
	import taikoDrumSfx from '$lib/taiko_drum.ogg';
	import alien1Sfx from '$lib/alien_1.ogg';
	import monkeySfx from '$lib/monkey.ogg';

	// Marionette scream SFX (random one plays before reset) - Lazy loaded
	// import mar01Sfx from '$lib/Mar01.ogg'; // Lazy loaded
	// import mar02Sfx from '$lib/Mar02.ogg'; // Lazy loaded
	// import mar03Sfx from '$lib/Mar03.ogg'; // Lazy loaded
	// import mar04Sfx from '$lib/Mar04.ogg'; // Lazy loaded
	// import mar05Sfx from '$lib/Mar05.ogg'; // Lazy loaded

	// Ghoul encounter assets - Lazy loaded
	import ghoulImg from '$lib/ghoul.png';
	import ghoulGif from '$lib/ghoul.gif';
	// import ghoulSfx from '$lib/ghoul.ogg'; // Lazy loaded
	// import heartbeatSfx from '$lib/heartbeat.ogg'; // Lazy loaded


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
	let selectedThemePreset = $state<'penguin' | 'haunted' | 'monkey' | 'user'>('user'); // Default to user theme

	// Theme state - Active theme (whatever is currently selected)
	let primaryColor = $state('#1E90FF'); // Dodger blue
	let secondaryColor = $state('#FFFFFF'); // White
	let wheelBackground = $state('#E0F2FF'); // Light ice blue
	let selectedFont = $state('Arial');
	let wheelColors = $state(['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6']);
	let backgroundWallpaper = $state('arctic_bg.jpg'); // Arctic background
	let wheelCenterImage = $state('penguin.jpg'); // Penguin center image

	// Custom Theme SAVED state (persistent, protected)
	let customPrimaryColor = $state('#1E90FF');
	let customSecondaryColor = $state('#FFFFFF');
	let customWheelBackground = $state('#E0F2FF');
	let customSelectedFont = $state('Arial');
	let customWheelColors = $state(['#1E90FF', '#87CEEB', '#4682B4', '#5F9EA0', '#B0E0E6']);
	let customBackgroundWallpaper = $state('arctic_bg.jpg');
	let customWheelCenterImage = $state('penguin.jpg');
	let customWheelClickSound = $state('rimshot');
	let customResultSound = $state('alien');  // Default to alien for custom theme

	// Custom upload data URLs (for custom theme)
	let customWallpaperDataUrl = $state<string | null>(null);
	let customCenterImageDataUrl = $state<string | null>(null);
	let customWheelClickDataUrl = $state<string | null>(null);
	let customResultDataUrl = $state<string | null>(null);

	// Responsive canvas sizing
	let canvasSize = $state(500); // Default size, will be updated dynamically

	// ========================================================================
	// SFX CONFIGURATION (Modular sound effects system)
	// ========================================================================
	let sfxEnabled = $state(true); // Master SFX toggle
	let wheelClickEnabled = $state(true); // Wheel click sound
	let wheelClickSound = $state('rimshot'); // 'wheel_click' or 'rimshot'
	let resultSound = $state('alien'); // 'alien' (default), 'victory', 'monkey', 'horror_result'
	let resultEnabled = $state(true); // Result sound
	let sfxVolume = $state(0.2); // Master volume (0-1)

	// Audio instances (created on mount to avoid SSR issues)
	let wheelClickAudioPool: HTMLAudioElement[] = []; // Pool of 10 instances for overlapping
	let wheelClickPoolIndex = 0; // Round-robin index
	let resultAudio: HTMLAudioElement;

	// ========================================================================
	// LAZY LOADING STATE
	// ========================================================================
	let horrorAssetsLoaded = $state(false);
	let horrorAssetsLoading = $state(false);
	let monkeyAssetsLoaded = $state(false);
	let monkeyAssetsLoading = $state(false);

	// Lazy-loaded asset URLs (populated when horror mode assets load)
	let horrorBackgroundUrl = $state<string | null>(null);
	let horrorCenterImageUrl = $state<string | null>(null);
	let horrorJumpscareGifUrl = $state<string | null>(null);

	// Lazy-loaded monkey mode asset URLs
	let monkeyBackgroundUrl = $state<string | null>(null);
	let monkeyCenterImageUrl = $state<string | null>(null);
	let monkeyWheelClickUrl = $state<string | null>(null);

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

		// Candlelight multi-layered radius system (NEW - Phase 4 Enhanced)
		// Core zone - always bright, minimal flicker
		coreRadiusMin: 45,
		coreRadiusMax: 55,

		// Inner glow - main illumination zone, moderate flicker
		innerRadiusMin: 150,
		innerRadiusMax: 170,

		// Outer falloff - ambient light, most flicker/fade
		outerRadiusMin: 180,
		outerRadiusMax: 200,

		// Fast flickers (dancing flame effect)
		fastFlickerCheckInterval: 150, // Check every 150ms for smoother feel
		fastFlickerChance: 0.15, // 15% chance per check (less frequent)
		fastFlickerDurationMin: 200, // Slower, more gradual flickers
		fastFlickerDurationMax: 400,

		// Breathing animation (gentle pulsing from hand movement)
		breathingSpeedMin: 2000, // 2-4 second breathing cycles
		breathingSpeedMax: 4000,
		breathingIntensity: 0.15, // 15% size variance

		// Guttering (rare flame nearly going out)
		gutterCheckInterval: 1000, // Check every second
		gutterChance: 0.05, // 5% chance per check (rare)
		gutterDurationMin: 400, // How long the dip lasts
		gutterDurationMax: 800,
		gutterIntensity: 0.5, // Shrinks to 50% of size

		// Gas can properties
		gasCanSizeInitial: 100, // Starting size in px
		gasCanSizeMin: 20, // Minimum size in px
		gasCanShrinkRate: 0.94, // 6% smaller each refuel (was 8%)
		gasCanMinSizeThreshold: 10, // Stop shrinking after 10 refuels
		gasCanOpacity: 0.6, // Base opacity
		gasCanCooldownMin: 3000, // Min 3 seconds cooldown before next spawn
		gasCanCooldownMax: 7000, // Max 7 seconds cooldown before next spawn

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

	// Gas can cooldown system (NEW - Phase 4)
	let gasCanCooldown = $state(false); // Is gas can on cooldown?
	let gasCanCooldownTimer: number | null = null;

	// ========================================================================
	// MONKEY MODE CONFIGURATION
	// ========================================================================
	const monkeyConfig = {
		// Quadrant system
		quadrantCheckInterval: 100, // Update favor every 100ms
		stateChangeIntervalMin: 3000, // 3 seconds min
		stateChangeIntervalMax: 11000, // 11 seconds max

		// Audio feedback
		feedbackSfxIntervalMin: 5000, // 5 seconds min
		feedbackSfxIntervalMax: 15000, // 15 seconds max
		feedbackSliceDurationMin: 3000, // 3 seconds
		feedbackSliceDurationMax: 5000, // 5 seconds

		// Favor thresholds
		favorBallingThreshold: 100,
		favorAngryThreshold: -100,
		favorVideoThreshold: 100,
	};

	// Monkey Mode state variables
	let monkeyModeActive = $state(false);
	let monkeyFavor = $state(0); // Internal favor tracking (updates every 100ms)
	let monkeyFavorDisplay = $state(0); // Displayed favor (updates every second)
	let monkeyFavorLastCheck = $state(0); // For tracking delta since last SFX
	let monkeyThought = $state<string>(''); // Current emotional state text

	// Quadrant tracking
	type QuadrantState = 'auspicious' | 'ominous' | 'grounded' | 'benevolent' |
	                     'eerie' | 'unfortunate' | 'subtle' | 'foolish' | 'normal';

	interface QuadrantData {
		state: QuadrantState;
		timeInState: number; // Seconds spent in current state
		stateTimer: number | null; // Interval ID for state changes
		initialDirection?: 'gain' | 'loss'; // For subtle/foolish
	}

	let quadrants = $state<{
		topLeft: QuadrantData;
		topRight: QuadrantData;
		bottomLeft: QuadrantData;
		bottomRight: QuadrantData;
	}>({
		topLeft: { state: 'normal', timeInState: 0, stateTimer: null },
		topRight: { state: 'normal', timeInState: 0, stateTimer: null },
		bottomLeft: { state: 'normal', timeInState: 0, stateTimer: null },
		bottomRight: { state: 'normal', timeInState: 0, stateTimer: null },
	});

	let currentQuadrant = $state<'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight'>('topLeft');

	// Audio/Video
	let monkeyBgMusic: HTMLAudioElement | null = null;
	let monkeyGoodSfx: HTMLAudioElement | null = null;
	let monkeyBadSfx: HTMLAudioElement | null = null;
	let feedbackSfxTimer: number | null = null;
	let favorUpdateInterval: number | null = null;
	let favorDisplayInterval: number | null = null;

	// Video reward
	let showMonkeyVideo = $state(false);
	let monkeyVideoOpacity = $state(0);
	let monkeyVideoElement: HTMLVideoElement | null = null;
	let monkeyVideoUrl = $state<string | null>(null);
	let monkeyFadeToBlack = $state(false);
	let monkeyFadeOpacity = $state(0);

	// Legacy placeholders (for backward compatibility)
	let targetQuadrant = $state<'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'>('top-right');
	let gasCanSpawning = $state(false); // Is gas can fading in? (NEW - Phase 4 Enhanced)
	let gasCanFadeDuration = $state(3000); // Random fade-in duration (3-7 seconds)

	// Candlelight multi-layered system (NEW - Phase 4 Enhanced)
	let coreRadius = $state(50); // Core bright zone
	let innerRadius = $state(160); // Main illumination
	let outerRadius = $state(225); // Ambient falloff
	let breathingAnimationFrame: number | null = null;
	let breathingPhase = $state(0); // 0-1 phase of breathing cycle
	let breathingSpeed = $state(3000); // Current cycle duration
	let fastFlickerInterval: number | null = null;
	let gutterInterval: number | null = null;
	let isGuttering = $state(false);

	// Hand-shake offset (simulates person trying to steady the lantern)
	let handShakeOffsetX = $state(0);
	let handShakeOffsetY = $state(0);
	let handShakeInterval: number | null = null;

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
				showHorrorIntro = false;
				// Start horror game systems
				startFuelDrain();
				startFlickerAnimation();
				if (horrorBgAudio) {
					// Start at random position (max 50% through track)
					const playFromRandom = () => {
						if (horrorBgAudio.duration && !isNaN(horrorBgAudio.duration)) {
							const maxStartTime = horrorBgAudio.duration * 0.5;
							const randomStartTime = Math.random() * maxStartTime;
							horrorBgAudio.currentTime = randomStartTime;
							horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMin;
							horrorBgAudio.play().catch(e => console.error('Failed to play horror music:', e));
						} else {
							// Fallback to beginning if duration unavailable
							horrorBgAudio.currentTime = 0;
							horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMin;
							horrorBgAudio.play().catch(e => console.error('Failed to play horror music:', e));
						}
					};

					if (horrorBgAudio.readyState >= 2) {
						// Audio metadata already loaded
						playFromRandom();
					} else {
						// Wait for metadata to load
						horrorBgAudio.addEventListener('loadedmetadata', playFromRandom, { once: true });
					}
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

		if (flickerAnimationFrame) {
			cancelAnimationFrame(flickerAnimationFrame);
		}
	}

	// ========================================================================
	// HORROR MODE AUDIO SYSTEM
	// ========================================================================
	function updateBgMusicVolume() {

		if (!horrorBgAudio || horrorGamePaused) return;

		// Logarithmic volume increase as fuel depletes
		// At 100% fuel: min volume, At 0% fuel: max volume
		const fuelRatio = fuelLevel / 100;
		const volumeRange = hauntedConfig.bgMusicVolumeMax - hauntedConfig.bgMusicVolumeMin;
		horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMax - (fuelRatio * volumeRange);
	}

	function startTempoVariation() {

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

		if (tempoVariationInterval) {
			clearInterval(tempoVariationInterval);
		}
	}

	// ========================================================================
	// JUMPSCARE SYSTEM
	// ========================================================================
	function startJumpscareCountdown() {

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

	/**
	 * Helper function to position an element away from the cursor
	 * @param marginPercent - Percentage of screen size to avoid around cursor (0.1 = 10%, 0.15 = 15%)
	 * @returns Object with x and y coordinates
	 */
	function getPositionAwayFromCursor(marginPercent: number): { x: number; y: number } {
		const width = typeof window !== 'undefined' ? window.innerWidth : 800;
		const height = typeof window !== 'undefined' ? window.innerHeight : 600;

		const xRange = width * marginPercent;
		const yRange = height * marginPercent;

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

		return { x, y };
	}

	function repositionGhoul() {
		const pos = getPositionAwayFromCursor(0.15); // 15% margin
		ghoulX = pos.x;
		ghoulY = pos.y;
	}

	function despawnGhoul() {

		ghoulActive = false;
		ghoulCountdown = -1;
		ghoulHoverDuration = 0;
		isHoveringGhoul = false;

		if (ghoulCountdownInterval) clearInterval(ghoulCountdownInterval);
		if (ghoulHoverCheckInterval) clearInterval(ghoulHoverCheckInterval);
		if (heartbeatAudio) heartbeatAudio.pause();
	}

	function onGhoulMouseEnter() {

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

		isHoveringGhoul = false;
		ghoulHoverDuration = 0;

		if (ghoulHoverCheckInterval) {
			clearInterval(ghoulHoverCheckInterval);
		}
	}

	function ghoulFound() {

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

		if (typeof document === 'undefined') return; // SSR guard

		// Handle haunted mode
		if (selectedThemePreset === 'haunted') {
			if (document.hidden) {
				pauseHorrorMode();
			} else {
				resumeHorrorMode();
			}
		}

		// Handle monkey mode
		if (monkeyModeActive) {
			if (document.hidden) {
				// Tab became inactive - lose all favor
				monkeyFavor = 0;
				monkeyFavorLastCheck = 0;

				// Pause quadrant timers
				stopQuadrantTimer('topLeft');
				stopQuadrantTimer('topRight');
				stopQuadrantTimer('bottomLeft');
				stopQuadrantTimer('bottomRight');

				// Pause favor tracking
				stopMonkeyFavorTracking();
				stopFeedbackSfxLoop();
				stopFavorDisplayUpdate();

				// Pause background music
				if (monkeyBgMusic) {
					monkeyBgMusic.pause();
				}
			} else {
				// Tab became active - resume
				startQuadrantTimer('topLeft');
				startQuadrantTimer('topRight');
				startQuadrantTimer('bottomLeft');
				startQuadrantTimer('bottomRight');

				startMonkeyFavorTracking();
				startFeedbackSfxLoop();
				startFavorDisplayUpdate();

				if (monkeyBgMusic) {
					monkeyBgMusic.play().catch(() => {});
				}
			}
		}
	}

	function pauseHorrorMode() {

		horrorGamePaused = true;

		// Mute/pause all audio
		if (horrorBgAudio) horrorBgAudio.volume = 0;
		if (warningAudio) warningAudio.pause();
		// All intervals check horrorGamePaused flag, so they'll freeze automatically
	}

	function resumeHorrorMode() {

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

	async function playResult() {

		if (!sfxEnabled || !resultEnabled) {
			return;
		}

		// Choose appropriate result sound based on theme
		const audioToPlay = selectedThemePreset === 'haunted'
			? horrorResultAudio
			: resultAudio;

		if (!audioToPlay) {
			return;
		}


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
			} else if (fuelLevel <= 0 && jumpscareCountdown === -1) {
				// SAFEGUARD: Only trigger jumpscare if fuel is truly depleted
				fuelLevel = 0; // Ensure it's exactly 0
				startJumpscareCountdown();
			}
		}, hauntedConfig.fuelDrainInterval);
	}

	function stopFuelDrain() {

		if (fuelDrainInterval) {
			clearInterval(fuelDrainInterval);
		}
	}

	function refuelLantern() {

		// SAFEGUARD: Cancel jumpscare countdown if it was running
		if (jumpscareCountdown >= 0) {
			clearInterval(jumpscareCountdownInterval);
			jumpscareCountdown = -1;
			// Stop warning audio
			if (warningAudio) {
				warningAudio.pause();
				warningAudio.currentTime = 0;
			}
		}

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

		// Start gas can cooldown (NEW - Phase 4)
		gasCanCooldown = true;
		const cooldownDuration = hauntedConfig.gasCanCooldownMin +
			Math.random() * (hauntedConfig.gasCanCooldownMax - hauntedConfig.gasCanCooldownMin);


		gasCanCooldownTimer = window.setTimeout(() => {
			gasCanCooldown = false;
		}, cooldownDuration);

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

	// ============================================================================
	// LANTERN FLICKER SYSTEM (NEW - Phase 4)
	// ============================================================================

	// ============================================================================
	// CANDLELIGHT SYSTEM - Multi-layered organic lighting
	// ============================================================================

	/**
	 * Start all three candlelight animation systems
	 */
	function startCandlelightSystem() {

		// Initialize radii to base values
		coreRadius = (hauntedConfig.coreRadiusMin + hauntedConfig.coreRadiusMax) / 2;
		innerRadius = (hauntedConfig.innerRadiusMin + hauntedConfig.innerRadiusMax) / 2;
		outerRadius = (hauntedConfig.outerRadiusMin + hauntedConfig.outerRadiusMax) / 2;

		// Start all systems
		startFastFlickers();
		startBreathingAnimation();
		startGutterSystem();
		startHandShake();
	}

	/**
	 * Stop all candlelight animation systems
	 */
	function stopCandlelightSystem() {

		// Stop fast flickers
		if (fastFlickerInterval) {
			clearInterval(fastFlickerInterval);
			fastFlickerInterval = null;
		}

		// Stop breathing animation
		if (breathingAnimationFrame) {
			cancelAnimationFrame(breathingAnimationFrame);
			breathingAnimationFrame = null;
		}

		// Stop gutter system
		if (gutterInterval) {
			clearInterval(gutterInterval);
			gutterInterval = null;
		}

		// Stop hand shake
		if (handShakeInterval) {
			clearInterval(handShakeInterval);
			handShakeInterval = null;
		}

		// Reset state
		isGuttering = false;
		breathingPhase = 0;
		handShakeOffsetX = 0;
		handShakeOffsetY = 0;
	}

	/**
	 * Fast flickers - Quick dancing flame effect on outer zone
	 */
	function startFastFlickers() {
		if (fastFlickerInterval) return;

		fastFlickerInterval = window.setInterval(() => {
			if (Math.random() < hauntedConfig.fastFlickerChance) {
				triggerFastFlicker();
			}
		}, hauntedConfig.fastFlickerCheckInterval);
	}

	function triggerFastFlicker() {
		// Randomly expand outer radius briefly
		const targetOuter = hauntedConfig.outerRadiusMin +
			Math.random() * (hauntedConfig.outerRadiusMax - hauntedConfig.outerRadiusMin);

		// Also flicker inner slightly
		const targetInner = hauntedConfig.innerRadiusMin +
			Math.random() * (hauntedConfig.innerRadiusMax - hauntedConfig.innerRadiusMin);

		outerRadius = targetOuter;
		innerRadius = targetInner;

		// Quick return to normal (CSS transition handles smoothing)
		const flickerDuration = hauntedConfig.fastFlickerDurationMin +
			Math.random() * (hauntedConfig.fastFlickerDurationMax - hauntedConfig.fastFlickerDurationMin);

		window.setTimeout(() => {
			// Return to breathing baseline
			const breathingMod = Math.sin(breathingPhase * Math.PI * 2) * hauntedConfig.breathingIntensity;
			const fuelMultiplier = 0.5 + (fuelLevel / 100) * 0.5;
			outerRadius = ((hauntedConfig.outerRadiusMin + hauntedConfig.outerRadiusMax) / 2) * (1 + breathingMod) * fuelMultiplier;
			innerRadius = ((hauntedConfig.innerRadiusMin + hauntedConfig.innerRadiusMax) / 2) * (1 + breathingMod) * fuelMultiplier;
		}, flickerDuration);
	}

	/**
	 * Breathing animation - Gentle continuous pulsing (sine wave)
	 */
	function startBreathingAnimation() {
		// Random breathing speed
		breathingSpeed = hauntedConfig.breathingSpeedMin +
			Math.random() * (hauntedConfig.breathingSpeedMax - hauntedConfig.breathingSpeedMin);

		let lastTime = performance.now();

		function animate(currentTime: number) {
			const deltaTime = currentTime - lastTime;
			lastTime = currentTime;

			// Update breathing phase (0-1 cycle)
			breathingPhase += deltaTime / breathingSpeed;
			if (breathingPhase >= 1) {
				breathingPhase = 0;
				// Randomize speed for next cycle
				breathingSpeed = hauntedConfig.breathingSpeedMin +
					Math.random() * (hauntedConfig.breathingSpeedMax - hauntedConfig.breathingSpeedMin);
			}

			// Apply sine wave breathing effect (unless guttering)
			if (!isGuttering) {
				const breathingMod = Math.sin(breathingPhase * Math.PI * 2) * hauntedConfig.breathingIntensity;
				// Fuel multiplier - light shrinks as fuel depletes (50% to 100% size based on fuel)
				const fuelMultiplier = 0.5 + (fuelLevel / 100) * 0.5;

				// Core barely breathes
				coreRadius = ((hauntedConfig.coreRadiusMin + hauntedConfig.coreRadiusMax) / 2) * (1 + breathingMod * 0.2) * fuelMultiplier;

				// Inner and outer breathe more
				innerRadius = ((hauntedConfig.innerRadiusMin + hauntedConfig.innerRadiusMax) / 2) * (1 + breathingMod) * fuelMultiplier;
				outerRadius = ((hauntedConfig.outerRadiusMin + hauntedConfig.outerRadiusMax) / 2) * (1 + breathingMod) * fuelMultiplier;
			}

			breathingAnimationFrame = requestAnimationFrame(animate);
		}

		breathingAnimationFrame = requestAnimationFrame(animate);
	}

	/**
	 * Guttering system - Rare flame nearly going out
	 */
	function startGutterSystem() {
		if (gutterInterval) return;

		gutterInterval = window.setInterval(() => {
			if (!isGuttering && Math.random() < hauntedConfig.gutterChance) {
				triggerGutter();
			}
		}, hauntedConfig.gutterCheckInterval);
	}

	function triggerGutter() {
		isGuttering = true;

		// Shrink all zones dramatically
		const gutterMod = hauntedConfig.gutterIntensity;
		coreRadius = ((hauntedConfig.coreRadiusMin + hauntedConfig.coreRadiusMax) / 2) * gutterMod;
		innerRadius = ((hauntedConfig.innerRadiusMin + hauntedConfig.innerRadiusMax) / 2) * gutterMod;
		outerRadius = ((hauntedConfig.outerRadiusMin + hauntedConfig.outerRadiusMax) / 2) * gutterMod;

		// Recover after duration
		const gutterDuration = hauntedConfig.gutterDurationMin +
			Math.random() * (hauntedConfig.gutterDurationMax - hauntedConfig.gutterDurationMin);

		window.setTimeout(() => {
			isGuttering = false;
			// Breathing animation will take over again
		}, gutterDuration);
	}

	/**
	 * Hand shake - Simulates person trying to steady the lantern
	 * Slight random movements to make it feel more alive
	 */
	function startHandShake() {
		if (handShakeInterval) return;

		handShakeInterval = window.setInterval(() => {
			// Small random offsets (±3px to ±8px range)
			const maxOffset = 8;
			handShakeOffsetX = (Math.random() - 0.5) * maxOffset;
			handShakeOffsetY = (Math.random() - 0.5) * maxOffset;
		}, 150); // Update every 150ms for subtle movement
	}

	function repositionGasCan() {
		const pos = getPositionAwayFromCursor(0.1); // 10% margin
		gasCanX = pos.x;
		gasCanY = pos.y;

		// Trigger fade-in animation with random duration (3-7 seconds)
		gasCanFadeDuration = 3000 + Math.random() * 4000; // Random between 3000-7000ms
		gasCanSpawning = true;
		setTimeout(() => {
			gasCanSpawning = false;
		}, gasCanFadeDuration);
	}

	// Preload center images
	let penguinImage: HTMLImageElement | null = null;
	let handsImage: HTMLImageElement | null = null;
	if (typeof window !== 'undefined') {
		penguinImage = new Image();
		penguinImage.src = penguinImg;
		// handsImage will be set when horror assets are loaded
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

	async function applyThemePreset(preset: 'penguin' | 'haunted' | 'monkey' | 'user') {
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
			// Set wheel click sound for penguin theme
			wheelClickSound = 'rimshot';
			// Set result sound for penguin theme
			resultSound = 'alien';
			loadResultSound();

		} else if (preset === 'haunted') {

			// Load horror assets first
			if (!horrorAssetsLoaded) {
				await loadHorrorAssets();
			}

			// Only apply theme if assets loaded successfully
			if (horrorAssetsLoaded) {
				// Haunted Mode (Black & Red)
				primaryColor = '#8B0000'; // Dark red
				secondaryColor = '#000000'; // Black
				wheelBackground = '#1a0000'; // Very dark red
				selectedFont = 'Georgia';
				wheelColors = ['#8B0000', '#FF0000', '#DC143C', '#B22222', '#A52A2A'];
				backgroundWallpaper = 'creepy.jpg';
				wheelCenterImage = 'hands.jpg';
				// Set wheel click sound for haunted theme
				wheelClickSound = 'wood_wheel';
			} else {
				// Loading failed, revert to penguin
				selectedThemePreset = 'penguin';
				applyThemePreset('penguin');
				return;
			}

		} else if (preset === 'monkey') {

			// Load monkey assets first
			if (!monkeyAssetsLoaded) {
				await loadMonkeyAssets();
			}

			if (monkeyAssetsLoaded) {
				// Monkey Mode theme
				primaryColor = '#32CD32'; // Lime green
				secondaryColor = '#8B4513'; // Saddle brown
				wheelBackground = '#2F4F2F'; // Dark green
				selectedFont = 'Comic Sans MS, cursive';
				wheelColors = ['#8B4513', '#A0522D', '#CD853F', '#DEB887', '#D2691E'];
				backgroundWallpaper = 'custom'; // Use monkey_mode_bg.webp
				wheelCenterImage = 'custom'; // Use monkey_mode_center.webp
				wheelClickSound = 'custom'; // Will use monkey_mode_wheel.ogg
				resultSound = 'monkey'; // Default monkey sound
				loadResultSound(); // Load the monkey result sound

				// Set custom assets
				customWallpaperDataUrl = monkeyBackgroundUrl;
				customCenterImageDataUrl = monkeyCenterImageUrl;
				customWheelClickDataUrl = monkeyWheelClickUrl;

				// Mark monkey mode as active
				monkeyModeActive = true;

				// Start background music if available
				if (monkeyBgMusic) {
					monkeyBgMusic.currentTime = 0;
					monkeyBgMusic.play().catch(() => {});
				}
			} else {
				selectedThemePreset = 'penguin';
				applyThemePreset('penguin');
				return;
			}

		} else if (preset === 'user') {

			// User Theme - Load from localStorage or default to penguin style
			loadUserTheme();
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
				const fontSize = Math.max(12, Math.floor(radius * 0.067)); // Scale with radius
				const textRadius = radius - (radius * 0.083); // Scale text position
				ctx.font = `bold ${fontSize}px Arial`;
				ctx.fillText(movies[i], textRadius, fontSize / 3);
				ctx.restore();
			}
		}

		// Draw center circle with penguin image (scaled dynamically)
		const centerRadius = radius * 0.25; // 25% of wheel radius
		ctx.beginPath();
		ctx.arc(centerX, centerY, centerRadius, 0, 2 * Math.PI);
		ctx.fillStyle = '#ffffff';
		ctx.fill();
		ctx.strokeStyle = primaryColor;
		ctx.lineWidth = Math.max(2, radius * 0.01); // Scale line width
		ctx.stroke();

		// Draw center image if selected
		if (wheelCenterImage === 'custom' && customCenterImageDataUrl) {
			// Draw custom uploaded image
			const img = new Image();
			img.src = customCenterImageDataUrl;
			if (img.complete) {
				const imgRadius = centerRadius - 2;
				ctx.save();
				ctx.beginPath();
				ctx.arc(centerX, centerY, imgRadius, 0, 2 * Math.PI);
				ctx.clip();
				ctx.drawImage(img, centerX - imgRadius, centerY - imgRadius, imgRadius * 2, imgRadius * 2);
				ctx.restore();
			}
		} else if (wheelCenterImage === 'penguin.jpg' && penguinImage && penguinImage.complete) {
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

		// Play monkey video immediately if favor >= 100
		if (shouldPlayMonkeyVideo()) {
			playMonkeyVideoReward();
		}

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

			// Play result sound when wheel stops
			// (playResult now handles horror mode automatically)
			playResult();

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
				wheelColors = theme.wheelColors || wheelColors;
				backgroundWallpaper = theme.backgroundWallpaper || backgroundWallpaper;
				wheelCenterImage = theme.wheelCenterImage || wheelCenterImage;
			} catch (e) {
				console.error('Failed to load theme:', e);
			}
		}

		// Force non-haunted theme on mobile devices
		if (isMobileDevice() && selectedThemePreset === 'haunted') {
			selectedThemePreset = 'penguin';
			applyThemePreset('penguin');
		}
	}

	function isMobileDevice(): boolean {
		if (typeof window === 'undefined') return false;
		// Check for touch device and small screen
		return (
			('ontouchstart' in window || navigator.maxTouchPoints > 0) &&
			window.innerWidth <= 768
		);
	}

	function saveUserTheme() {

		// If we're on Custom Theme, save the current active values to custom variables first
		if (selectedThemePreset === 'user') {

			customPrimaryColor = primaryColor;
			customSecondaryColor = secondaryColor;
			customWheelBackground = wheelBackground;
			customSelectedFont = selectedFont;
			customWheelColors = [...wheelColors];
			customBackgroundWallpaper = backgroundWallpaper;
			customWheelCenterImage = wheelCenterImage;
			customWheelClickSound = wheelClickSound;

		} else {
		}

		// Always save from custom* variables to localStorage
		const userTheme = {
			primaryColor: customPrimaryColor,
			secondaryColor: customSecondaryColor,
			wheelBackground: customWheelBackground,
			selectedFont: customSelectedFont,
			wheelColors: customWheelColors,
			backgroundWallpaper: customBackgroundWallpaper,
			wheelCenterImage: customWheelCenterImage,
			wheelClickSound: customWheelClickSound,
			resultSound: customResultSound,
			customWallpaperDataUrl,
			customCenterImageDataUrl,
			customWheelClickDataUrl,
			customResultDataUrl
		};
		localStorage.setItem('slippymud_user_theme', JSON.stringify(userTheme));
	}

	function loadUserTheme() {
		const saved = localStorage.getItem('slippymud_user_theme');
		if (saved) {
			try {
				const userTheme = JSON.parse(saved);
				// Load into custom* variables first (using saved values, NO defaults)
				customPrimaryColor = userTheme.primaryColor;
				customSecondaryColor = userTheme.secondaryColor;
				customWheelBackground = userTheme.wheelBackground;
				customSelectedFont = userTheme.selectedFont;
				customWheelColors = userTheme.wheelColors;
				customBackgroundWallpaper = userTheme.backgroundWallpaper;
				customWheelCenterImage = userTheme.wheelCenterImage;
				customWheelClickSound = userTheme.wheelClickSound;
				customResultSound = userTheme.resultSound || 'alien';  // Default to alien if not saved
				customWallpaperDataUrl = userTheme.customWallpaperDataUrl || null;
				customCenterImageDataUrl = userTheme.customCenterImageDataUrl || null;
				customWheelClickDataUrl = userTheme.customWheelClickDataUrl || null;
				customResultDataUrl = userTheme.customResultDataUrl || null;

				// Copy to active variables
				primaryColor = customPrimaryColor;
				secondaryColor = customSecondaryColor;
				wheelBackground = customWheelBackground;
				selectedFont = customSelectedFont;
				wheelColors = [...customWheelColors];
				backgroundWallpaper = customBackgroundWallpaper;
				wheelCenterImage = customWheelCenterImage;
				wheelClickSound = customWheelClickSound;

			} catch (e) {
				console.error('Failed to load custom theme:', e);
				// If loading fails, keep custom* variables as-is (they're already initialized)
				// Just copy them to active variables
				primaryColor = customPrimaryColor;
				secondaryColor = customSecondaryColor;
				wheelBackground = customWheelBackground;
				selectedFont = customSelectedFont;
				wheelColors = [...customWheelColors];
				backgroundWallpaper = customBackgroundWallpaper;
				wheelCenterImage = customWheelCenterImage;
				wheelClickSound = customWheelClickSound;
			}
		} else {
			// FIRST TIME ONLY: No saved custom theme exists yet
			// Custom* variables are already initialized to penguin-like defaults at the top of the file
			// Just copy them to active variables and save to localStorage
			primaryColor = customPrimaryColor;
			secondaryColor = customSecondaryColor;
			wheelBackground = customWheelBackground;
			selectedFont = customSelectedFont;
			wheelColors = [...customWheelColors];
			backgroundWallpaper = customBackgroundWallpaper;
			wheelCenterImage = customWheelCenterImage;
			wheelClickSound = customWheelClickSound;

			// Save the initial custom theme to localStorage so it never resets again
			saveUserTheme();
		}
	}

	// ============================================================================
	// FILE UPLOAD HANDLERS
	// ============================================================================

	async function handleWallpaperUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		// Validate file type
		if (!file.type.startsWith('image/')) {
			alert('Please upload an image file');
			return;
		}

		// Validate file size (limit to 2MB for localStorage)
		if (file.size > 2 * 1024 * 1024) {
			alert('Image too large. Please use an image under 2MB');
			return;
		}

		// Convert to data URL
		const reader = new FileReader();
		reader.onload = (e) => {
			customWallpaperDataUrl = e.target?.result as string;
			backgroundWallpaper = 'custom';
			if (selectedThemePreset === 'user') {
				saveUserTheme();
			}
		};
		reader.readAsDataURL(file);
	}

	async function handleCenterImageUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		// Validate file type
		if (!file.type.startsWith('image/')) {
			alert('Please upload an image file');
			return;
		}

		// Validate file size (limit to 2MB for localStorage)
		if (file.size > 2 * 1024 * 1024) {
			alert('Image too large. Please use an image under 2MB');
			return;
		}

		// Convert to data URL
		const reader = new FileReader();
		reader.onload = (e) => {
			customCenterImageDataUrl = e.target?.result as string;
			wheelCenterImage = 'custom';
			if (selectedThemePreset === 'user') {
				saveUserTheme();
			}
			drawWheel(); // Redraw with new image
		};
		reader.readAsDataURL(file);
	}

	async function handleWheelClickUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		// Validate file type (accept common audio formats)
		const validTypes = ['audio/mpeg', 'audio/mp3', 'audio/wav', 'audio/ogg', 'audio/x-wav', 'audio/webm'];
		if (!validTypes.includes(file.type) && !file.name.match(/\.(mp3|wav|ogg|webm)$/i)) {
			alert('Please upload an audio file (.mp3, .wav, .ogg, or .webm)');
			return;
		}

		// Validate file size (limit to 500KB for localStorage)
		if (file.size > 500 * 1024) {
			alert('Audio too large. Please use a file under 500KB');
			return;
		}

		// Convert to data URL
		const reader = new FileReader();
		reader.onload = async (e) => {
			const dataUrl = e.target?.result as string;

			// Validate audio duration (must be ≤1 second)
			const audio = new Audio();
			audio.src = dataUrl;

			// Wait for audio metadata to load
			await new Promise<void>((resolve, reject) => {
				audio.onloadedmetadata = () => resolve();
				audio.onerror = () => reject(new Error('Failed to load audio'));
			});

			if (audio.duration > 12.0) {
				alert(`Audio too long (${audio.duration.toFixed(2)}s). Please use an audio file that is 12 seconds or shorter.`);
				input.value = ''; // Clear the file input
				return;
			}

			// Duration is valid, proceed
			customWheelClickDataUrl = dataUrl;
			wheelClickSound = 'custom';
			if (selectedThemePreset === 'user') {
				saveUserTheme();
			}
			// Reinitialize audio pool with custom sound
			initWheelClickAudioPool();
		};
		reader.readAsDataURL(file);
	}

	async function handleResultUpload(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		// Validate file type (accept common audio formats)
		const validTypes = ['audio/mpeg', 'audio/mp3', 'audio/wav', 'audio/ogg', 'audio/x-wav', 'audio/webm'];
		if (!validTypes.includes(file.type) && !file.name.match(/\.(mp3|wav|ogg|webm)$/i)) {
			alert('Please upload an audio file (.mp3, .wav, .ogg, or .webm)');
			return;
		}

		// Validate file size (limit to 500KB for localStorage)
		if (file.size > 500 * 1024) {
			alert('Audio too large. Please use a file under 500KB');
			return;
		}

		// Convert to data URL
		const reader = new FileReader();
		reader.onload = async (e) => {
			const dataUrl = e.target?.result as string;

			// Validate audio duration (must be ≤1 second)
			const audio = new Audio();
			audio.src = dataUrl;

			// Wait for audio metadata to load
			await new Promise<void>((resolve, reject) => {
				audio.onloadedmetadata = () => resolve();
				audio.onerror = () => reject(new Error('Failed to load audio'));
			});

			if (audio.duration > 1.0) {
				alert(`Audio too long (${audio.duration.toFixed(2)}s). Please use an audio file that is 1 second or shorter.`);
				input.value = ''; // Clear the file input
				return;
			}

			// Duration is valid, proceed
			customResultDataUrl = dataUrl;
			if (selectedThemePreset === 'user') {
				saveUserTheme();
			}
			// Reinitialize victory audio
			if (resultAudio) {
				resultAudio.src = customResultDataUrl;
				resultAudio.load();
			}
		};
		reader.readAsDataURL(file);
	}

	// ============================================================================
	// IMPORT/EXPORT THEME
	// ============================================================================

	function exportTheme() {
		const themeData = {
			version: "1.0",
			timestamp: new Date().toISOString(),
			theme: {
				primaryColor: customPrimaryColor,
				secondaryColor: customSecondaryColor,
				wheelBackground: customWheelBackground,
				selectedFont: customSelectedFont,
				wheelColors: [...customWheelColors],
				backgroundWallpaper: customBackgroundWallpaper,
				wheelCenterImage: customWheelCenterImage,
				wheelClickSound: customWheelClickSound,
				resultSound: customResultSound,
				customAssets: {
					wallpaper: customWallpaperDataUrl,
					centerImage: customCenterImageDataUrl,
					wheelClick: customWheelClickDataUrl,
					result: customResultDataUrl
				}
			}
		};

		const jsonString = JSON.stringify(themeData, null, 2);
		const blob = new Blob([jsonString], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const link = document.createElement('a');
		link.href = url;
		link.download = `slippymud-theme-${Date.now()}.json`;
		link.click();
		URL.revokeObjectURL(url);
	}

	async function importTheme(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		if (!file.name.endsWith('.json')) {
			alert('Please upload a valid JSON theme file');
			return;
		}

		if (file.size > 10 * 1024 * 1024) {
			alert('Theme file too large. Maximum size is 10MB');
			return;
		}

		try {
			const text = await file.text();
			const themeData = JSON.parse(text);

			if (!themeData.version || !themeData.theme) {
				throw new Error('Invalid theme file structure');
			}

			const { theme } = themeData;

			// Apply settings to custom theme
			customPrimaryColor = theme.primaryColor || customPrimaryColor;
			customSecondaryColor = theme.secondaryColor || customSecondaryColor;
			customWheelBackground = theme.wheelBackground || customWheelBackground;
			customSelectedFont = theme.selectedFont || customSelectedFont;
			customWheelColors = theme.wheelColors || customWheelColors;
			customBackgroundWallpaper = theme.backgroundWallpaper || customBackgroundWallpaper;
			customWheelCenterImage = theme.wheelCenterImage || customWheelCenterImage;
			customWheelClickSound = theme.wheelClickSound || customWheelClickSound;
			customResultSound = theme.resultSound || customResultSound;

			// Import custom assets if present
			if (theme.customAssets) {
				customWallpaperDataUrl = theme.customAssets.wallpaper || null;
				customCenterImageDataUrl = theme.customAssets.centerImage || null;
				customWheelClickDataUrl = theme.customAssets.wheelClick || null;
				customResultDataUrl = theme.customAssets.result || null;
			}

			// Apply if on user preset
			if (selectedThemePreset === 'user') {
				loadUserTheme();
			}

			saveUserTheme();
			alert('Theme imported successfully!');
			drawWheel();

		} catch (error) {
			console.error('Failed to import theme:', error);
			alert('Failed to import theme. Please ensure the file is a valid SlippyMudWheel theme export.');
		}

		input.value = '';
	}

	// ============================================================================
	// LAZY LOADING SYSTEM
	// ============================================================================

	async function loadHorrorAssets() {
		if (horrorAssetsLoaded || horrorAssetsLoading) return;

		horrorAssetsLoading = true;

		try {
			const [
				horrorBgModule,
				creepyBgModule,
				handsImgModule,
				fnafJumpscareGifModule,
				fnafJumpscareSfxModule,
				fnafHallwaySfxModule,
				refillSfxModule,
				horrorStartSfxModule,
				horrorResultSfxModule,
				mar01Module,
				mar02Module,
				mar03Module,
				mar04Module,
				mar05Module,
				ghoulSfxModule,
				heartbeatSfxModule,
			] = await Promise.all([
				import('$lib/horror_bg.ogg'),
				import('$lib/creepy.jpg'),
				import('$lib/hands.jpg'),
				import('$lib/fnaf_jumpscare.gif'),
				import('$lib/fnaf2_puppet_js.ogg'),
				import('$lib/fnaf2_hallway_sfx.ogg'),
				import('$lib/refill.ogg'),
				import('$lib/horror_start.ogg'),
				import('$lib/horror_result.ogg'),
				import('$lib/Mar01.ogg'),
				import('$lib/Mar02.ogg'),
				import('$lib/Mar03.ogg'),
				import('$lib/Mar04.ogg'),
				import('$lib/Mar05.ogg'),
				import('$lib/ghoul.ogg'),
				import('$lib/heartbeat.ogg'),
			]);

			// Initialize audio instances
			horrorBgAudio = new Audio(horrorBgModule.default);
			horrorBgAudio.loop = false; // Custom loop handling
			horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMin;

			// Add custom loop handler - loop from beginning after first play
			horrorBgAudio.addEventListener('ended', () => {
				if (selectedThemePreset === 'haunted') {
					horrorBgAudio.currentTime = 0;
					horrorBgAudio.play().catch(() => {});
				}
			});

			warningAudio = new Audio(fnafHallwaySfxModule.default);
			warningAudio.loop = false;
			warningAudio.volume = hauntedConfig.warningVolume;

			jumpscareAudio = new Audio(fnafJumpscareSfxModule.default);

			refillAudio = new Audio(refillSfxModule.default);
			refillAudio.volume = hauntedConfig.refillVolume;

			marionetteScreamAudios = [
				new Audio(mar01Module.default),
				new Audio(mar02Module.default),
				new Audio(mar03Module.default),
				new Audio(mar04Module.default),
				new Audio(mar05Module.default)
			];

			horrorStartAudio = new Audio(horrorStartSfxModule.default);
			horrorResultAudio = new Audio(horrorResultSfxModule.default);

			heartbeatAudio = new Audio(heartbeatSfxModule.default);
			heartbeatAudio.volume = hauntedConfig.warningVolume;
			heartbeatAudio.loop = true;

			ghoulJumpscareAudio = new Audio(ghoulSfxModule.default);

			// Store image URLs
			horrorBackgroundUrl = creepyBgModule.default;
			horrorCenterImageUrl = handsImgModule.default;
			horrorJumpscareGifUrl = fnafJumpscareGifModule.default;

			// Preload hands image
			if (typeof window !== 'undefined') {
				handsImage = new Image();
				handsImage.src = handsImgModule.default;
			}

			horrorAssetsLoaded = true;
		} catch (error) {
			console.error('Failed to load horror assets:', error);
			alert('Failed to load horror mode assets. Please check your connection and try again.');
			selectedThemePreset = 'penguin';
		} finally {
			horrorAssetsLoading = false;
		}
	}

	async function loadMonkeyAssets() {
		if (monkeyAssetsLoaded || monkeyAssetsLoading) return;

		monkeyAssetsLoading = true;

		try {
			const imports = await Promise.all([
				import('$lib/monkey_mode_bg.webp'),
				import('$lib/monkey_mode_center.webp'),
				import('$lib/monkey_mode_wheel.ogg'),
				import('$lib/monkey_good.ogg'),
				import('$lib/monkey_bad.ogg'),
				import('$lib/monkey_rizz_edit_hq.webm'),
			]);

			const [bgModule, centerModule, wheelModule, goodModule, badModule, videoModule] = imports;

			// Store URLs
			monkeyBackgroundUrl = bgModule.default;
			monkeyCenterImageUrl = centerModule.default;
			monkeyWheelClickUrl = wheelModule.default;
			monkeyVideoUrl = videoModule.default;

			// Initialize audio with -24dB volume (approximately 6.25% volume)
			monkeyGoodSfx = new Audio(goodModule.default);
			monkeyGoodSfx.volume = 0.0625;
			monkeyBadSfx = new Audio(badModule.default);
			monkeyBadSfx.volume = 0.0625;

			// Note: Background music removed for now - add monkey_mode_music.ogg if desired
			// and uncomment the loading code

			monkeyAssetsLoaded = true;
		} catch (error) {
			console.error('Failed to load monkey assets:', error);
			alert('Failed to load monkey mode assets.');
			selectedThemePreset = 'penguin';
		} finally {
			monkeyAssetsLoading = false;
		}
	}

	// ============================================================================
	// MONKEY MODE - Quadrant State System
	// ============================================================================

	function getRandomQuadrantState(): QuadrantState {
		const rand = Math.random();

		if (rand < 0.05) return 'auspicious';
		if (rand < 0.10) return 'ominous';
		if (rand < 0.15) return 'grounded';
		if (rand < 0.20) return 'benevolent';
		if (rand < 0.25) return 'eerie';
		if (rand < 0.30) return 'unfortunate';
		if (rand < 0.35) return 'subtle';
		if (rand < 0.40) return 'foolish';
		return 'normal'; // 60% chance
	}

	function changeQuadrantState(quadrant: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight') {
		const newState = getRandomQuadrantState();
		quadrants[quadrant].state = newState;
		quadrants[quadrant].timeInState = 0;

		// Set initial direction for subtle/foolish
		if (newState === 'subtle' || newState === 'foolish') {
			quadrants[quadrant].initialDirection = Math.random() > 0.5 ? 'gain' : 'loss';
		}
	}

	function startQuadrantTimer(quadrant: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight') {
		// Random interval between 3-11 seconds
		const interval = monkeyConfig.stateChangeIntervalMin +
		                 Math.random() * (monkeyConfig.stateChangeIntervalMax - monkeyConfig.stateChangeIntervalMin);

		quadrants[quadrant].stateTimer = window.setTimeout(() => {
			changeQuadrantState(quadrant);
			startQuadrantTimer(quadrant); // Restart timer
		}, interval);
	}

	function stopQuadrantTimer(quadrant: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight') {
		if (quadrants[quadrant].stateTimer) {
			clearTimeout(quadrants[quadrant].stateTimer);
			quadrants[quadrant].stateTimer = null;
		}
	}

	function calculateFavorDelta(quadrant: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight'): number {
		const q = quadrants[quadrant];
		const t = q.timeInState;

		switch (q.state) {
			case 'auspicious':
				return 5;

			case 'ominous':
				return -5;

			case 'grounded':
				// Gain nothing, but +1 more each second
				return t; // 0s=0, 1s=1, 2s=2, 3s=3...

			case 'benevolent':
				// +5 favor, but -1 more each second (can go negative)
				return 5 - t;

			case 'eerie':
				// Lose nothing, but -1 more each second
				return -t; // 0s=0, 1s=-1, 2s=-2...

			case 'unfortunate':
				// -10 favor, but +3 less each second (can go positive)
				return -10 + (t * 3);

			case 'subtle':
				// Start ±5, get ∓1 each second (flips over time)
				if (q.initialDirection === 'gain') {
					return 5 - t;
				} else {
					return -5 + t;
				}

			case 'foolish':
				// Start ±5, get ±1 each second (accelerates)
				if (q.initialDirection === 'gain') {
					return 5 + t;
				} else {
					return -5 - t;
				}

			case 'normal':
			default:
				// +1 favor every 5 seconds
				return t % 5 === 0 && t > 0 ? 1 : 0;
		}
	}

	function updateMonkeyFavor() {
		if (!monkeyModeActive) return;

		// Increment time in current quadrant
		quadrants[currentQuadrant].timeInState += 0.1; // 100ms = 0.1 seconds

		// Calculate and apply favor delta
		const delta = calculateFavorDelta(currentQuadrant);
		monkeyFavor += delta;
	}

	function startMonkeyFavorTracking() {
		if (favorUpdateInterval) return;

		favorUpdateInterval = window.setInterval(() => {
			updateMonkeyFavor();
		}, monkeyConfig.quadrantCheckInterval);
	}

	function stopMonkeyFavorTracking() {
		if (favorUpdateInterval) {
			clearInterval(favorUpdateInterval);
			favorUpdateInterval = null;
		}
	}

	// Update displayed favor every second (whole number only)
	function startFavorDisplayUpdate() {
		if (favorDisplayInterval) return;

		favorDisplayInterval = window.setInterval(() => {
			monkeyFavorDisplay = Math.round(monkeyFavor);
		}, 1000); // Update every 1 second
	}

	function stopFavorDisplayUpdate() {
		if (favorDisplayInterval) {
			clearInterval(favorDisplayInterval);
			favorDisplayInterval = null;
		}
	}

	// ============================================================================
	// MONKEY MODE - Audio Feedback System
	// ============================================================================

	function playMonkeyFeedbackSfx() {
		if (!monkeyModeActive) return;

		// Calculate delta since last check
		const delta = monkeyFavor - monkeyFavorLastCheck;
		monkeyFavorLastCheck = monkeyFavor;

		// Determine which SFX to play
		const sfx = delta > 0 ? monkeyGoodSfx : monkeyBadSfx;
		if (!sfx) return;

		// Random slice duration (3-5 seconds)
		const duration = monkeyConfig.feedbackSliceDurationMin +
		                 Math.random() * (monkeyConfig.feedbackSliceDurationMax - monkeyConfig.feedbackSliceDurationMin);

		// Reset to beginning and play
		sfx.currentTime = 0;
		sfx.volume = 0;
		sfx.play().catch(() => {});

		// Fade in over 0.5 seconds
		let fadeInProgress = 0;
		const fadeInInterval = setInterval(() => {
			fadeInProgress += 50;
			sfx.volume = Math.min(1, fadeInProgress / 500);
			if (fadeInProgress >= 500) {
				clearInterval(fadeInInterval);
			}
		}, 50);

		// Fade out and stop after duration
		setTimeout(() => {
			let fadeOutProgress = 0;
			const fadeOutInterval = setInterval(() => {
				fadeOutProgress += 50;
				sfx.volume = Math.max(0, 1 - fadeOutProgress / 500);
				if (fadeOutProgress >= 500) {
					clearInterval(fadeOutInterval);
					sfx.pause();
				}
			}, 50);
		}, duration - 500);
	}

	function startFeedbackSfxLoop() {
		if (feedbackSfxTimer) return;

		function scheduleNext() {
			const interval = monkeyConfig.feedbackSfxIntervalMin +
			                 Math.random() * (monkeyConfig.feedbackSfxIntervalMax - monkeyConfig.feedbackSfxIntervalMin);

			feedbackSfxTimer = window.setTimeout(() => {
				playMonkeyFeedbackSfx();
				scheduleNext();
			}, interval);
		}

		scheduleNext();
	}

	function stopFeedbackSfxLoop() {
		if (feedbackSfxTimer) {
			clearTimeout(feedbackSfxTimer);
			feedbackSfxTimer = null;
		}

		// Stop any playing audio
		if (monkeyGoodSfx) monkeyGoodSfx.pause();
		if (monkeyBadSfx) monkeyBadSfx.pause();
	}

	// ============================================================================
	// MONKEY MODE - Mouse Tracking & Quadrant Detection
	// ============================================================================

	function updateCurrentQuadrant(e: MouseEvent) {
		if (!monkeyModeActive) return;
		if (typeof window === 'undefined') return; // SSR guard

		const centerX = window.innerWidth / 2;
		const centerY = window.innerHeight / 2;

		let newQuadrant: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight';

		if (e.clientX < centerX && e.clientY < centerY) {
			newQuadrant = 'topLeft';
		} else if (e.clientX >= centerX && e.clientY < centerY) {
			newQuadrant = 'topRight';
		} else if (e.clientX < centerX && e.clientY >= centerY) {
			newQuadrant = 'bottomLeft';
		} else {
			newQuadrant = 'bottomRight';
		}

		// Reset time counter when switching quadrants
		if (newQuadrant !== currentQuadrant) {
			currentQuadrant = newQuadrant;
			quadrants[currentQuadrant].timeInState = 0;
		}
	}

	// ============================================================================
	// MONKEY MODE - Emotional State Display
	// ============================================================================

	function updateMonkeyThought() {
		if (monkeyFavor > monkeyConfig.favorBallingThreshold) {
			monkeyThought = 'balling';
		} else if (monkeyFavor > 0) {
			monkeyThought = 'happy';
		} else if (monkeyFavor < monkeyConfig.favorAngryThreshold) {
			monkeyThought = 'angry';
		} else if (monkeyFavor < 0) {
			monkeyThought = 'sad';
		} else {
			monkeyThought = 'neutral';
		}
	}

	// $effect to update monkey thoughts when favor changes
	$effect(() => {
		// SSR guard - only run in browser
		if (typeof window === 'undefined') return;

		if (monkeyModeActive) {
			updateMonkeyThought();
		}
	});

	// ============================================================================
	// MONKEY MODE - Video Reward System
	// ============================================================================

	function shouldPlayMonkeyVideo(): boolean {
		return monkeyModeActive && monkeyFavor >= monkeyConfig.favorVideoThreshold;
	}

	async function playMonkeyVideoReward() {
		if (!monkeyVideoUrl) return;

		showMonkeyVideo = true;

		// Mute all audio on the site while video plays
		if (resultAudio) resultAudio.muted = true;
		if (wheelClickAudioPool) {
			wheelClickAudioPool.forEach(audio => audio.muted = true);
		}
		if (monkeyGoodSfx) monkeyGoodSfx.muted = true;
		if (monkeyBadSfx) monkeyBadSfx.muted = true;
		if (monkeyBgMusic) monkeyBgMusic.muted = true;

		// Wait for DOM to update and video element to be rendered
		await tick();

		if (!monkeyVideoElement) {
			console.error('Video element not found after tick');
			return;
		}

		// Fade in video
		monkeyVideoOpacity = 0;
		const fadeInStart = Date.now();
		const fadeInDuration = 1000; // 1 second fade in

		const fadeInInterval = setInterval(() => {
			const elapsed = Date.now() - fadeInStart;
			monkeyVideoOpacity = Math.min(1, elapsed / fadeInDuration);

			if (elapsed >= fadeInDuration) {
				clearInterval(fadeInInterval);
			}
		}, 16);

		// Start playing video
		monkeyVideoElement.currentTime = 0;

		// Set playback speed based on favor level with nightcore effect
		if (monkeyFavor >= 1000) {
			monkeyVideoElement.preservesPitch = false; // Allow pitch to change (nightcore effect)
			monkeyVideoElement.playbackRate = 1.5;
		} else {
			monkeyVideoElement.preservesPitch = true; // Normal pitch
			monkeyVideoElement.playbackRate = 1.0;
		}

		await monkeyVideoElement.play();

		// Wait for video to near end (last 2 seconds)
		const videoDuration = monkeyVideoElement.duration;
		const fadeOutStartTime = videoDuration - 2;

		monkeyVideoElement.addEventListener('timeupdate', function handleTimeUpdate() {
			if (monkeyVideoElement!.currentTime >= fadeOutStartTime) {
				// Start fade out
				const fadeOutStart = Date.now();
				const fadeOutDuration = 2000; // 2 second fade out

				const fadeOutInterval = setInterval(() => {
					const elapsed = Date.now() - fadeOutStart;
					monkeyVideoOpacity = Math.max(0, 1 - elapsed / fadeOutDuration);

					if (elapsed >= fadeOutDuration) {
						clearInterval(fadeOutInterval);
						showMonkeyVideo = false;
						monkeyVideoElement!.pause();

						// Unmute all audio after video ends
						if (resultAudio) resultAudio.muted = false;
						if (wheelClickAudioPool) {
							wheelClickAudioPool.forEach(audio => audio.muted = false);
						}
						if (monkeyGoodSfx) monkeyGoodSfx.muted = false;
						if (monkeyBadSfx) monkeyBadSfx.muted = false;
						if (monkeyBgMusic) monkeyBgMusic.muted = false;
					}
				}, 16);

				monkeyVideoElement!.removeEventListener('timeupdate', handleTimeUpdate);
			}
		});
	}

	// ============================================================================
	// MONKEY MODE - Lifecycle Management
	// ============================================================================

	// $effect to start/stop monkey mode systems
	$effect(() => {
		// SSR guard - only run in browser
		if (typeof window === 'undefined') return;

		if (selectedThemePreset === 'monkey' && monkeyModeActive) {
			// Start all systems
			console.log('Monkey mode activated');

			// Reset state
			monkeyFavor = 0;
			monkeyFavorLastCheck = 0;
			monkeyThought = '';

			// Initialize all quadrants to random states
			changeQuadrantState('topLeft');
			changeQuadrantState('topRight');
			changeQuadrantState('bottomLeft');
			changeQuadrantState('bottomRight');

			// Start timers
			startQuadrantTimer('topLeft');
			startQuadrantTimer('topRight');
			startQuadrantTimer('bottomLeft');
			startQuadrantTimer('bottomRight');

			startMonkeyFavorTracking();
			startFeedbackSfxLoop();
			startFavorDisplayUpdate();

		} else if (selectedThemePreset !== 'monkey' && monkeyModeActive) {
			// Stop all systems
			console.log('Monkey mode deactivated');

			monkeyModeActive = false;

			// Stop timers
			stopQuadrantTimer('topLeft');
			stopQuadrantTimer('topRight');
			stopQuadrantTimer('bottomLeft');
			stopQuadrantTimer('bottomRight');

			stopMonkeyFavorTracking();
			stopFeedbackSfxLoop();
			stopFavorDisplayUpdate();

			// Stop audio
			if (monkeyBgMusic) {
				monkeyBgMusic.pause();
				monkeyBgMusic.currentTime = 0;
			}
			if (monkeyGoodSfx) monkeyGoodSfx.pause();
			if (monkeyBadSfx) monkeyBadSfx.pause();

			// Reset state
			monkeyFavor = 0;
			monkeyFavorLastCheck = 0;
			monkeyThought = '';
		}
	});

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
		if (wheelClickSound === 'custom' && customWheelClickDataUrl) {
			soundFile = customWheelClickDataUrl;
		} else if (wheelClickSound === 'rimshot') {
			soundFile = rimClickSfx;
		} else if (wheelClickSound === 'wood_wheel') {
			soundFile = woodWheelSfx;
		} else if (wheelClickSound === 'kick_13') {
			soundFile = KICK13Sfx;
		} else if (wheelClickSound === 'taiko_drum') {
			soundFile = taikoDrumSfx;
		} else {
			soundFile = wheelClickSfx;
		}
		for (let i = 0; i < 10; i++) {
			wheelClickAudioPool.push(new Audio(soundFile));
		}
		wheelClickPoolIndex = 0;
	}

	function loadResultSound() {
		
		let soundFile;
		if (resultSound === 'custom' && customResultDataUrl) {
			soundFile = customResultDataUrl;
		} else if (resultSound === 'alien') {
			soundFile = alien1Sfx;
		} else if (resultSound === 'monkey') {
			soundFile = monkeySfx;
		} else if (resultSound === 'victory') {
			soundFile = victorySfx;
		} else if (resultSound === 'horror_result') {
			soundFile = horrorResultSfx;
		} else {
			soundFile = alien1Sfx;  // Default 'alien' option
		}
		
		
		if (resultAudio) {
			resultAudio.src = soundFile;
			resultAudio.load();
		} else {
		}
	}

	// Alias for calling from upload handlers
	function initWheelClickAudioPool() {
		loadWheelClickPool();
	}

	// ============================================================================
	// RESPONSIVE CANVAS SIZING
	// ============================================================================

	function calculateOptimalCanvasSize() {
		if (typeof window === 'undefined') return 500;

		const viewportWidth = window.innerWidth;
		const viewportHeight = window.innerHeight;

		// For 1920x1080, target ~800px canvas
		// For smaller screens, scale down proportionally
		const sidebarWidth = 300;
		const availableWidth = viewportWidth - sidebarWidth - 100; // 100px padding
		const availableHeight = viewportHeight - 300; // Reserve space for header/footer/controls

		// Use the smaller dimension to ensure it fits
		let size = Math.min(availableWidth, availableHeight);

		// Clamp between 400px and 900px
		size = Math.max(400, Math.min(900, size));

		return Math.floor(size);
	}

	function handleResize() {
		const newSize = calculateOptimalCanvasSize();
		if (newSize !== canvasSize) {
			canvasSize = newSize;
			// Redraw wheel with new size
			if (wheelCanvas) {
				drawWheel();
			}
		}
	}

	onMount(() => {
		// Initialize audio instances (must be done client-side to avoid SSR issues)
		loadWheelClickPool();
		resultAudio = new Audio();  // Create empty audio element
		loadResultSound();  // Load the selected result sound (default: alien)

		// Horror mode audio will be initialized lazily when theme is selected
		// (removed initialization from here)

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
			handleRefresh();
		} else {
		}

		// Gas can position will be initialized after haunted intro (6-second delay)
		// Removed early repositionGasCan() call to prevent spawning before intro completes

		// Add visibility change listener for pause/resume
		if (typeof document !== 'undefined') {
			document.addEventListener('visibilitychange', handleVisibilityChange);
		}

		// Add mousemove listener for monkey mode quadrant tracking
		if (typeof document !== 'undefined') {
			document.addEventListener('mousemove', updateCurrentQuadrant);
		}

		// Note: Haunted Mode systems are started by the $effect() below, not here
		// This prevents duplicate intervals from running

		// Listen for mouse movement globally (fixes lantern freeze on modals)
		if (typeof window !== 'undefined') {
			window.addEventListener('mousemove', handleMouseMove);
		}

		// Calculate initial canvas size and add resize listener
		canvasSize = calculateOptimalCanvasSize();
		window.addEventListener('resize', handleResize);

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
		// Clean up monkey mode event listeners
		if (typeof document !== 'undefined') {
			document.removeEventListener('mousemove', updateCurrentQuadrant);
		}
		if (jumpscareCountdownInterval) clearInterval(jumpscareCountdownInterval);
		// Remove visibility change listener (browser only)
		if (typeof document !== 'undefined') {
			document.removeEventListener('visibilitychange', handleVisibilityChange);
		}
		// Remove resize listener
		if (typeof window !== 'undefined') {
			window.removeEventListener('resize', handleResize);
		}
		stopHorrorIntroAnimation();
		stopLanternIntroAnimation();
		if (itemCountNotifTimeout) clearTimeout(itemCountNotifTimeout);

		// Remove global mousemove listener
		if (typeof window !== 'undefined') {
			window.removeEventListener('mousemove', handleMouseMove);
		}

	});

	// Watch for theme changes to start/stop Haunted Mode systems
	$effect(() => {
		if (selectedThemePreset === 'haunted' && hauntedInit == false) {
			hauntedInit = true;
			if (showThemeSettings) showThemeSettings = false;
			// Start bg music and tempo variation ONCE, immediately (FIXED - Phase 4)
			if (horrorBgAudio) {
				horrorBgAudio.currentTime = 0; // Reset to start
				horrorBgAudio.volume = hauntedConfig.bgMusicVolumeMin;
				horrorBgAudio.play().catch(e => console.error('Failed to play horror music:', e));
			}
			startTempoVariation();
			// Start haunted intro animation (candlelight starts AFTER intro completes)
			startHauntedIntro();
		} else if (selectedThemePreset !== 'haunted' && hauntedInit == true){
			hauntedInit = false;
			stopFuelDrain();
			stopFlickerAnimation();
			stopTempoVariation();
			stopLanternIntroAnimation();
			stopGhoulSpawnSystem(); // Stop ghoul system
			stopCandlelightSystem(); // Stop candlelight system (NEW - Phase 4 Enhanced)
			// Stop and reset music (FIXED - Phase 4)
			if (horrorBgAudio) {
				horrorBgAudio.pause();
				horrorBgAudio.currentTime = 0; // Reset for next time
			}
			if (warningAudio) warningAudio.pause();
			if (jumpscareCountdownInterval) clearInterval(jumpscareCountdownInterval);
			if (heartbeatAudio) heartbeatAudio.pause();
			// Reset gas can cooldown (NEW - Phase 4)
			if (gasCanCooldownTimer) {
				clearTimeout(gasCanCooldownTimer);
				gasCanCooldownTimer = null;
			}
			gasCanCooldown = false;
			gasCanSpawning = false;
			fuelLevel = 100;
			gasCanSize = hauntedConfig.gasCanSizeInitial;
			gasCanRefuelCount = 0;
			lightFlicker = 0;
			showJumpscare = false;
			jumpscareCountdown = -1;
			jumpscarePhase = 'idle';
			jumpscareAnimationProgress = 0;
			currentJumpscareType = 'default';
			horrorGamePaused = false;
			lanternIntroProgress = 1;
			ghoulActive = false;
			ghoulSize = hauntedConfig.ghoulSizeInitial;
			ghoulCountdown = -1;
			ghoulHoverDuration = 0;
			isHoveringGhoul = false;
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

	// Watch for custom result audio changes
	$effect(() => {
		// Track customResultDataUrl to trigger reload when custom upload changes
		customResultDataUrl;

		if (resultAudio && resultSound === 'custom') {
			// Only reload if we're actually using the custom sound
			loadResultSound();
		}
	});

	// Lantern intro progress for haunted mode (0 = dark, 1 = normal lantern)
	let lanternIntroProgress = $state(1); // 1 = normal, 0 = dark
	let lanternIntroAnimationFrame: number;

		// Helper to start haunted intro animation
		function startHauntedIntro() {
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
					// Start candlelight animations (NOW - after intro fade complete)
					startCandlelightSystem();
					// Start game logic only (no bg music here)
					startFuelDrain();
					startFlickerAnimation();
// Delay first gas can spawn by 6 seconds
					setTimeout(() => {
						repositionGasCan();
					}, 6000);
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

<svelte:head>
	<title>SlippyMudWheel :3</title>
</svelte:head>

		<!-- ============================================================================
		<!-- MAIN LAYOUT
		// ============================================================================ -->

	<div class="app-container" role="application" style="font-family: {selectedFont}; background-color: {wheelBackground}; {backgroundWallpaper === 'custom' && customWallpaperDataUrl ? `background-image: url('${customWallpaperDataUrl}'); background-size: cover; background-position: center; background-attachment: fixed;` : backgroundWallpaper === 'arctic_bg.jpg' ? `background-image: url('${arcticBg}'); background-size: cover; background-position: center; background-attachment: fixed;` : backgroundWallpaper === 'creepy.jpg' && horrorBackgroundUrl ? `background-image: url('${horrorBackgroundUrl}'); background-size: cover; background-position: center; background-attachment: fixed;` : ''}">

		<!-- Header :3 -->
		<header style="background-color: {primaryColor}; border-bottom-color: {primaryColor};">
			<h1 style="color: {secondaryColor};">SlippyMudWheel!</h1>
			<button class="theme-button" style="border-color: {secondaryColor}; color: {secondaryColor};" onclick={toggleThemeSettings}>Theme Settings</button>
		</header>

		<main>
			<!-- ==================================================================== -->
			<!-- GOOGLE DOCS INPUT SECTION -->
			<!-- ==================================================================== -->

			<section class="google-docs-input" style="border-color: {primaryColor};">
				<div class="link-input-container">
					<button class="refresh-button" style="border-color: {primaryColor}; color: {primaryColor};" onclick={handleRefresh} disabled={!googleDocsLink.trim()}>
						<b>Refresh</b>
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
					width={canvasSize}
					height={canvasSize}
					class="wheel-canvas"
					onclick={() => spin(1)}
				></canvas>

				<!-- Monkey Mode UI Overlays -->
				{#if selectedThemePreset === 'monkey' && monkeyModeActive}
					<div class="monkey-favor-display">
						<div class="favor-value">Monkey Social Favor: {monkeyFavorDisplay}</div>
					{#if monkeyThought}
						<div class="monkey-thought" style="color: #FF0000; font-size: 0.8em; margin-top: 12px;">
							monkey thoughts: {monkeyThought}
						</div>
					{/if}
				</div>
			{/if}
			</section>

			<!-- ==================================================================== -->
			<!-- SPIN CONTROLS -->
			<!-- ==================================================================== -->

			<section class="spin-controls" style="border-color: {primaryColor};">
				<div class="multi-spin-header">
					<h3 style="color: {secondaryColor};">Multi Spin</h3>
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
		<!-- Spraypaint tags -->
		<img src={blackTag} alt="" class="spraypaint-left" />
		<img src={black2Tag} alt="" class="spraypaint-right" />

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
					if (selectedThemePreset === 'user') {
						saveUserTheme();
					}
					toggleThemeSettings();
				}}>Save Changes</button>
				<button class="cancel-button" style="border-color: {primaryColor}; color: {primaryColor};" onclick={toggleThemeSettings}>Cancel</button>
			</div>

			<!-- Theme Presets Section -->
			<section class="settings-section" style="border-bottom-color: {primaryColor};">
				<h3 style="color: {primaryColor};">Theme Presets</h3>
				<div class="theme-preset-buttons">

					<!-- CUSTOM THEME -->
					<button
						class="preset-button"
						class:active={selectedThemePreset === 'user'}
						style="border-color: {primaryColor}; {selectedThemePreset === 'user' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
						onclick={() => applyThemePreset('user')}
					>
						🎨 Custom Theme
					</button>

					<!-- ARCTIC -->
					<button
						class="preset-button"
						class:active={selectedThemePreset === 'penguin'}
						style="border-color: {primaryColor}; {selectedThemePreset === 'penguin' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
						onclick={() => applyThemePreset('penguin')}
					>
						🐧 Penguin
					</button>

					<!-- HAUNTED (Hidden on mobile) -->
					{#if !isMobileDevice()}
					<button
						class="preset-button"
						class:active={selectedThemePreset === 'haunted'}
						style="border-color: {primaryColor}; {selectedThemePreset === 'haunted' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
						onclick={() => applyThemePreset('haunted')}
					>
						👻 HAUNTED MODE
					</button>
					{/if}

					<!-- MONKEY MODE (Beta) -->
					<button
						class="preset-button"
						class:active={selectedThemePreset === 'monkey'}
						style="border-color: {primaryColor}; {selectedThemePreset === 'monkey' ? `background-color: ${primaryColor}; color: ${secondaryColor};` : `color: ${primaryColor};`}"
						onclick={() => applyThemePreset('monkey')}
					>
						🐵 MONKEY MODE (Beta)
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
					<button
						style="border-color: {primaryColor}; color: {primaryColor};"
						onclick={() => {
							const input = document.createElement('input');
							input.type = 'file';
							input.accept = '.json';
							input.onchange = importTheme;
							input.click();
						}}
					>Import Theme JSON</button>
					<button
						style="border-color: {primaryColor}; color: {primaryColor};"
						onclick={exportTheme}
					>Export Theme JSON</button>
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
						<input type="file" accept="image/*" style="border-color: {primaryColor};" onchange={handleWallpaperUpload} />
						<select bind:value={backgroundWallpaper} style="border-color: {primaryColor};">
							<option value="">None</option>
							<option value="arctic_bg.jpg">Arctic</option>
							<option value="creepy.jpg">Creepy</option>
							{#if customWallpaperDataUrl}
								<option value="custom">📁 Custom Upload</option>
							{/if}
						</select>
					</div>
					{#if backgroundWallpaper === 'custom' && customWallpaperDataUrl}
						<button class="remove-custom-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
							customWallpaperDataUrl = null;
							backgroundWallpaper = 'arctic_bg.jpg';
							if (selectedThemePreset === 'user') saveUserTheme();
						}}>Remove Custom Wallpaper</button>
					{/if}
				</div>

				<div class="upload-setting-compact">
					<label>Wheel Center Image</label>
					<div class="upload-row">
						<input type="file" accept="image/*" style="border-color: {primaryColor};" onchange={handleCenterImageUpload} />
						<select bind:value={wheelCenterImage} style="border-color: {primaryColor};" onchange={() => drawWheel()}>
							<option value="">None</option>
							<option value="penguin.jpg">Penguin</option>
							<option value="hands.jpg">Hands</option>
							{#if customCenterImageDataUrl}
								<option value="custom">📁 Custom Upload</option>
							{/if}
						</select>
					</div>
					{#if wheelCenterImage === 'custom' && customCenterImageDataUrl}
						<button class="remove-custom-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
							customCenterImageDataUrl = null;
							wheelCenterImage = 'penguin.jpg';
							if (selectedThemePreset === 'user') saveUserTheme();
							drawWheel();
						}}>Remove Custom Image</button>
					{/if}
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

				<!-- Wheel Click Sound -->
				<div class="upload-setting-compact">
					<label class="checkbox-label">
						<input type="checkbox" bind:checked={wheelClickEnabled} disabled={!sfxEnabled} />
						<span style="color: {primaryColor};">Wheel Click Sound</span>
					</label>
					<div class="upload-row">
						<input type="file" accept="audio/*" style="border-color: {primaryColor};" onchange={handleWheelClickUpload} disabled={!sfxEnabled || !wheelClickEnabled} />
						<select
							bind:value={wheelClickSound}
							disabled={!sfxEnabled || !wheelClickEnabled}
							style="border-color: {primaryColor};"
						>
							<option value="wheel_click">Wheel Click</option>
							<option value="rimshot">Rimshot</option>
							<option value="wood_wheel">Wood Creak</option>
							<option value="kick_13">KICK 13</option>
							<option value="taiko_drum">Taiko Drum</option>
							{#if customWheelClickDataUrl}
								<option value="custom">📁 Custom Upload</option>
							{/if}
						</select>
					</div>
					{#if wheelClickSound === 'custom' && customWheelClickDataUrl}
						<button class="remove-custom-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
							customWheelClickDataUrl = null;
							wheelClickSound = 'rimshot';
							if (selectedThemePreset === 'user') saveUserTheme();
							loadWheelClickPool();
						}}>Remove Custom Sound</button>
					{/if}
				</div>

				<!-- Results Sound -->
				<div class="upload-setting-compact">
					<label class="checkbox-label">
						<input type="checkbox" bind:checked={resultEnabled} disabled={!sfxEnabled} />
						<span style="color: {primaryColor};">Results Sound</span>
					</label>
					<div class="upload-row">
						<input type="file" accept="audio/*" style="border-color: {primaryColor};" onchange={handleResultUpload} disabled={!sfxEnabled || !resultEnabled} />
						<select
							bind:value={resultSound}
							disabled={!sfxEnabled || !resultEnabled}
							style="border-color: {primaryColor};"
							onchange={() => {
								if (resultAudio) {
									loadResultSound();
								}
							}}
						>
							<option value="victory">Victory</option>
							<option value="alien">Alien</option>
							<option value="monkey">Monkey</option>
							{#if customResultDataUrl}
								<option value="custom">📁 Custom Upload</option>
							{/if}
						</select>
					</div>
					{#if customResultDataUrl}
						<button class="remove-custom-btn" style="border-color: {primaryColor}; color: {primaryColor};" onclick={() => {
							customResultDataUrl = null;
							resultSound = 'alien';  // Reset to default alien sound
							if (selectedThemePreset === 'user') saveUserTheme();
							loadResultSound();  // Reload with alien sound
						}}>Remove Custom Result</button>
					{/if}
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
	<!-- HAUNTED MODE OVERLAYS AND ELEMENTS -->
	<!-- ============================================================================ -->

	{#if selectedThemePreset === 'haunted'}
	<!-- Use only one lantern-overlay for haunted mode, always rendered -->
	<div
		class="lantern-overlay"
		style="
			--cursor-x: {cursorX + handShakeOffsetX}px;
			--cursor-y: {cursorY + handShakeOffsetY}px;
			--fuel-level: {fuelLevel};
			--flicker: {lightFlicker};
			--lantern-intro: {lanternIntroProgress};
			--core-radius: {coreRadius}px;
			--inner-radius: {innerRadius}px;
			--outer-radius: {outerRadius}px;
		"
	></div>


	<!-- Show gas can only after intro is done AND not on cooldown -->
	{#if lanternIntroProgress === 1 && !gasCanCooldown}
		<img
			src={gasCanImg}
			alt="Gas Can"
			class="gas-can {gasCanSpawning ? 'spawning' : ''}"
			style="left: {gasCanX}px; top: {gasCanY}px; width: {gasCanSize}px; --fade-duration: {gasCanFadeDuration}ms;"
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
	{#if selectedThemePreset === 'haunted' && showJumpscare}
	<div class="jumpscare-overlay" class:fadeout={jumpscarePhase === 'fadeout'}>
		{#if jumpscarePhase === 'jumpscare' || jumpscarePhase === 'scream'}
			<img
				src={currentJumpscareType === 'ghoul' ? ghoulGif : (horrorJumpscareGifUrl || '')}
				alt="Jumpscare"
				class="jumpscare-gif"
				style="opacity: {jumpscareAnimationProgress};"
				bind:this={jumpscareGifEl}
				{...{loop: false}}
			/>
		{/if}
	</div>
	{/if}

	<!-- Monkey Mode Fade to Black Overlay -->
	{#if selectedThemePreset === 'monkey' && monkeyFadeToBlack}
	<div class="monkey-fade-overlay" style="opacity: {monkeyFadeOpacity};"></div>
	{/if}

	<!-- Monkey Mode Video Reward Overlay -->
	{#if selectedThemePreset === 'monkey' && showMonkeyVideo && monkeyVideoUrl}
	<div class="monkey-video-overlay" style="opacity: {monkeyVideoOpacity};">
		<video
			bind:this={monkeyVideoElement}
			src={monkeyVideoUrl}
			class="monkey-video"
			autoplay
			muted={false}
			preload="auto"
		></video>
	</div>
	{/if}

<!-- ============================================================================ -->
<!-- LOADING OVERLAY -->
<!-- ============================================================================ -->

{#if horrorAssetsLoading || monkeyAssetsLoading}
<div class="loading-overlay">
	<div class="loading-spinner"></div>
	<p style="color: white; font-size: 1.5rem; margin-top: 1rem;">
		{horrorAssetsLoading ? 'Loading Horror Mode...' : 'Loading Monkey Mode...'}
	</p>
</div>
{/if}

	<!-- ============================================================================ -->
	<!-- STYLES -->
	<!-- ============================================================================ -->

	<style>
		/* CSS Custom Properties / Variables */
		:root {
			/* Border radius tokens */
			--radius-sm: 4px;
			--radius-md: 8px;
			--radius-lg: 12px;
			--radius-xl: 16px;

			/* Transition tokens */
			--transition-fast: all 0.15s ease;
			--transition-standard: all 0.2s ease;
			--transition-slow: all 0.3s ease;
		}

		:global(*) {
			margin: 0;
			padding: 0;
			box-sizing: border-box;
		}

		:global(body) {
			font-family: Arial, sans-serif;
			overflow: hidden;
		}

		/* Base Button Styles */
		.btn {
			padding: 0.5rem 1rem;
			font-size: 1rem;
			cursor: pointer;
			border: 2px solid;
			border-radius: var(--radius-md);
			background: #fff;
			transition: var(--transition-standard);
		}

		.btn:hover:not(:disabled) {
			background: rgba(114, 47, 55, 0.1);
		}

		.btn:disabled {
			opacity: 0.5;
			cursor: not-allowed;
		}

		.btn-transparent {
			background: transparent;
		}

		.btn-transparent:hover {
			background: rgba(255, 255, 255, 0.2);
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
			border-radius: var(--radius-md);
			background: transparent;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-md);
			background: #fff;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-sm);
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
			border-radius: var(--radius-lg);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-md);
			background: #fff;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-md);
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
			border-radius: var(--radius-md);
			background: #fff;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-md);
			transition: var(--transition-standard);
		}

		.history-item:hover {
			transform: translateX(-4px);
			box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
		}

		.history-item-content {
			display: flex;
			justify-content: space-between;
			align-items: flex-start;  /* Allow vertical expansion instead of horizontal */
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
			transition: var(--transition-standard);
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
			position: relative;     /* Allows absolute positioning of spraypaint images */
			overflow: hidden;       /* Clip all overflow - no scrollbars */
		}

		/* Spraypaint tag images - positioned to look "wrapped on" the footer */
		.spraypaint-left {
			position: absolute;
			left: 5%;           /* Closer to center */
			top: 5%;        /* Partially cut off at bottom */
			width: 300px;         /* Larger size */
			transform: rotate(-15deg);  /* Tilted left for natural spraypaint look */
			opacity: 0.7;
			pointer-events: none; /* Don't interfere with footer text clicks */
			z-index: 1;          /* Above footer background, below text */
		}

		.spraypaint-right {
			position: absolute;
			right: 5%;          /* Closer to center */
			top: 2%;        /* Partially cut off at bottom */
			width: 280px;         /* Larger size */
			transform: rotate(12deg);   /* Tilted right for natural spraypaint look */
			opacity: 0.7;
			pointer-events: none;
			z-index: 1;
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
			border-radius: var(--radius-md);
			background: #fff;
			cursor: pointer;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-lg);
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
			border-radius: var(--radius-sm);
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
			border-radius: var(--radius-sm);
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
			border-radius: var(--radius-md);
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
			transition: var(--transition-standard);
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
			transition: var(--transition-standard);
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
			border-radius: var(--radius-md);
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

		.remove-custom-btn {
			margin-top: 0.5rem;
			padding: 0.4rem 0.8rem;
			border: 2px solid;
			border-radius: 6px;
			background-color: transparent;
			font-size: 0.85rem;
			cursor: pointer;
			transition: var(--transition-standard);
		}

		.remove-custom-btn:hover {
			background-color: rgba(255, 0, 0, 0.1);
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
			border-radius: var(--radius-md);
			cursor: pointer;
			transition: var(--transition-standard);
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
			border-radius: var(--radius-xl);
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
			border-radius: var(--radius-lg);
			padding: 1.5rem;
			animation: bounceIn 0.5s ease-out;
		}

		.result-content {
			display: flex;
			justify-content: space-between;
			align-items: flex-start;  /* Allow vertical expansion instead of horizontal */
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
			border-radius: var(--radius-md);
			font-size: 1rem;
			cursor: pointer;
			transition: var(--transition-standard);
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
			z-index: 999999; /* Cover everything including theme settings modal */

			/* ====================================================================
			   CANDLELIGHT SYSTEM - Multi-layered organic lighting
			   ==================================================================== */
			/* Enhanced dimming: 30% at 0% fuel, 100% at 100% fuel */
			--light-brightness: calc(0.3 + (var(--fuel-level) / 100) * 0.7);

			/* Logarithmic color temperature (warm → cool transition)
			   Stays warm until ~20% fuel, then rapidly shifts to cool blue */
			--coolness-factor: calc(max(0, 1 - pow(var(--fuel-level) / 20, 2)));

			--warm-red: calc(255 - (75 * var(--coolness-factor)));    /* 255 → 180 */
			--warm-green: calc(220 - (20 * var(--coolness-factor)));  /* 220 → 200 */
			--warm-blue: calc(180 + (75 * var(--coolness-factor)));   /* 180 → 255 */

			/* Enhanced vignette darkening - much darker at low fuel */
			--vignette: calc(0.85 + (0.15 * (1 - var(--fuel-level) / 100)));

			/* ====================================================================
			   MULTI-LAYERED CANDLELIGHT WITH THEME INTEGRATION
			   ==================================================================== */
			background:
				/* Layer 3: Outer glow - ambient light falloff with exponential dimming */
				radial-gradient(
					circle var(--outer-radius) at var(--cursor-x) var(--cursor-y),
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.35 * pow(var(--light-brightness), 1.5))) 0%,
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.08 * pow(var(--light-brightness), 1.5))) 40%,
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.02 * pow(var(--light-brightness), 1.5))) 85%,
					transparent 100%
				),
				/* Layer 2: Inner glow - main illumination zone */
				radial-gradient(
					circle var(--inner-radius) at var(--cursor-x) var(--cursor-y),
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.35 * var(--light-brightness))) 0%,
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.20 * var(--light-brightness))) 50%,
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.05 * var(--light-brightness))) 85%,
					transparent 100%
				),
				/* Layer 1: Core - bright center */
				radial-gradient(
					circle var(--core-radius) at var(--cursor-x) var(--cursor-y),
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.60 * var(--light-brightness))) 0%,
					rgba(var(--warm-red), var(--warm-green), var(--warm-blue), calc(0.40 * var(--light-brightness))) 60%,
					transparent 100%
				),
				/* Base darkness - much darker at low fuel */
				radial-gradient(
					circle 300px at var(--cursor-x) var(--cursor-y),
					transparent 0%,
					rgba(0, 0, 0, calc(0.7 * var(--vignette))) 40%,
					rgba(0, 0, 0, calc(0.95 * var(--vignette))) 100%
				);
			transition: background 0.4s ease-in-out;
		}

		.lantern-overlay.haunted-intro {
			/* Animate all three candlelight radii during intro fade-in */
			--core-radius: calc(50px * var(--lantern-intro));
			--inner-radius: calc(160px * var(--lantern-intro));
			--outer-radius: calc(225px * var(--lantern-intro));
			--light-brightness: calc((var(--fuel-level) / 100) * var(--lantern-intro));
			transition: background 0.15s ease-out, --light-brightness 1.5s;
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

		/* ========================================================================== */
		/* LOADING OVERLAY */
		/* ========================================================================== */

		.loading-overlay {
			position: fixed;
			top: 0;
			left: 0;
			width: 100vw;
			height: 100vh;
			background: rgba(0, 0, 0, 0.9);
			display: flex;
			flex-direction: column;
			align-items: center;
			justify-content: center;
			z-index: 9999999;
		}

		.loading-spinner {
			width: 60px;
			height: 60px;
			border: 4px solid rgba(255, 255, 255, 0.3);
			border-top-color: white;
			border-radius: 50%;
			animation: spin 1s linear infinite;
		}

		@keyframes spin {
			to { transform: rotate(360deg); }
		}

		.gas-can.spawning {
			animation: gasCanFadeIn var(--fade-duration, 3000ms) ease-out forwards;
		}

		@keyframes gasCanFadeIn {
			from {
				opacity: 0;
				transform: scale(0.8);
				filter: drop-shadow(0 0 0 rgba(255, 200, 0, 0));
			}
			to {
				opacity: 0.7;
				transform: scale(1);
				filter: drop-shadow(0 0 12px rgba(255, 200, 0, 0.6));
			}
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
			z-index: 1000000; /* Above everything */
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
			border-radius: var(--radius-md);
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

		/* ================================================================== */
		/* RESPONSIVE MEDIA QUERIES */
		/* ================================================================== */

		/* Large screens (1920x1080 and similar widescreen) */
		@media (min-width: 1800px) {
			.app-container {
				grid-template-columns: 1fr 350px; /* Wider sidebar */
			}

			.wheel-section {
				padding: 2rem 4rem; /* More horizontal padding */
			}

			h1 {
				font-size: clamp(2rem, 3vw, 3.5rem);
			}

			button {
				font-size: clamp(0.9rem, 1.1vw, 1.1rem);
			}
		}

		/* Medium screens (standard laptops) */
		@media (max-width: 1366px) {
			.app-container {
				grid-template-columns: 1fr 250px; /* Narrower sidebar */
			}

			h1 {
				font-size: clamp(1.5rem, 2.5vw, 2rem);
			}
		}

		/* Mobile devices - vertical stacking layout */
		@media (max-width: 768px) {
			.app-container {
				grid-template-columns: 1fr; /* Full width, stack vertically */
				grid-template-rows: auto auto auto auto; /* Header, Main, Sidebar, Footer */
				height: auto;
				overflow-y: auto;
				overflow-x: hidden;
			}

			header {
				padding: 0.75rem 1rem;
				display: flex;
				flex-direction: column;
				gap: 0.75rem;
				align-items: center;
			}
			
			.theme-button {
				width: 100%;
				max-width: 300px;
			}

			/* Order: 1. Link input at top */
			.google-docs-input {
				order: 1;
				padding: 1rem;
			}

			/* Order: 2. Wheel section */
			.wheel-section {
				order: 2;
				padding: 1rem;
				min-height: auto;
			}

			/* Order: 3. History sidebar becomes full-width section */
			.history-sidebar {
				order: 3;
				max-height: none; /* Allow full height - scroll page to see */
				overflow-y: visible;
				border-left: none;
				border-top: 2px solid;
				padding: 1rem;
			}

			/* Order: 4. Footer at bottom */
			footer {
				order: 4;
				padding: 1rem;
			}

			/* Adjust main to be flex container for proper ordering */
			main {
				display: flex;
				flex-direction: column;
				overflow-y: visible;
			}

			/* Typography adjustments */
			h1 {
				font-size: 1.5rem;
			}

			h2 {
				font-size: 1.25rem;
			}

			button {
				font-size: 0.875rem;
				padding: 0.5rem 1rem;
			}

			/* Hide horror mode elements on mobile */
			.lantern-overlay,
			.gas-can,
			.ghoul-sprite,
			.fuel-gauge {
				display: none !important;
			}

			/* Make canvas responsive */
			canvas {
				max-width: 100%;
				height: auto;
			}

			/* Adjust multi-spin controls */
			.multi-spin-controls {
				flex-wrap: wrap;
				justify-content: center;
				gap: 0.5rem;
			}

			.custom-spin-container {
				width: 100%;
				justify-content: center;
			}
			
			/* Spraypaint tags for mobile - only show right tag */
			.spraypaint-left {
				display: none; /* Hide left spraypaint on mobile */
			}
			
			.spraypaint-right {
				width: 110px;
				right: -15px;
				top: -15px;
			}
			
			/* Better mobile spacing */
			.google-docs-input {
				padding: 0.75rem;
			}
			
			.link-input {
				font-size: 0.875rem;
				padding: 0.5rem;
			}
			
			.refresh-button {
				padding: 0.5rem 0.75rem;
				font-size: 0.875rem;
			}
			
			/* Footer text sizing */
			.footer-main {
				font-size: 1rem;
			}
			
			.footer-small {
				font-size: 0.8rem;
			}
			
			/* Ticker sizing */
			.ticker-container {
				padding: 0.5rem 1rem;
			}
			
			.ticker-text {
				font-size: 1.5rem;
			}
			
			/* Multi-spin button sizing */
			.multi-spin-btn {
				padding: 0.4rem 0.8rem;
				font-size: 0.875rem;
			}
			
			.custom-spin-input {
				width: 60px;
				padding: 0.4rem;
				font-size: 0.875rem;
			}
			
			.custom-spin-btn {
				padding: 0.4rem 0.8rem;
				font-size: 0.875rem;
			}
		}

		/* ================================================================ */
		/* MONKEY MODE STYLES */
		/* ================================================================ */

		.monkey-favor-display {
			position: absolute;
			top: 10px;
			right: 10px;
			background: rgba(139, 69, 19, 0.8);
			padding: 0.5rem 1rem;
			border-radius: 8px;
			color: white;
			font-weight: bold;
			z-index: 100;
		}

		.monkey-thought {
			font-size: 1.2rem;
			font-weight: bold;
			text-shadow: 0 0 10px rgba(0, 0, 0, 0.8);
		}

		.monkey-fade-overlay {
			position: fixed;
			top: 0;
			left: 0;
			width: 100vw;
			height: 100vh;
			background: black;
			z-index: 9999999;
			pointer-events: none;
			transition: opacity 0.3s ease;
		}

		.monkey-video-overlay {
			position: fixed;
			top: 0;
			left: 0;
			width: 100vw;
			height: 100vh;
			background: black;
			z-index: 10000000;
			display: flex;
			align-items: center;
			justify-content: center;
			pointer-events: none;
			transition: opacity 0.5s ease;
		}

		.monkey-video {
			max-width: 100%;
			max-height: 100%;
			object-fit: contain;
		}

	</style>
