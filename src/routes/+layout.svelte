<script lang="ts">
import favicon from '$lib/assets/favicon.svg';
let { children } = $props();

import { dev } from '$app/environment';
import { injectAnalytics } from '@vercel/analytics/sveltekit';
injectAnalytics({ mode: dev ? 'development' : 'production' });
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link href="https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Inter:wght@300;400;500&display=swap" rel="stylesheet" />
</svelte:head>

{@render children()}

<style>
	@font-face {
		font-family: 'Cozette';
		src: url('/fonts/cozette.ttf') format('truetype');
		font-weight: normal;
		font-style: normal;
		font-display: swap;
	}

	:global(*, *::before, *::after) {
		box-sizing: border-box;
		margin: 0;
		padding: 0;
	}

	:global(html) {
		-webkit-font-smoothing: antialiased;
		scroll-behavior: smooth;
	}

	:global(body) {
		min-height: 100dvh;
		line-height: 1.6;
		background: none;
	}

	:global(a) { text-decoration: none; }
	:global(button) { cursor: pointer; background: none; border: none; font: inherit; }
	:global(img, svg) { display: block; max-width: 100%; }

	:global(:focus-visible) {
		outline: 2px solid currentColor;
		outline-offset: 3px;
	}

</style>
