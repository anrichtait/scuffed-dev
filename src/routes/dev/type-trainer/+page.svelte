<script lang="ts">
	// ── Type data ─────────────────────────────────────────────────
	const typeColors: Record<string, string> = {
		Normal: '#A8A77A', Fire: '#EE8130', Water: '#6390F0',
		Electric: '#F7D02C', Grass: '#7AC74C', Ice: '#96D9D6',
		Fighting: '#C22E28', Poison: '#A33EA1', Ground: '#E2BF65',
		Flying: '#A98FF3', Psychic: '#F95587', Bug: '#A6B91A',
		Rock: '#B6A136', Ghost: '#735797', Dragon: '#6F35FC',
		Dark: '#705746', Steel: '#B7B7CE', Fairy: '#D685AD'
	};

	const allTypes = Object.keys(typeColors);

	// Offensive: attacker → types it hits super effectively
	const superEffective: Record<string, string[]> = {
		Normal: [], Fire: ['Grass','Ice','Bug','Steel'],
		Water: ['Fire','Ground','Rock'], Electric: ['Water','Flying'],
		Grass: ['Water','Ground','Rock'], Ice: ['Grass','Ground','Flying','Dragon'],
		Fighting: ['Normal','Ice','Rock','Dark','Steel'], Poison: ['Grass','Fairy'],
		Ground: ['Fire','Electric','Poison','Rock','Steel'], Flying: ['Grass','Fighting','Bug'],
		Psychic: ['Fighting','Poison'], Bug: ['Grass','Psychic','Dark'],
		Rock: ['Fire','Ice','Flying','Bug'], Ghost: ['Psychic','Ghost'],
		Dragon: ['Dragon'], Dark: ['Psychic','Ghost'],
		Steel: ['Ice','Rock','Fairy'], Fairy: ['Fighting','Dragon','Dark']
	};

	// Offensive: attacker → types it hits for 0.5x
	const notVeryEffective: Record<string, string[]> = {
		Normal: ['Rock','Steel'], Fire: ['Fire','Water','Rock','Dragon'],
		Water: ['Water','Grass','Dragon'], Electric: ['Electric','Grass','Dragon'],
		Grass: ['Fire','Grass','Poison','Flying','Bug','Dragon','Steel'],
		Ice: ['Water','Ice'], Fighting: ['Poison','Flying','Psychic','Bug','Fairy'],
		Poison: ['Poison','Ground','Rock','Ghost'], Ground: ['Grass','Bug'],
		Flying: ['Electric','Rock','Steel'], Psychic: ['Psychic','Steel'],
		Bug: ['Fire','Fighting','Flying','Ghost','Steel','Fairy'],
		Rock: ['Fighting','Ground','Steel'], Ghost: ['Dark'],
		Dragon: [], Dark: ['Fighting','Dark','Fairy'],
		Steel: ['Fire','Water','Electric','Steel'], Fairy: ['Fire','Poison','Steel']
	};

	// Offensive: attacker → types it cannot hit
	const noEffect: Record<string, string[]> = {
		Normal: ['Ghost'], Fire: [], Water: [], Electric: ['Ground'],
		Grass: [], Ice: [], Fighting: ['Ghost'], Poison: ['Steel'],
		Ground: ['Flying'], Flying: [], Psychic: ['Dark'], Bug: [],
		Rock: [], Ghost: ['Normal'], Dragon: ['Fairy'], Dark: [],
		Steel: [], Fairy: []
	};

	// Defensive: type → what it is immune to (takes 0 damage from)
	const immuneTo: Record<string, string[]> = {
		Normal: ['Ghost'], Fire: [], Water: [], Electric: [],
		Grass: [], Ice: [], Fighting: [], Poison: [],
		Ground: ['Electric'], Flying: ['Ground'], Psychic: [],
		Bug: [], Rock: [], Ghost: ['Normal','Fighting'],
		Dragon: [], Dark: ['Psychic'], Steel: ['Poison'],
		Fairy: ['Dragon']
	};

	// Defensive: type → what it resists (takes 0.5x from)
	const resistedBy: Record<string, string[]> = {
		Normal: [], Fire: ['Fire','Grass','Ice','Bug','Steel','Fairy'],
		Water: ['Fire','Water','Ice','Steel'], Electric: ['Electric','Flying','Steel'],
		Grass: ['Water','Electric','Grass','Ground'], Ice: ['Ice'],
		Fighting: ['Bug','Rock','Dark'], Poison: ['Grass','Fighting','Poison','Bug','Fairy'],
		Ground: ['Poison','Rock'], Flying: ['Grass','Fighting','Bug'],
		Psychic: ['Fighting','Psychic'], Bug: ['Grass','Fighting','Ground'],
		Rock: ['Normal','Fire','Poison','Flying'], Ghost: ['Poison','Bug'],
		Dragon: ['Fire','Water','Electric','Grass'], Dark: ['Ghost','Dark'],
		Steel: ['Normal','Grass','Ice','Flying','Psychic','Bug','Rock','Dragon','Steel','Fairy'],
		Fairy: ['Fighting','Bug','Dark']
	};

	// Defensive: type → what it is weak to (takes 2x from)
	const weakTo: Record<string, string[]> = {
		Normal: ['Fighting'], Fire: ['Water','Ground','Rock'],
		Water: ['Electric','Grass'], Electric: ['Ground'],
		Grass: ['Fire','Ice','Poison','Flying','Bug'], Ice: ['Fire','Fighting','Rock','Steel'],
		Fighting: ['Flying','Psychic','Fairy'], Poison: ['Ground','Psychic'],
		Ground: ['Water','Grass','Ice'], Flying: ['Electric','Ice','Rock'],
		Psychic: ['Bug','Ghost','Dark'], Bug: ['Fire','Flying','Rock'],
		Rock: ['Water','Grass','Fighting','Ground','Steel'], Ghost: ['Ghost','Dark'],
		Dragon: ['Ice','Dragon','Fairy'], Dark: ['Fighting','Bug','Fairy'],
		Steel: ['Fire','Fighting','Ground'], Fairy: ['Poison','Steel']
	};

	type QuestionType = 'superEffective' | 'notVeryEffective' | 'noEffect' | 'immuneTo' | 'resistedBy' | 'weakTo';

	const questionMeta: Record<QuestionType, { prompt: string; none: string }> = {
		superEffective:   { prompt: 'is super effective against',   none: 'nothing' },
		notVeryEffective: { prompt: 'is not very effective against', none: 'nothing' },
		noEffect:         { prompt: 'has no effect against',        none: 'nothing' },
		immuneTo:         { prompt: 'is immune to',                 none: 'no types' },
		resistedBy:       { prompt: 'resists',                      none: 'nothing' },
		weakTo:           { prompt: 'is weak to',                   none: 'nothing' }
	};

	const questionTypes = Object.keys(questionMeta) as QuestionType[];

	// ── Game state ────────────────────────────────────────────────
	const MAX_LIVES = 3;

	let currentType    = $state('Fire');
	let currentQ       = $state<QuestionType>('superEffective');
	let selected       = $state<string[]>([]);
	let noneSelected   = $state(false);
	let result         = $state<'correct' | 'wrong' | null>(null);
	let streak         = $state(0);
	let score          = $state(0);
	let lives          = $state(MAX_LIVES);
	let gameOver       = $state(false);
	let showAnswer     = $state(false);

	let correctAnswers = $derived(getCorrectAnswers(currentType, currentQ));
	let isNoneCorrect  = $derived(correctAnswers.length === 0);

	function getCorrectAnswers(type: string, q: QuestionType): string[] {
		const map = { superEffective, notVeryEffective, noEffect, immuneTo, resistedBy, weakTo };
		return map[q][type] ?? [];
	}

	function newRound() {
		currentType  = allTypes[Math.floor(Math.random() * allTypes.length)];
		currentQ     = questionTypes[Math.floor(Math.random() * questionTypes.length)];
		selected     = [];
		noneSelected = false;
		result       = null;
		showAnswer   = false;
	}

	function toggleType(type: string) {
		if (result !== null || noneSelected) return;
		if (selected.includes(type)) {
			selected = selected.filter(t => t !== type);
		} else {
			selected = [...selected, type];
		}
	}

	function toggleNone() {
		if (result !== null) return;
		noneSelected = !noneSelected;
		if (noneSelected) selected = [];
	}

	function submit() {
		if (result !== null) return;

		const allCorrectSelected = isNoneCorrect
			? noneSelected
			: correctAnswers.every(t => selected.includes(t)) &&
			  selected.every(t => correctAnswers.includes(t)) &&
			  !noneSelected;

		if (allCorrectSelected) {
			result = 'correct';
			score += 10 + streak * 2;
			streak++;
			setTimeout(() => newRound(), 1000);
		} else {
			result = 'wrong';
			streak = 0;
			lives--;
			showAnswer = true;
			if (lives <= 0) {
				setTimeout(() => { gameOver = true; }, 1200);
			} else {
				setTimeout(() => newRound(), 1800);
			}
		}
	}

	function restart() {
		score    = 0;
		lives    = MAX_LIVES;
		streak   = 0;
		gameOver = false;
		newRound();
	}

	function canSubmit(): boolean {
		return noneSelected || selected.length > 0;
	}
</script>

<svelte:head>
	<title>Type Trainer — ScuffedDev</title>
</svelte:head>

<div class="game">

	<!-- ── Header ─────────────────────────────────────────────── -->
	<header class="hdr">
		<a href="/dev" class="hdr__back">← dev</a>
		<span class="hdr__title">type-trainer</span>
		<div class="hdr__stats">
			<span class="stat">
				<span class="stat__label">score</span>
				<span class="stat__val">{score}</span>
			</span>
			<span class="stat">
				<span class="stat__label">streak</span>
				<span class="stat__val">{streak}</span>
			</span>
			<span class="stat">
				<span class="stat__label">lives</span>
				<span class="stat__val lives-val">
					{#each Array(MAX_LIVES) as _, i}
						<span class="life" class:life--lost={i >= lives}>♥</span>
					{/each}
				</span>
			</span>
		</div>
	</header>

	<!-- ── Game Over overlay ──────────────────────────────────── -->
	{#if gameOver}
		<div class="overlay">
			<div class="overlay__box">
				<p class="overlay__label">game over</p>
				<p class="overlay__score">{score}</p>
				<p class="overlay__sub">final score</p>
				<p class="overlay__streak">best streak this session: {streak}</p>
				<button class="btn" onclick={restart}>play again</button>
				<a href="/dev" class="btn btn--ghost">← back to dev</a>
			</div>
		</div>
	{/if}

	<!-- ── Question ───────────────────────────────────────────── -->
	<div class="question">
		<div class="question__type">
			<span class="type-badge" style={`background:${typeColors[currentType]}`}>
				{currentType}
			</span>
		</div>
		<p class="question__prompt">
			{questionMeta[currentQ].prompt}
			{#if isNoneCorrect}
				<span class="question__hint">(select "none")</span>
			{:else}
				<span class="question__hint">— select {correctAnswers.length === 1 ? '1 type' : `all ${correctAnswers.length} types`}</span>
			{/if}
		</p>
	</div>

	<!-- ── Type grid ──────────────────────────────────────────── -->
	<div class="grid-wrap">
		<div class="type-grid">
			{#each allTypes as type}
				<button
					class="type-btn"
					class:type-btn--selected={selected.includes(type)}
					class:type-btn--correct={showAnswer && correctAnswers.includes(type)}
					class:type-btn--wrong={result === 'wrong' && selected.includes(type) && !correctAnswers.includes(type)}
					class:type-btn--disabled={noneSelected}
					style={`--tc: ${typeColors[type]}`}
					onclick={() => toggleType(type)}
					disabled={result !== null || noneSelected}
				>
					{type}
				</button>
			{/each}

			<!-- None button -->
			<button
				class="type-btn type-btn--none"
				class:type-btn--selected={noneSelected}
				class:type-btn--correct={showAnswer && isNoneCorrect}
				class:type-btn--wrong={result === 'wrong' && noneSelected && !isNoneCorrect}
				style="--tc: rgba(255,255,255,0.25)"
				onclick={toggleNone}
				disabled={result !== null}
			>
				none
			</button>
		</div>
	</div>

	<!-- ── Feedback + Submit ───────────────────────────────────── -->
	<div class="footer">
		{#if result === 'correct'}
			<p class="feedback feedback--correct">✓ correct! +{10 + (streak - 1) * 2}</p>
		{:else if result === 'wrong'}
			<p class="feedback feedback--wrong">
				✗ wrong — {isNoneCorrect ? 'answer is none' : `answer: ${correctAnswers.join(', ')}`}
			</p>
		{:else}
			<p class="feedback feedback--idle">
				{selected.length > 0
					? `${selected.length} selected — submit when ready`
					: noneSelected
						? 'submitting "none"'
						: 'select all correct types then submit'}
			</p>
		{/if}

		<button
			class="btn btn--submit"
			onclick={submit}
			disabled={result !== null || !canSubmit()}
		>
			submit →
		</button>
	</div>

</div>

<style>
/* ── Shell ────────────────────────────────────────────────────── */
.game {
	background: #2f2f2f;
	min-height: 100dvh;
	height: 100dvh;
	display: grid;
	grid-template-rows: auto auto 1fr auto;
	font-family: 'Cozette', monospace;
	color: #fff;
	overflow: hidden;
}

/* ── Header ───────────────────────────────────────────────────── */
.hdr {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 0.75rem 1.5rem;
	border-bottom: 1px solid rgba(255,255,255,0.06);
	flex-wrap: wrap;
	gap: 0.5rem;
}
.hdr__back {
	font-size: 0.75rem;
	color: rgba(255,255,255,0.4);
	text-decoration: none;
	transition: color 150ms ease;
}
.hdr__back:hover { color: #fff; }
.hdr__title {
	font-size: 0.75rem;
	color: rgba(255,255,255,0.25);
	letter-spacing: 0.08em;
}
.hdr__stats {
	display: flex;
	gap: 1.5rem;
}
.stat {
	display: flex;
	flex-direction: column;
	align-items: flex-end;
}
.stat__label {
	font-size: 0.6rem;
	color: rgba(255,255,255,0.3);
	text-transform: uppercase;
	letter-spacing: 0.1em;
}
.stat__val {
	font-size: 0.9rem;
	color: #fff;
}
.lives-val { display: flex; gap: 0.2rem; }
.life { color: #EE8130; font-size: 0.85rem; transition: opacity 200ms ease; }
.life--lost { opacity: 0.15; }

/* ── Question ─────────────────────────────────────────────────── */
.question {
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 0.6rem;
	padding: 1.25rem 1.5rem 0.75rem;
}
.question__type { display: flex; align-items: center; gap: 0.75rem; }
.type-badge {
	font-size: 0.9rem;
	padding: 0.3rem 1rem;
	border-radius: 2px;
	color: #fff;
	letter-spacing: 0.05em;
}
.question__prompt {
	font-size: 0.8125rem;
	color: rgba(255,255,255,0.65);
	display: flex;
	align-items: center;
	gap: 0.5rem;
	flex-wrap: wrap;
	justify-content: center;
	text-align: center;
}
.question__hint {
	font-size: 0.7rem;
	color: rgba(255,255,255,0.3);
}

/* ── Grid ─────────────────────────────────────────────────────── */
.grid-wrap {
	display: flex;
	align-items: center;
	justify-content: center;
	padding: 0.5rem 1.5rem;
	overflow: hidden;
}

.type-grid {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(80px, 1fr));
	gap: 0.4rem;
	width: 100%;
	max-width: 900px;
}

.type-btn {
	font-family: 'Cozette', monospace;
	font-size: 0.72rem;
	padding: 0.45rem 0.3rem;
	border: 1px solid rgba(255,255,255,0.1);
	color: rgba(255,255,255,0.75);
	background: rgba(255,255,255,0.03);
	cursor: pointer;
	transition: border-color 100ms ease, background 100ms ease, color 100ms ease;
	letter-spacing: 0.03em;
	text-align: center;
	user-select: none;
}
.type-btn:hover:not(:disabled):not(.type-btn--disabled) {
	border-color: var(--tc);
	color: #fff;
	background: color-mix(in oklab, var(--tc) 15%, transparent);
}
.type-btn--selected {
	border-color: var(--tc) !important;
	background: color-mix(in oklab, var(--tc) 22%, transparent) !important;
	color: #fff !important;
}
.type-btn--correct {
	border-color: #7AC74C !important;
	background: rgba(122,199,76,0.18) !important;
	color: #7AC74C !important;
}
.type-btn--wrong {
	border-color: #EE8130 !important;
	background: rgba(238,129,48,0.15) !important;
	color: #EE8130 !important;
}
.type-btn--disabled {
	opacity: 0.3;
	cursor: default;
}
.type-btn--none {
	border-style: dashed;
	color: rgba(255,255,255,0.4);
}

/* ── Footer ───────────────────────────────────────────────────── */
.footer {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 0.75rem 1.5rem;
	border-top: 1px solid rgba(255,255,255,0.06);
	gap: 1rem;
	flex-wrap: wrap;
}

.feedback {
	font-size: 0.75rem;
	letter-spacing: 0.04em;
	flex: 1;
	min-width: 0;
}
.feedback--correct { color: #7AC74C; }
.feedback--wrong   { color: #EE8130; }
.feedback--idle    { color: rgba(255,255,255,0.3); }

.btn {
	font-family: 'Cozette', monospace;
	font-size: 0.8125rem;
	color: rgba(255,255,255,0.9);
	border: 1px solid rgba(255,255,255,0.2);
	padding: 0.5rem 1rem;
	text-decoration: none;
	display: inline-flex;
	align-items: center;
	cursor: pointer;
	background: none;
	transition: border-color 150ms ease, background 150ms ease;
	white-space: nowrap;
}
.btn:hover:not(:disabled) {
	border-color: rgba(255,255,255,0.5);
	background: rgba(255,255,255,0.04);
}
.btn:disabled { opacity: 0.35; cursor: default; }
.btn--ghost { color: rgba(255,255,255,0.4); border-color: rgba(255,255,255,0.1); }
.btn--submit { border-color: rgba(255,255,255,0.3); }

/* ── Game Over overlay ────────────────────────────────────────── */
.overlay {
	position: fixed;
	inset: 0;
	background: rgba(0,0,0,0.75);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 100;
	backdrop-filter: blur(4px);
}
.overlay__box {
	background: #1e1e1e;
	border: 1px solid rgba(255,255,255,0.1);
	padding: 3rem 3.5rem;
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 0.75rem;
	text-align: center;
}
.overlay__label  { font-size: 0.75rem; color: rgba(255,255,255,0.3); text-transform: uppercase; letter-spacing: 0.12em; }
.overlay__score  { font-size: clamp(3rem, 8vw, 5rem); line-height: 1; color: #fff; }
.overlay__sub    { font-size: 0.75rem; color: rgba(255,255,255,0.35); margin-bottom: 0.5rem; }
.overlay__streak { font-size: 0.75rem; color: rgba(255,255,255,0.4); margin-bottom: 1rem; }

/* ── Mobile ───────────────────────────────────────────────────── */
@media (max-width: 600px) {
	.type-grid { grid-template-columns: repeat(auto-fill, minmax(64px, 1fr)); }
	.hdr__stats { gap: 1rem; }
	.overlay__box { padding: 2rem 1.5rem; }
}
</style>