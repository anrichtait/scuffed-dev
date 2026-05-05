<script lang="ts">
	let email = $state('');
	let submitted = $state(false);
	let error = $state('');

	function handleSubmit(e: Event) {
		e.preventDefault();
		if (!email.includes('@') || !email.includes('.')) {
			error = 'Please enter a valid email address.';
			return;
		}
		error = '';
		submitted = true;
		// TODO: wire up to a real email service (e.g. Resend, Mailchimp)
	}
</script>

<svelte:head>
	<title>Anrich Tait — Coming Soon</title>
	<meta name="description" content="The personal site of Anrich Tait. Coming soon." />
</svelte:head>

<div class="page">

	<!-- Gradient orbs -->
	<div class="orb orb--1" aria-hidden="true"></div>
	<div class="orb orb--2" aria-hidden="true"></div>
	<div class="orb orb--3" aria-hidden="true"></div>

	<main class="content">

		<a href="/" class="back" aria-label="Back to home">
			<svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" aria-hidden="true">
				<path d="M19 12H5M5 12l7-7M5 12l7 7"/>
			</svg>
			Back
		</a>

		<div class="eyebrow">
			<span class="dot"></span>
			<span>Something is coming</span>
		</div>

		<h1 class="heading">Anrich<br/>Tait</h1>

		<p class="sub">
			Photography, writing, and geopolitics.<br/>
			A personal corner of the internet — coming soon.
		</p>

		{#if !submitted}
			<form class="form" onsubmit={handleSubmit} novalidate>
				<p class="form__label">Get notified when it launches</p>
				<div class="form__row">
					<input
						class="form__input"
						type="email"
						placeholder="your@email.com"
						bind:value={email}
						aria-label="Email address"
						autocomplete="email"
					/>
					<button class="form__btn" type="submit">Notify me</button>
				</div>
				{#if error}
					<p class="form__error">{error}</p>
				{/if}
			</form>
		{:else}
			<div class="success">
				<svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#0071e3" stroke-width="2" aria-hidden="true">
					<path d="M20 6L9 17l-5-5"/>
				</svg>
				<p>You're on the list. We'll be in touch.</p>
			</div>
		{/if}

	</main>

	<footer class="footer">
		<p>© {new Date().getFullYear()} Anrich Tait</p>
		<a href="/dev">ScuffedDev →</a>
	</footer>

</div>

<style>
/* ── Page ─────────────────────────────────────────────────────── */
.page {
	min-height: 100dvh;
	background: #ffffff;
	display: flex;
	flex-direction: column;
	font-family: 'Inter', sans-serif;
	position: relative;
	overflow: hidden;
}

/* ── Gradient orbs ────────────────────────────────────────────── */
.orb {
	position: absolute;
	border-radius: 50%;
	pointer-events: none;
	filter: blur(100px);
}
.orb--1 {
	width: 600px;
	height: 600px;
	background: radial-gradient(circle, rgba(0,113,227,0.1) 0%, transparent 70%);
	top: -150px;
	right: -100px;
}
.orb--2 {
	width: 400px;
	height: 400px;
	background: radial-gradient(circle, rgba(90,200,250,0.1) 0%, transparent 70%);
	bottom: -100px;
	left: -80px;
}
.orb--3 {
	width: 300px;
	height: 300px;
	background: radial-gradient(circle, rgba(0,113,227,0.06) 0%, transparent 70%);
	top: 40%;
	left: 30%;
}

/* ── Content ──────────────────────────────────────────────────── */
.content {
	flex: 1;
	display: flex;
	flex-direction: column;
	justify-content: center;
	padding: clamp(4rem, 10vw, 8rem) clamp(1.5rem, 8vw, 6rem);
	max-width: 680px;
	position: relative;
	z-index: 1;
}

/* ── Back link ────────────────────────────────────────────────── */
.back {
	display: inline-flex;
	align-items: center;
	gap: 0.4rem;
	font-size: 0.8125rem;
	color: #6e6e73;
	text-decoration: none;
	margin-bottom: 3rem;
	width: fit-content;
	transition: color 150ms ease;
}
.back:hover { color: #1d1d1f; }

/* ── Eyebrow ──────────────────────────────────────────────────── */
.eyebrow {
	display: flex;
	align-items: center;
	gap: 0.5rem;
	font-size: 0.75rem;
	font-weight: 500;
	color: #0071e3;
	letter-spacing: 0.08em;
	text-transform: uppercase;
	margin-bottom: 1.5rem;
}
.dot {
	width: 6px;
	height: 6px;
	background: #0071e3;
	border-radius: 50%;
	animation: pulse 2s ease-in-out infinite;
}
@keyframes pulse {
	0%, 100% { opacity: 1; transform: scale(1); }
	50%       { opacity: 0.4; transform: scale(0.8); }
}

/* ── Heading ──────────────────────────────────────────────────── */
.heading {
	font-family: 'Bebas Neue', sans-serif;
	font-size: clamp(5rem, 12vw, 9rem);
	font-weight: 400;
	line-height: 0.92;
	letter-spacing: 0.01em;
	color: #1d1d1f;
	margin-bottom: 1.75rem;
}

/* ── Sub ──────────────────────────────────────────────────────── */
.sub {
	font-size: clamp(1rem, 1.5vw, 1.125rem);
	font-weight: 300;
	color: #6e6e73;
	line-height: 1.7;
	margin-bottom: 3rem;
}

/* ── Form ─────────────────────────────────────────────────────── */
.form__label {
	font-size: 0.8125rem;
	color: #1d1d1f;
	font-weight: 500;
	margin-bottom: 0.75rem;
}
.form__row {
	display: flex;
	gap: 0.5rem;
	flex-wrap: wrap;
}
.form__input {
	flex: 1;
	min-width: 200px;
	font-family: 'Inter', sans-serif;
	font-size: 0.9375rem;
	font-weight: 300;
	color: #1d1d1f;
	background: rgba(0,0,0,0.03);
	border: 1px solid rgba(0,0,0,0.1);
	border-radius: 980px;
	padding: 0.65rem 1.25rem;
	outline: none;
	transition: border-color 150ms ease, box-shadow 150ms ease;
}
.form__input::placeholder { color: #b0b0b5; }
.form__input:focus {
	border-color: rgba(0,113,227,0.5);
	box-shadow: 0 0 0 3px rgba(0,113,227,0.1);
}
.form__btn {
	font-family: 'Inter', sans-serif;
	font-size: 0.9375rem;
	font-weight: 500;
	color: #ffffff;
	background: #0071e3;
	border: none;
	border-radius: 980px;
	padding: 0.65rem 1.5rem;
	cursor: pointer;
	transition: background 150ms ease;
	white-space: nowrap;
}
.form__btn:hover { background: #0077ed; }
.form__btn:active { background: #006edb; }

.form__error {
	font-size: 0.8125rem;
	color: #ff3b30;
	margin-top: 0.5rem;
}

/* ── Success ──────────────────────────────────────────────────── */
.success {
	display: flex;
	align-items: center;
	gap: 0.6rem;
	font-size: 0.9375rem;
	font-weight: 400;
	color: #1d1d1f;
}

/* ── Footer ───────────────────────────────────────────────────── */
.footer {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 1.5rem clamp(1.5rem, 8vw, 6rem);
	font-size: 0.8125rem;
	color: #6e6e73;
	border-top: 1px solid rgba(0,0,0,0.06);
	position: relative;
	z-index: 1;
}
.footer a {
	color: #0071e3;
	text-decoration: none;
	transition: color 150ms ease;
}
.footer a:hover { color: #0077ed; }

/* ── Mobile ───────────────────────────────────────────────────── */
@media (max-width: 480px) {
	.form__row { flex-direction: column; }
	.form__input, .form__btn { width: 100%; border-radius: 12px; }
}
</style>