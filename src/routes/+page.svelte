<script lang="ts">
	let mouseX = $state(0);
	let mouseY = $state(0);
	let devHovered = $state(false);
	let personalHovered = $state(false);
    let currentSprite = $state('');

    const sprites = [
        'images/sprites/whimsicott.png',
        'images/sprites/suicune.png',
        'images/sprites/shaymin-sky.png',
        'images/sprites/poliwrath.png',
        'images/sprites/poliwrath.png',
        'images/sprites/oricorio-sensu.png',
        'images/sprites/gengar.png',
        'images/sprites/empoleon.png',
        'images/sprites/drifloon.png'
    ]

    function randomSprite() {
        return sprites[Math.floor(Math.random() * sprites.length)];
    }

	function onMouseMove(e: MouseEvent) {
		mouseX = e.clientX;
		mouseY = e.clientY;
	}

    function onDevEnter() {
        currentSprite = randomSprite();
        devHovered = true;
    }

    function onDevLeave() {
        devHovered = false;
    }
</script>

<svelte:head>
	<title>Anrich Tait</title>
	<meta name="description" content="ScuffedDev tools and personal work by Anrich Tait." />
</svelte:head>

<svelte:window onmousemove={onMouseMove} />

<!-- Floating preview popup -->
{#if devHovered || personalHovered}
	<div
		class="popup"
		class:popup--dev={devHovered}
		class:popup--personal={personalHovered}
		style="left: {mouseX + 12}px; top: {mouseY + 12}px;"
	>
		<img
			src={devHovered ? currentSprite : '/images/personal-preview.jpeg'}
			alt={devHovered ? 'Pokemon Sprite' : 'Personal preview'}
			width="120"
			height="120"
			loading="eager"
		/>
	</div>
{/if}

<div class="split">

	<!-- Dev side -->
	<a
		href="/dev"
		class="half half--dev"
		aria-label="Go to ScuffedDev"
		onmouseenter={onDevEnter}
		onmouseleave={onDevLeave}
	>
		<!-- Gradient orbs -->
		<div class="dev-orb dev-orb--1" aria-hidden="true"></div>
		<div class="dev-orb dev-orb--2" aria-hidden="true"></div>

		<div class="half__inner">
			<p class="dev-label">~/scuffed-dev</p>
			<h1 class="dev-title">Scuffed</h1>
			<h1 class="dev-title">Developer</h1>
			<p class="dev-sub">Tools. Calcs. Blog.</p>
			<span class="dev-cta">
				cd ./dev <span class="cursor">▋</span>
			</span>
		</div>
	</a>

	<!-- Personal side -->
	<a
		href="/anrich"
		class="half half--personal"
		aria-label="Go to Anrich Tait personal site"
		onmouseenter={() => personalHovered = true}
		onmouseleave={() => personalHovered = false}
	>
		<!-- Gradient orbs -->
		<div class="personal-orb personal-orb--1" aria-hidden="true"></div>
		<div class="personal-orb personal-orb--2" aria-hidden="true"></div>

		<div class="half__inner">
			<p class="personal-label">EST. 2000 · JHB</p>
			<h2 class="personal-title">Anrich<br/>Tait</h2>
			<p class="personal-sub">Photography & writing.</p>
			<span class="personal-cta">
				<span class="personal-cta__inner">Enter →</span>
			</span>
		</div>
	</a>

</div>

<style>
/* ── Layout ───────────────────────────────────────────────── */
.split {
	display: grid;
	grid-template-columns: 1fr 1fr;
	min-height: 100dvh;
}

.half {
	display: flex;
	align-items: center;
	justify-content: center;
	padding: 3rem;
	position: relative;
	overflow: hidden;
	transition: filter 400ms ease;
}

.half__inner {
	display: flex;
	flex-direction: column;
	gap: 1rem;
	max-width: 420px;
	width: 100%;
	position: relative;
	z-index: 1;
}

.split:has(.half--dev:hover) .half--personal,
.split:has(.half--personal:hover) .half--dev {
	filter: brightness(0.55);
}

/* ── Dev side ─────────────────────────────────────────────── */
.half--dev {
	background-color: #2f2f2f;
	border-right: 1px solid rgba(255, 255, 255, 0.06);
}

/* Subtle gradient orbs sitting behind the text */
.dev-orb {
	position: absolute;
	border-radius: 50%;
	filter: blur(80px);
	opacity: 0;
	transition: opacity 600ms ease;
	pointer-events: none;
}
.half--dev:hover .dev-orb { opacity: 1; }

.dev-orb--1 {
	width: 340px;
	height: 340px;
	background: radial-gradient(circle, rgba(80, 200, 160, 0.18) 0%, transparent 70%);
	top: -80px;
	left: -60px;
}
.dev-orb--2 {
	width: 280px;
	height: 280px;
	background: radial-gradient(circle, rgba(100, 120, 255, 0.14) 0%, transparent 70%);
	bottom: -60px;
	right: -40px;
}

.dev-label {
	font-family: 'Cozette', monospace;
	font-size: 0.75rem;
	color: rgba(255, 255, 255, 0.3);
	letter-spacing: 0.05em;
}
.dev-title {
	font-family: 'Cozette', monospace;
	font-size: clamp(2.5rem, 5vw, 4rem);
	font-weight: normal;
	line-height: 1;
	color: #ffffff;
}
.dev-sub {
	font-family: 'Cozette', monospace;
	font-size: 0.8125rem;
	color: rgba(255, 255, 255, 0.45);
	letter-spacing: 0.03em;
}
.dev-cta {
	font-family: 'Cozette', monospace;
	font-size: 0.8125rem;
	color: rgba(255, 255, 255, 0.9);
	display: flex;
	align-items: center;
	gap: 0.25rem;
	margin-top: 1rem;
	border: 1px solid rgba(255, 255, 255, 0.12);
	padding: 0.5rem 0.75rem;
	width: fit-content;
	transition: border-color 200ms ease, background 200ms ease;
}
.half--dev:hover .dev-cta {
	border-color: rgba(255, 255, 255, 0.35);
	background: rgba(255, 255, 255, 0.04);
}

.cursor {
	animation: blink 1.1s step-end infinite;
}
@keyframes blink {
	0%, 100% { opacity: 1; }
	50%       { opacity: 0; }
}

/* ── Personal side ────────────────────────────────────────── */
.half--personal {
	background: #ffffff;
}

/* Soft blue gradient orbs — Apple-style */
.personal-orb {
	position: absolute;
	border-radius: 50%;
	filter: blur(90px);
	pointer-events: none;
	transition: opacity 600ms ease;
}

.personal-orb--1 {
	width: 500px;
	height: 500px;
	background: radial-gradient(circle, rgba(0, 113, 227, 0.1) 0%, transparent 70%);
	top: -100px;
	right: -100px;
	opacity: 0.6;
}
.personal-orb--2 {
	width: 380px;
	height: 380px;
	background: radial-gradient(circle, rgba(90, 200, 250, 0.12) 0%, transparent 70%);
	bottom: -80px;
	left: -60px;
	opacity: 0;
	transition: opacity 600ms ease;
}
.half--personal:hover .personal-orb--2 { opacity: 0.8; }
.half--personal:hover .personal-orb--1 { opacity: 1; }

.personal-label {
	font-family: 'Inter', sans-serif;
	font-size: 0.6875rem;
	font-weight: 400;
	color: #6e6e73;
	letter-spacing: 0.12em;
	text-transform: uppercase;
}
.personal-title {
	font-family: 'Bebas Neue', sans-serif;
	font-size: clamp(4rem, 8vw, 7rem);
	font-weight: 400;
	line-height: 0.95;
	letter-spacing: 0.01em;
	color: #1d1d1f;
}
.personal-sub {
	font-family: 'Inter', sans-serif;
	font-size: 0.9375rem;
	font-weight: 300;
	color: #6e6e73;
}

/* Frosted glass Apple-style button */
.personal-cta {
	margin-top: 1rem;
	width: fit-content;
}
.personal-cta__inner {
	display: inline-flex;
	align-items: center;
	gap: 0.25rem;
	font-family: 'Inter', sans-serif;
	font-size: 0.875rem;
	font-weight: 500;
	color: #0071e3;
	background: rgba(0, 113, 227, 0.08);
	backdrop-filter: blur(12px);
	-webkit-backdrop-filter: blur(12px);
	border: 1px solid rgba(0, 113, 227, 0.2);
	border-radius: 980px;
	padding: 0.5rem 1.1rem;
	transition: background 200ms ease, border-color 200ms ease, color 200ms ease;
}
.half--personal:hover .personal-cta__inner {
	background: rgba(0, 113, 227, 0.14);
	border-color: rgba(0, 113, 227, 0.4);
	color: #0077ed;
}

/* ── Floating popup ───────────────────────────────────────── */
.popup {
	position: fixed;
	z-index: 1000;
	pointer-events: none;
	width: 120px;
	height: 120px;
	animation: popup-in 150ms ease forwards;
}
@keyframes popup-in {
	from { opacity: 0; transform: scale(0.85); }
	to   { opacity: 1; transform: scale(1); }
}

.popup img {
	width: 120px;
	height: 120px;
	display: block;
}
.popup--dev img {
	object-fit: contain;
	background: transparent;
	border: none;
	padding: 8px;
	image-rendering: pixelated;
}
.popup--personal img {
	object-fit: cover;
	/*border-radius: 6px;*/
	box-shadow: 0 8px 32px rgba(0, 0, 0, 0.14);
}

/* ── Mobile ───────────────────────────────────────────────── */
@media (max-width: 640px) {
	.split {
		grid-template-columns: 1fr;
		grid-template-rows: 1fr 1fr;
	}
	.half { padding: 3rem 2rem; min-height: 50dvh; }
	.split:has(.half--dev:hover) .half--personal,
	.split:has(.half--personal:hover) .half--dev {
		filter: none;
	}
	.popup { display: none; }
}
</style>