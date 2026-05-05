<script lang="ts">
	import { onMount } from 'svelte';

	const typeColors: Record<string, string> = {
		Normal: '#A8A77A',
		Fire: '#EE8130',
		Water: '#6390F0',
		Electric: '#F7D02C',
		Grass: '#7AC74C',
		Ice: '#96D9D6',
		Fighting: '#C22E28',
		Poison: '#A33EA1',
		Ground: '#E2BF65',
		Flying: '#A98FF3',
		Psychic: '#F95587',
		Bug: '#A6B91A',
		Rock: '#B6A136',
		Ghost: '#735797',
		Dragon: '#6F35FC',
		Dark: '#705746',
		Steel: '#B7B7CE',
		Fairy: '#D685AD'
	};

	const allTypes = Object.keys(typeColors);

	// Full gen 1-9 super-effective chart (attacker → defending types it hits hard)
	const superEffectiveChart: Record<string, string[]> = {
		Normal:   [],
		Fire:     ['Grass','Ice','Bug','Steel'],
		Water:    ['Fire','Ground','Rock'],
		Electric: ['Water','Flying'],
		Grass:    ['Water','Ground','Rock'],
		Ice:      ['Grass','Ground','Flying','Dragon'],
		Fighting: ['Normal','Ice','Rock','Dark','Steel'],
		Poison:   ['Grass','Fairy'],
		Ground:   ['Fire','Electric','Poison','Rock','Steel'],
		Flying:   ['Grass','Fighting','Bug'],
		Psychic:  ['Fighting','Poison'],
		Bug:      ['Grass','Psychic','Dark'],
		Rock:     ['Fire','Ice','Flying','Bug'],
		Ghost:    ['Psychic','Ghost'],
		Dragon:   ['Dragon'],
		Dark:     ['Psychic','Ghost'],
		Steel:    ['Ice','Rock','Fairy'],
		Fairy:    ['Fighting','Dragon','Dark']
	};

	// ── Type Trainer teaser ───────────────────────────────────────
	let tqAttacker = $state('Fire');
	let tqSelected = $state<string[]>([]);
	let tqResult = $state<'correct' | 'wrong' | null>(null);
	let tqCorrect = $derived(superEffectiveChart[tqAttacker] ?? []);

	function newTQRound() {
		tqAttacker = allTypes[Math.floor(Math.random() * allTypes.length)];
		tqSelected = [];
		tqResult = null;
	}

	function answerType(type: string) {
		if (tqSelected.includes(type) || tqResult !== null) return;

		if (!tqCorrect.includes(type)) {
			tqResult = 'wrong';
			setTimeout(() => newTQRound(), 1200);
			return;
		}

		tqSelected = [...tqSelected, type];

		if (tqCorrect.length === 0 || tqSelected.length === tqCorrect.length) {
			tqResult = 'correct';
			setTimeout(() => newTQRound(), 900);
		}
	}

	// ── Articles ──────────────────────────────────────────────────
	const articles = [
		{ title: 'Building a Gen 3 Decomp ROM Hack from Scratch', date: 'Coming soon', tag: 'ROM Hacking' },
		{ title: 'DSPRE Basics: Editing Gen 4 Trainer Teams',      date: 'Coming soon', tag: 'ROM Hacking' },
		{ title: 'Team Building for VGC Doubles',                  date: 'Coming soon', tag: 'Competitive' },
		{ title: 'Understanding Type Matchups at a Deeper Level',  date: 'Coming soon', tag: 'Competitive' }
	];

	// ── Pokédex teaser ────────────────────────────────────────────
	let dexSprite = $state('https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/25.png');
	let dexName   = $state('Pikachu');
	let dexId     = $state(25);

	async function loadRandomPokemon() {
		const id = Math.floor(Math.random() * 151) + 1;
		dexId    = id;
		dexSprite = `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/${id}.png`;
		try {
			const res  = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
			const data = await res.json();
			dexName = data.name.charAt(0).toUpperCase() + data.name.slice(1);
		} catch { dexName = `#${id}`; }
	}

	// ── Guess the Type teaser ─────────────────────────────────────
	let guessSprite   = $state('');
	let guessTypes    = $state<string[]>([]);
	let guessOptions  = $state<string[]>([]);
	let guessSelected = $state<string[]>([]);
	let guessResult   = $state<'correct' | 'wrong' | null>(null);
	let guessLoading  = $state(true);

	async function loadGuessRound() {
		guessLoading  = true;
		guessResult   = null;
		guessSelected = [];

		const id = Math.floor(Math.random() * 151) + 1;
		guessSprite = `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/${id}.png`;

		try {
			const res  = await fetch(`https://pokeapi.co/api/v2/pokemon/${id}`);
			const data = await res.json();

			guessTypes = data.types.map(
				(t: any) => t.type.name.charAt(0).toUpperCase() + t.type.name.slice(1)
			);

			const wrong = allTypes
				.filter(t => !guessTypes.includes(t))
				.sort(() => Math.random() - 0.5)
				.slice(0, 6 - guessTypes.length);

			guessOptions = [...guessTypes, ...wrong].sort(() => Math.random() - 0.5);
		} catch {
			guessTypes   = ['Normal'];
			guessOptions = allTypes.sort(() => Math.random() - 0.5).slice(0, 6);
		}

		guessLoading = false;
	}

	function guessAnswer(type: string) {
		if (guessSelected.includes(type) || guessResult !== null) return;

		if (!guessTypes.includes(type)) {
			guessResult = 'wrong';
			setTimeout(() => loadGuessRound(), 1200);
			return;
		}

		guessSelected = [...guessSelected, type];

		if (guessSelected.length === guessTypes.length) {
			guessResult = 'correct';
			setTimeout(() => loadGuessRound(), 900);
		}
	}

	onMount(() => {
		newTQRound();
		loadRandomPokemon();
		loadGuessRound();
	});
</script>

<svelte:head>
	<title>ScuffedDev</title>
	<meta name="description" content="Pokémon tools, articles, and games by ScuffedDev." />
</svelte:head>

<div class="dev-page">

	<!-- ── Type Trainer ───────────────────────────────────────── -->
	<section class="section section--type">
		<div class="section__inner">
			<div class="section__text">
				<p class="tag">Game</p>
				<h2 class="section__title">Type Trainer</h2>
				<p class="section__desc">
					Master offensive and defensive type matchups.
					Two game modes, streak tracking, and instant feedback.
				</p>
				<a href="/dev/type-trainer" class="btn">Compete now &rarr;</a>
			</div>

			<div class="teaser">
				<p class="teaser__prompt">
					<span class="type-badge" style={`background:${typeColors[tqAttacker]}`}>
						{tqAttacker}
					</span>
					{#if tqCorrect.length === 0}
						hits nothing super effectively — select any type
					{:else if tqCorrect.length === 1}
						is super effective against? <span class="teaser__hint">(1 answer)</span>
					{:else}
						is super effective against? <span class="teaser__hint">({tqCorrect.length} answers)</span>
					{/if}
				</p>

				<div class="teaser__options teaser__options--all">
					{#each allTypes as type}
						<button
							class="type-btn"
							class:type-btn--selected={tqSelected.includes(type)}
							class:type-btn--correct={tqResult !== null && tqCorrect.includes(type)}
							class:type-btn--wrong={tqResult === 'wrong' && !tqCorrect.includes(type) && tqSelected.includes(type)}
							style={`--type-color: ${typeColors[type]}`}
							onclick={() => answerType(type)}
							disabled={tqResult !== null || tqSelected.includes(type)}
						>
							{type}
						</button>
					{/each}
				</div>

				{#if tqResult}
					<p class="teaser__feedback"
						class:feedback--correct={tqResult === 'correct'}
						class:feedback--wrong={tqResult === 'wrong'}>
						{tqResult === 'correct'
							? '✓ Correct!'
							: tqCorrect.length === 0
								? '✓ No super effective types!'
								: `✗ Super effective against: ${tqCorrect.join(', ')}`}
					</p>
				{/if}
			</div>
		</div>
	</section>

	<!-- ── Articles ───────────────────────────────────────────── -->
	<section class="section section--articles">
		<div class="section__inner">
			<div class="section__text">
				<p class="tag">Articles</p>
				<h2 class="section__title">Pokémon Writing</h2>
				<p class="section__desc">ROM hacking, competitive strategy, team building, and more.</p>
				<a href="/dev/articles" class="btn btn--ghost">Browse all &rarr;</a>
			</div>

			<div class="teaser teaser--articles">
				{#each articles as article}
					<div class="article-row">
						<div class="article-row__left">
							<span class="article-tag">{article.tag}</span>
							<p class="article-title">{article.title}</p>
						</div>
						<span class="article-date">{article.date}</span>
					</div>
				{/each}
			</div>
		</div>
	</section>

	<!-- ── Pokédex ────────────────────────────────────────────── -->
	<section class="section section--dex">
		<div class="section__inner">
			<div class="section__text">
				<p class="tag">Reference</p>
				<h2 class="section__title">Pokédex</h2>
				<p class="section__desc">
					Stats, moves, learnsets and type data pulled live from PokéAPI. Filter by generation.
				</p>
				<a href="/dev/pokedex" class="btn">Open Pokédex &rarr;</a>
			</div>

			<div class="teaser teaser--dex">
				<div class="dex-card">
					<img src={dexSprite} alt={dexName} width="96" height="96" style="image-rendering:pixelated;" />
					<p class="dex-name">{dexName}</p>
					<p class="dex-num">#{String(dexId).padStart(3, '0')}</p>
					<button class="btn btn--ghost btn--sm" onclick={loadRandomPokemon}>Random ↻</button>
				</div>
			</div>
		</div>
	</section>

	<!-- ── Guess the Type ─────────────────────────────────────── -->
	<section class="section section--guess">
		<div class="section__inner">
			<div class="section__text">
				<p class="tag">Game</p>
				<h2 class="section__title">Guess the Type</h2>
				<p class="section__desc">
					A Pokémon appears — what type is it? Test your Pokédex knowledge.
				</p>
				<a href="/dev/guess-the-type" class="btn">Play now &rarr;</a>
			</div>

			<div class="teaser teaser--guess">
				{#if guessLoading}
					<div class="guess-loading">Loading...</div>
				{:else}
					<img
						class="guess-sprite"
						src={guessSprite}
						alt="Guess this Pokémon's type"
						width="120"
						height="120"
						style="image-rendering:pixelated;"
					/>

					<p class="teaser__prompt">
						{guessTypes.length === 2 ? 'Pick both types' : 'What type is this?'}
					</p>

					<div class="teaser__options">
						{#each guessOptions as type}
							<button
								class="type-btn"
								class:type-btn--selected={guessSelected.includes(type)}
								class:type-btn--correct={guessResult !== null && guessTypes.includes(type)}
								class:type-btn--wrong={guessResult === 'wrong' && !guessTypes.includes(type) && guessSelected.includes(type)}
								style={`--type-color: ${typeColors[type]}`}
								onclick={() => guessAnswer(type)}
								disabled={guessResult !== null || guessSelected.includes(type)}
							>
								{type}
							</button>
						{/each}
					</div>

					{#if guessResult}
						<p class="teaser__feedback"
							class:feedback--correct={guessResult === 'correct'}
							class:feedback--wrong={guessResult === 'wrong'}>
							{guessResult === 'correct' ? '✓ Correct!' : `✗ It's ${guessTypes.join('/')}`}
						</p>
					{/if}
				{/if}
			</div>
		</div>
	</section>

</div>

<style>
.dev-page {
	background: #2f2f2f;
	min-height: 100dvh;
	font-family: 'Cozette', monospace;
	color: #ffffff;
}

.section {
	border-bottom: 1px solid rgba(255,255,255,0.06);
	padding: clamp(4rem, 8vw, 7rem) clamp(1.5rem, 6vw, 5rem);
}
.section:last-child { border-bottom: none; }

.section__inner {
	max-width: 1100px;
	margin-inline: auto;
	display: grid;
	grid-template-columns: 1fr 1fr;
	gap: 4rem;
	align-items: start;
}

.section:nth-child(even) .section__text { order: 2; }
.section:nth-child(even) .teaser        { order: 1; }

.tag {
	font-size: 0.6875rem;
	color: rgba(255,255,255,0.3);
	text-transform: uppercase;
	letter-spacing: 0.12em;
	margin-bottom: 0.75rem;
}
.section__title {
	font-size: clamp(1.75rem, 3vw, 2.75rem);
	font-weight: normal;
	line-height: 1.1;
	margin-bottom: 1rem;
}
.section__desc {
	font-size: 0.875rem;
	color: rgba(255,255,255,0.5);
	line-height: 1.7;
	max-width: 38ch;
	margin-bottom: 2rem;
}

/* Buttons */
.btn {
	display: inline-flex;
	align-items: center;
	font-family: 'Cozette', monospace;
	font-size: 0.8125rem;
	color: rgba(255,255,255,0.9);
	border: 1px solid rgba(255,255,255,0.2);
	padding: 0.5rem 1rem;
	text-decoration: none;
	transition: border-color 180ms ease, background 180ms ease;
	width: fit-content;
}
.btn:hover {
	border-color: rgba(255,255,255,0.5);
	background: rgba(255,255,255,0.04);
	color: #fff;
}
.btn--ghost { color: rgba(255,255,255,0.5); border-color: rgba(255,255,255,0.1); }
.btn--sm    { font-size: 0.75rem; padding: 0.35rem 0.75rem; }

/* Teaser shell */
.teaser {
	display: flex;
	flex-direction: column;
	gap: 1.25rem;
	padding: 2rem;
	background: rgba(255,255,255,0.03);
	border: 1px solid rgba(255,255,255,0.07);
}

.teaser__prompt {
	font-size: 0.8125rem;
	color: rgba(255,255,255,0.7);
	display: flex;
	align-items: center;
	gap: 0.5rem;
	flex-wrap: wrap;
}
.teaser__hint {
	font-size: 0.7rem;
	color: rgba(255,255,255,0.3);
}

.type-badge {
	font-size: 0.75rem;
	padding: 0.2rem 0.6rem;
	border-radius: 2px;
	color: #fff;
	letter-spacing: 0.04em;
}

/* Options grid — compact for all-18 display */
.teaser__options {
	display: flex;
	flex-wrap: wrap;
	gap: 0.4rem;
}
.teaser__options--all {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(72px, 1fr));
	gap: 0.4rem;
}

.type-btn {
	font-family: 'Cozette', monospace;
	font-size: 0.7rem;
	padding: 0.3rem 0.5rem;
	border: 1px solid rgba(255,255,255,0.15);
	color: rgba(255,255,255,0.8);
	background: rgba(255,255,255,0.03);
	cursor: pointer;
	transition: border-color 120ms ease, background 120ms ease;
	letter-spacing: 0.02em;
	text-align: center;
}
.type-btn:hover:not(:disabled) {
	border-color: var(--type-color);
	color: #fff;
	background: color-mix(in oklab, var(--type-color) 15%, transparent);
}
.type-btn--selected {
	border-color: var(--type-color) !important;
	background: color-mix(in oklab, var(--type-color) 20%, transparent) !important;
	color: #fff !important;
}
.type-btn--correct {
	border-color: #7AC74C !important;
	background: rgba(122,199,76,0.15) !important;
	color: #7AC74C !important;
}
.type-btn--wrong {
	border-color: #EE8130 !important;
	background: rgba(238,129,48,0.12) !important;
	color: #EE8130 !important;
}

.teaser__feedback { font-size: 0.75rem; letter-spacing: 0.05em; }
.feedback--correct { color: #7AC74C; }
.feedback--wrong   { color: #EE8130; }

/* Articles */
.teaser--articles { gap: 0; padding: 0; background: none; border: none; }
.article-row {
	display: flex;
	align-items: flex-start;
	justify-content: space-between;
	gap: 1rem;
	padding: 1rem 0;
	border-bottom: 1px solid rgba(255,255,255,0.06);
}
.article-row:first-child { border-top: 1px solid rgba(255,255,255,0.06); }
.article-row__left { display: flex; flex-direction: column; gap: 0.35rem; }
.article-tag   { font-size: 0.6875rem; color: rgba(255,255,255,0.3); text-transform: uppercase; letter-spacing: 0.1em; }
.article-title { font-size: 0.8125rem; color: rgba(255,255,255,0.75); max-width: 36ch; line-height: 1.5; }
.article-date  { font-size: 0.6875rem; color: rgba(255,255,255,0.25); white-space: nowrap; flex-shrink: 0; }

/* Pokédex */
.dex-card { display: flex; flex-direction: column; align-items: center; gap: 0.5rem; }
.dex-name { font-size: 1rem; color: #fff; letter-spacing: 0.03em; }
.dex-num  { font-size: 0.75rem; color: rgba(255,255,255,0.3); }

/* Guess */
.teaser--guess { align-items: flex-start; }
.guess-sprite  { image-rendering: pixelated; }
.guess-loading { font-size: 0.75rem; color: rgba(255,255,255,0.3); }

/* Mobile */
@media (max-width: 768px) {
	.section__inner { grid-template-columns: 1fr; gap: 2.5rem; }
	.section:nth-child(even) .section__text { order: 1; }
	.section:nth-child(even) .teaser        { order: 2; }
}
</style>