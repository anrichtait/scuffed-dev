<script lang="ts">
	// ── Type colours ──────────────────────────────────────────────
	const TYPE_COLORS: Record<string, string> = {
		normal:'#A8A77A', fire:'#EE8130', water:'#6390F0', electric:'#F7D02C',
		grass:'#7AC74C', ice:'#96D9D6', fighting:'#C22E28', poison:'#A33EA1',
		ground:'#E2BF65', flying:'#A98FF3', psychic:'#F95587', bug:'#A6B91A',
		rock:'#B6A136', ghost:'#735797', dragon:'#6F35FC', dark:'#705746',
		steel:'#B7B7CE', fairy:'#D685AD'
	};
	const ALL_TYPES = Object.keys(TYPE_COLORS);
	const MAX_LIVES = 3;

	// ── Pokémon pool ──────────────────────────────────────────────
	type PokeEntry = { id: number; name: string; types: string[]; sprite: string };

	let pool       = $state<PokeEntry[]>([]);
	let poolLoaded = $state(false);
	let poolError  = $state('');

	async function loadPool() {
		poolError = '';
		try {
			const res  = await fetch('https://pokeapi.co/api/v2/pokemon?limit=1025&offset=0');
			const data = await res.json();
			pool = data.results.map((p: any) => {
				const parts = p.url.split('/').filter(Boolean);
				const id = parseInt(parts[parts.length - 1]);
				return { id, name: p.name, types: [], sprite: '' };
			}).filter((p: any) => p.id <= 1025);
			poolLoaded = true;
			newRound();
		} catch {
			poolError = 'Failed to load Pokémon list. Check your connection.';
		}
	}

	// ── Game state ────────────────────────────────────────────────
	let current       = $state<PokeEntry | null>(null);
	let fetchingMon   = $state(false);
	let selected      = $state<string[]>([]);
	let submitted     = $state(false);
	let correct       = $state(false);
	let score         = $state(0);
	let streak        = $state(0);
	let bestStreak    = $state(0);
	let lives         = $state(MAX_LIVES);
	let gameOver      = $state(false);
	let roundNum      = $state(0);
	let advancing     = $state(false);

	const usedIds = new Set<number>();

	async function fetchMon(id: number): Promise<PokeEntry> {
		const res  = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
		if (!res.ok) throw new Error('fetch failed');
		const data = await res.json();
		return {
			id:     data.id,
			name:   data.name,
			types:  data.types.map((t: any) => t.type.name as string),
			sprite: data.sprites.front_default ?? ''
		};
	}

	async function newRound() {
		if (!poolLoaded || pool.length === 0) return;
		fetchingMon = true;
		selected    = [];
		submitted   = false;
		correct     = false;
		advancing   = false;

		let candidates = pool.filter(p => !usedIds.has(p.id));
		if (candidates.length === 0) { usedIds.clear(); candidates = pool; }
		const pick = candidates[Math.floor(Math.random() * candidates.length)];
		usedIds.add(pick.id);

		try {
			current   = await fetchMon(pick.id);
			roundNum++;
		} catch {
			usedIds.delete(pick.id);
			await newRound();
			return;
		}
		fetchingMon = false;
	}

	function toggle(type: string) {
		if (submitted) return;
		const maxSel = current?.types.length ?? 1;
		if (selected.includes(type)) {
			selected = selected.filter(t => t !== type);
		} else {
			if (selected.length >= maxSel) return;
			selected = [...selected, type];
		}
	}

	function submit() {
		if (!current || submitted || selected.length === 0) return;
		submitted = true;

		const ans   = [...current.types].sort().join(',');
		const guess = [...selected].sort().join(',');
		correct = ans === guess;

		if (correct) {
			const bonus = current.types.length === 2 ? 15 : 10;
			score  += bonus + streak * 2;
			streak++;
			if (streak > bestStreak) bestStreak = streak;
		} else {
			streak = 0;
			lives--;
		}

		advancing = true;

		if (lives <= 0) {
			setTimeout(() => { gameOver = true; advancing = false; }, 1400);
		} else {
			setTimeout(async () => {
				await newRound();
				advancing = false;
			}, correct ? 900 : 1800);
		}
	}

	function restart() {
		score    = 0;
		streak   = 0;
		lives    = MAX_LIVES;
		gameOver = false;
		roundNum = 0;
		usedIds.clear();
		newRound();
	}

	function fmt(s: string) {
		return s.replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
	}

	function missedTypes(): string[] {
		return (current?.types ?? []).filter(t => !selected.includes(t));
	}

	loadPool();
</script>

<svelte:head><title>Guess the Type — ScuffedDev</title></svelte:head>

<div class="game">

	<!-- Header -->
	<header class="hdr">
		<a href="/dev" class="hdr__back">← dev</a>
		<span class="hdr__title">guess-the-type</span>
		<div class="hdr__stats">
			<span class="stat"><span class="sl">score</span><span class="sv">{score}</span></span>
			<span class="stat"><span class="sl">streak</span><span class="sv">{streak > 0 ? streak + '🔥' : streak}</span></span>
			<span class="stat">
				<span class="sl">lives</span>
				<span class="sv lives-row">
					{#each Array(MAX_LIVES) as _, i}
						<span class="heart" class:lost={i >= lives}>♥</span>
					{/each}
				</span>
			</span>
		</div>
	</header>

	<!-- Game Over -->
	{#if gameOver}
	<div class="overlay">
		<div class="overlay__box">
			<p class="ov-label">game over</p>
			<p class="ov-score">{score}</p>
			<p class="ov-sub">final score · {roundNum - 1} rounds</p>
			<p class="ov-best">best streak: {bestStreak} 🔥</p>
			<button class="btn btn--lit" onclick={restart}>play again</button>
			<a href="/dev" class="btn">← back to dev</a>
		</div>
	</div>
	{/if}

	<!-- Loading state -->
	{#if !poolLoaded}
	<div class="ctr">
		{#if poolError}
			<p class="err-txt">{poolError}</p>
			<button class="btn" onclick={loadPool}>retry</button>
		{:else}
			<div class="spinner"></div>
			<p class="dim">Loading all 1025 Pokémon…</p>
		{/if}
	</div>

	{:else}

	<!-- Arena -->
	<div class="arena" class:ok={submitted && correct} class:bad={submitted && !correct}>

		<!-- Pokémon card -->
		<div class="mon-card">
			{#if fetchingMon || !current}
				<div class="spr-box"><div class="spinner"></div></div>
				<p class="mon-id">—</p>
			{:else}
				<div class="spr-box" class:spr-ok={submitted && correct} class:spr-bad={submitted && !correct}>
					<img src={current.sprite} alt={submitted ? fmt(current.name) : '???'} width="120" height="120" class="spr" />
				</div>

				{#if submitted}
					<p class="mon-name">{fmt(current.name)}</p>
					<div class="type-pills">
						{#each current.types as t}
							<span class="pill" style={`background:${TYPE_COLORS[t]}`}>{t}</span>
						{/each}
					</div>
					{#if correct}
						<p class="verdict v-ok">
							{current.types.length === 2 ? '🎯 both types!' : '✓ correct!'}
							{#if streak > 1}<span class="sb">+{(streak-1)*2} streak bonus</span>{/if}
						</p>
					{:else}
						<p class="verdict v-bad">
							{#if selected.some(t => current.types.includes(t)) && current.types.length === 2}
								got {selected.filter(t => current.types.includes(t)).length}/2 — missed {missedTypes().map(fmt).join(' + ')}
							{:else}
								✗ wrong
							{/if}
						</p>
					{/if}
				{:else}
					<p class="mon-id">#{String(current.id).padStart(4,'0')}</p>
					<p class="hint">
						{current.types.length === 2 ? 'dual type · select both' : 'single type · select one'}
					</p>
				{/if}
			{/if}
		</div>

		<!-- Slots -->
		{#if !submitted && current && !fetchingMon}
		<div class="slots">
			{#each Array(current.types.length) as _, i}
				<span class="slot" class:slot--on={selected[i] != null}
					style={selected[i] ? `border-color:${TYPE_COLORS[selected[i]]};color:#fff` : ''}>
					{selected[i] ? fmt(selected[i]) : '?'}
				</span>
				{#if i < current.types.length - 1}<span class="slot-sep">+</span>{/if}
			{/each}
		</div>
		{/if}

		<!-- Type grid -->
		<div class="type-grid">
			{#each ALL_TYPES as type}
				{@const isSel    = selected.includes(type)}
				{@const isRight  = submitted && current?.types.includes(type)}
				{@const isWrong  = submitted && selected.includes(type) && !current?.types.includes(type)}
				{@const isMissed = submitted && !selected.includes(type) && current?.types.includes(type)}
				{@const isMaxed  = !submitted && !isSel && selected.length >= (current?.types.length ?? 1)}
				<button
					class="tb"
					class:tb-sel={isSel}
					class:tb-right={isRight}
					class:tb-wrong={isWrong}
					class:tb-missed={isMissed}
					class:tb-maxed={isMaxed}
					style={`--tc:${TYPE_COLORS[type]}`}
					onclick={() => toggle(type)}
					disabled={submitted || (isMaxed && !isSel)}
					aria-pressed={isSel}
					aria-label={type}
				>{type}</button>
			{/each}
		</div>

		<!-- Submit -->
		<button
			class="sub"
			class:sub--on={selected.length > 0 && !submitted}
			onclick={submit}
			disabled={selected.length === 0 || submitted}
		>
			{#if submitted}
				{advancing ? (correct ? '✓ next up…' : '✗ next up…') : '—'}
			{:else if selected.length === 0}
				select a type to answer
			{:else if current && current.types.length === 2 && selected.length < 2}
				select second type ({selected.length}/2)
			{:else}
				confirm answer
			{/if}
		</button>

	</div>
	{/if}

</div>

<style>
.game {
	background: #2f2f2f;
	min-height: 100dvh;
	font-family: 'Cozette', monospace;
	color: #fff;
	display: flex;
	flex-direction: column;
}

/* ── Header ─────────────────────────────────────── */
.hdr {
	display: flex; align-items: center; justify-content: space-between;
	padding: 0.75rem 2rem;
	border-bottom: 1px solid rgba(255,255,255,0.06);
	flex-wrap: wrap; gap: 0.5rem;
}
.hdr__back { font-size: 0.7rem; color: rgba(255,255,255,0.3); text-decoration: none; transition: color 150ms; }
.hdr__back:hover { color: #fff; }
.hdr__title { font-size: 0.7rem; color: rgba(255,255,255,0.18); letter-spacing: 0.08em; }
.hdr__stats { display: flex; gap: 1.4rem; }
.stat { display: flex; flex-direction: column; align-items: center; gap: 0.05rem; }
.sl { font-size: 0.52rem; color: rgba(255,255,255,0.28); text-transform: uppercase; letter-spacing: 0.1em; }
.sv { font-size: 0.88rem; color: #fff; }
.lives-row { display: flex; gap: 0.12rem; }
.heart { color: #EE8130; transition: opacity 200ms; }
.heart.lost { opacity: 0.15; }

/* ── Util ───────────────────────────────────────── */
.ctr {
	flex: 1; display: flex; flex-direction: column;
	align-items: center; justify-content: center; gap: 1rem;
}
.dim { font-size: 0.75rem; color: rgba(255,255,255,0.25); }
.err-txt { font-size: 0.75rem; color: #EE8130; text-align: center; }
.spinner {
	width: 22px; height: 22px;
	border: 2px solid rgba(255,255,255,0.08);
	border-top-color: rgba(255,255,255,0.5);
	border-radius: 50%;
	animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── Overlay ────────────────────────────────────── */
.overlay {
	position: fixed; inset: 0; z-index: 60;
	background: rgba(0,0,0,0.72);
	backdrop-filter: blur(6px);
	display: flex; align-items: center; justify-content: center;
}
.overlay__box {
	background: #1c1c1c;
	border: 1px solid rgba(255,255,255,0.1);
	padding: 2.5rem 3rem;
	display: flex; flex-direction: column; align-items: center; gap: 0.5rem;
	text-align: center;
}
.ov-label { font-size: 0.6rem; color: rgba(255,255,255,0.28); text-transform: uppercase; letter-spacing: 0.12em; }
.ov-score { font-size: 3.5rem; line-height: 1; color: #fff; }
.ov-sub   { font-size: 0.68rem; color: rgba(255,255,255,0.3); }
.ov-best  { font-size: 0.72rem; color: rgba(255,255,255,0.48); margin-bottom: 0.6rem; }
.btn {
	font-family: 'Cozette', monospace;
	font-size: 0.73rem; padding: 0.5rem 1.2rem;
	border: 1px solid rgba(255,255,255,0.14);
	color: rgba(255,255,255,0.65); background: none;
	cursor: pointer; text-decoration: none; display: inline-block;
	transition: all 140ms; margin-top: 0.25rem;
}
.btn:hover { border-color: rgba(255,255,255,0.4); color: #fff; }
.btn--lit { border-color: rgba(255,255,255,0.38); color: #fff; }
.btn--lit:hover { background: rgba(255,255,255,0.06); }

/* ── Arena ──────────────────────────────────────── */
.arena {
	flex: 1; display: flex; flex-direction: column;
	align-items: center; padding: 1.5rem 1rem 2rem; gap: 1.1rem;
	transition: background 350ms;
}
.arena.ok  { background: rgba(122,199,76,0.04); }
.arena.bad { background: rgba(238,129,48,0.04); }

/* ── Mon card ───────────────────────────────────── */
.mon-card {
	display: flex; flex-direction: column; align-items: center; gap: 0.45rem;
	min-height: 200px;
}
.spr-box {
	padding: 0.85rem;
	background: rgba(255,255,255,0.025);
	border: 1px solid rgba(255,255,255,0.06);
	display: flex; align-items: center; justify-content: center;
	width: 148px; height: 148px;
	transition: border-color 300ms, background 300ms;
}
.spr-box.spr-ok  { border-color: rgba(122,199,76,0.45); background: rgba(122,199,76,0.07); }
.spr-box.spr-bad { border-color: rgba(238,129,48,0.4);  background: rgba(238,129,48,0.06); }
.spr { image-rendering: pixelated; display: block; }

.mon-id   { font-size: 0.7rem; color: rgba(255,255,255,0.3); }
.mon-name { font-size: 1rem; color: #fff; text-transform: capitalize; }
.hint     { font-size: 0.6rem; color: rgba(255,255,255,0.2); text-transform: uppercase; letter-spacing: 0.08em; }

.type-pills { display: flex; gap: 0.38rem; }
.pill {
	font-size: 0.63rem; padding: 0.18rem 0.55rem;
	border-radius: 2px; color: #fff; text-transform: capitalize;
}

.verdict { font-size: 0.75rem; display: flex; align-items: center; gap: 0.5rem; }
.v-ok  { color: #7AC74C; }
.v-bad { color: #EE8130; }
.sb {
	font-size: 0.6rem; padding: 0.1rem 0.35rem;
	border: 1px solid rgba(247,208,44,0.22); color: rgba(247,208,44,0.7);
}

/* ── Slots ──────────────────────────────────────── */
.slots { display: flex; align-items: center; gap: 0.5rem; }
.slot {
	font-size: 0.76rem; padding: 0.3rem 0.85rem;
	border: 1px dashed rgba(255,255,255,0.14);
	color: rgba(255,255,255,0.2); min-width: 90px; text-align: center;
	text-transform: capitalize; transition: border-color 150ms, color 150ms;
}
.slot--on { border-style: solid; }
.slot-sep { color: rgba(255,255,255,0.18); }

/* ── Type grid ──────────────────────────────────── */
.type-grid {
	display: grid;
	grid-template-columns: repeat(6, 1fr);
	gap: 0.3rem;
	width: 100%;
	max-width: 500px;
}

.tb {
	font-family: 'Cozette', monospace;
	font-size: 0.66rem; padding: 0.4rem 0.2rem;
	border: 1px solid rgba(255,255,255,0.07);
	color: rgba(255,255,255,0.5);
	background: rgba(255,255,255,0.02);
	cursor: pointer; text-transform: capitalize;
	transition: all 110ms; position: relative; overflow: hidden;
}
.tb::after {
	content: ''; position: absolute; inset: 0;
	background: var(--tc); opacity: 0; transition: opacity 110ms;
}
.tb:hover:not(:disabled)::after { opacity: 0.16; }
.tb:hover:not(:disabled) { border-color: var(--tc); color: #fff; }

.tb-sel {
	border-color: var(--tc);
	background: color-mix(in srgb, var(--tc) 20%, transparent);
	color: #fff;
}
.tb-right {
	border-color: var(--tc);
	background: color-mix(in srgb, var(--tc) 30%, transparent);
	color: #fff;
}
.tb-wrong {
	border-color: rgba(238,129,48,0.5);
	background: rgba(238,129,48,0.08);
	color: rgba(255,255,255,0.35);
	text-decoration: line-through;
}
.tb-missed {
	border-color: var(--tc);
	background: color-mix(in srgb, var(--tc) 18%, transparent);
	color: #fff;
	animation: pop 0.45s ease;
}
.tb-maxed { opacity: 0.28; cursor: not-allowed; }
.tb:disabled:not(.tb-sel):not(.tb-right):not(.tb-wrong):not(.tb-missed) { opacity: 0.42; }
.tb-sel::after, .tb-right::after, .tb-wrong::after, .tb-missed::after { opacity: 0 !important; }

@keyframes pop {
	0%   { transform: scale(1); }
	45%  { transform: scale(1.08); }
	100% { transform: scale(1); }
}

/* ── Submit ─────────────────────────────────────── */
.sub {
	font-family: 'Cozette', monospace;
	font-size: 0.76rem; padding: 0.6rem 2rem;
	border: 1px solid rgba(255,255,255,0.08);
	color: rgba(255,255,255,0.25);
	background: none; cursor: not-allowed;
	min-width: 230px; text-align: center;
	transition: all 140ms;
}
.sub--on {
	border-color: rgba(255,255,255,0.38);
	color: #fff; cursor: pointer;
}
.sub--on:hover {
	background: rgba(255,255,255,0.04);
	border-color: rgba(255,255,255,0.6);
}

/* ── Mobile ─────────────────────────────────────── */
@media (max-width: 480px) {
	.hdr { padding: 0.6rem 0.9rem; }
	.type-grid { grid-template-columns: repeat(3, 1fr); max-width: 100%; }
	.arena { padding: 1rem 0.75rem 1.5rem; }
}
</style>