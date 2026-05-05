<script lang="ts">
	// ── Constants ─────────────────────────────────────────────────
	const GENS: Record<string, { label: string; max: number; vg: string }> = {
		'4': { label: 'Gen IV', max: 493, vg: 'diamond-pearl' },
		'9': { label: 'Gen IX', max: 1025, vg: 'scarlet-violet' }
	};

	const TYPE_COLORS: Record<string, string> = {
		normal:'#A8A77A', fire:'#EE8130', water:'#6390F0', electric:'#F7D02C',
		grass:'#7AC74C', ice:'#96D9D6', fighting:'#C22E28', poison:'#A33EA1',
		ground:'#E2BF65', flying:'#A98FF3', psychic:'#F95587', bug:'#A6B91A',
		rock:'#B6A136', ghost:'#735797', dragon:'#6F35FC', dark:'#705746',
		steel:'#B7B7CE', fairy:'#D685AD'
	};

	const STAT_LABELS: Record<string, string> = {
		hp:'HP', attack:'Atk', defense:'Def',
		'special-attack':'SpA', 'special-defense':'SpD', speed:'Spe'
	};

	const DMGCLASS_COLORS: Record<string, string> = {
		physical: '#EE8130', special: '#6390F0', status: '#A8A77A'
	};

	// ── Types ─────────────────────────────────────────────────────
	type Category = 'pokemon' | 'move' | 'item' | 'ability';
	type Entry    = { name: string; id: number; cat: Category };

	// ── State ─────────────────────────────────────────────────────
	let selectedGen  = $state<'4'|'9'>('9');
	let query        = $state('');
	let suggestions  = $state<Entry[]>([]);
	let showDrop     = $state(false);
	let loadingList  = $state(true);

	// index lists
	let allPokemon   = $state<Entry[]>([]);
	let allMoves     = $state<Entry[]>([]);
	let allItems     = $state<Entry[]>([]);
	let allAbilities = $state<Entry[]>([]);

	// detail state
	let detail       = $state<any>(null);
	let detailCat    = $state<Category | null>(null);
	let loading      = $state(false);
	let error        = $state('');
	let fetchId      = 0;

	// pokemon-specific
	let species      = $state<any>(null);
	let evoChain     = $state<any[]>([]);
	let levelMoves   = $state<any[]>([]);
	let tmMoves      = $state<any[]>([]);
	let shiny        = $state(false);
	let shinyUnlocked = $state(false);
	let activeTab    = $state<'level'|'tm'|'learned'>('level');

	// navigation stack for back button
	let history      = $state<{cat: Category; name: string}[]>([]);

	// ── Fetch index lists ─────────────────────────────────────────
	async function loadAll(gen: '4'|'9') {
		loadingList = true;
		const max = GENS[gen].max;
		const [pRes, mRes, iRes, aRes] = await Promise.all([
			fetch(`https://pokeapi.co/api/v2/pokemon?limit=${max}&offset=0`),
			fetch(`https://pokeapi.co/api/v2/move?limit=937&offset=0`),
			fetch(`https://pokeapi.co/api/v2/item?limit=2176&offset=0`),
			fetch(`https://pokeapi.co/api/v2/ability?limit=371&offset=0`)
		]);
		const [pData, mData, iData, aData] = await Promise.all([
			pRes.json(), mRes.json(), iRes.json(), aRes.json()
		]);

		allPokemon = pData.results
			.map((p: any) => {
				const parts = p.url.split('/').filter(Boolean);
				return { name: p.name, id: parseInt(parts[parts.length-1]), cat: 'pokemon' as Category };
			})
			.filter((p: any) => p.id <= max);

		allMoves     = mData.results.map((m: any, i: number) => ({ name: m.name, id: i+1, cat: 'move'     as Category }));
		allItems     = iData.results.map((m: any, i: number) => ({ name: m.name, id: i+1, cat: 'item'     as Category }));
		allAbilities = aData.results.map((m: any, i: number) => ({ name: m.name, id: i+1, cat: 'ability'  as Category }));

		loadingList = false;
	}

	// ── Search ────────────────────────────────────────────────────
	function onInput() {
		const q = query.trim().toLowerCase();
		if (q.length < 2) { suggestions = []; showDrop = false; return; }
		const allEntries = [...allPokemon, ...allMoves, ...allItems, ...allAbilities];
		suggestions = allEntries
			.filter(e => e.name.includes(q) || (e.cat === 'pokemon' && String(e.id).padStart(3,'0').startsWith(q)))
			.slice(0, 16);
		showDrop = suggestions.length > 0;
	}

	const CAT_LABELS: Record<Category, string> = {
		pokemon: 'mon', move: 'move', item: 'item', ability: 'ability'
	};

	// ── Navigate to detail ────────────────────────────────────────
	async function open(cat: Category, name: string, pushHistory = true) {
		if (pushHistory && detail) {
			history = [...history, { cat: detailCat!, name: detail.name }];
		}
		query    = '';
		showDrop = false;
		loading  = true;
		error    = '';
		detail   = null;
		detailCat = cat;
		species  = null;
		evoChain = [];
		levelMoves = []; tmMoves = [];
		shiny    = false; shinyUnlocked = false;
		activeTab = cat === 'move' ? 'learned' : 'level';

		const thisId = ++fetchId;

		try {
			if (cat === 'pokemon') await loadPokemon(name, thisId);
			else if (cat === 'move') await loadMove(name, thisId);
			else if (cat === 'item') await loadItem(name, thisId);
			else if (cat === 'ability') await loadAbility(name, thisId);
		} catch(e: any) {
			if (thisId === fetchId) { error = e.message ?? 'Error loading data'; loading = false; }
			return;
		}
		if (thisId === fetchId) loading = false;
	}

	async function loadPokemon(name: string, thisId: number) {
		const vg = GENS[selectedGen].vg;
		const pRes = await fetch(`https://pokeapi.co/api/v2/pokemon/${name.toLowerCase()}`);
		if (!pRes.ok) throw new Error(`"${name}" not found`);
		const pData = await pRes.json();
		if (thisId !== fetchId) return;

		const genMoves = pData.moves.filter((m: any) =>
			m.version_group_details.some((v: any) => v.version_group.name === vg)
		);
		levelMoves = genMoves
			.filter((m: any) => m.version_group_details.some((v: any) =>
				v.version_group.name === vg && v.move_learn_method.name === 'level-up'
			))
			.map((m: any) => {
				const d = m.version_group_details.find((v: any) =>
					v.version_group.name === vg && v.move_learn_method.name === 'level-up'
				);
				return { name: m.move.name, level: d?.level_learned_at ?? 0 };
			})
			.sort((a: any, b: any) => a.level - b.level);

		tmMoves = genMoves
			.filter((m: any) => m.version_group_details.some((v: any) =>
				v.version_group.name === vg && v.move_learn_method.name === 'machine'
			))
			.map((m: any) => ({ name: m.move.name }))
			.sort((a: any, b: any) => a.name.localeCompare(b.name));

		const [sRes] = await Promise.all([fetch(pData.species.url)]);
		if (!sRes.ok) throw new Error('Species data unavailable');
		const sData = await sRes.json();
		if (thisId !== fetchId) return;

		const evoRes = await fetch(sData.evolution_chain.url);
		const evoData = await evoRes.json();
		if (thisId !== fetchId) return;

		detail   = pData;
		species  = sData;
		evoChain = flattenEvoChain(evoData.chain);

		if (Math.random() < 1/8192) { shinyUnlocked = true; shiny = true; }
	}

	async function loadMove(name: string, thisId: number) {
		const res = await fetch(`https://pokeapi.co/api/v2/move/${name.toLowerCase()}`);
		if (!res.ok) throw new Error(`Move "${name}" not found`);
		const data = await res.json();
		if (thisId !== fetchId) return;
		detail = data;
	}

	async function loadItem(name: string, thisId: number) {
		const res = await fetch(`https://pokeapi.co/api/v2/item/${name.toLowerCase()}`);
		if (!res.ok) throw new Error(`Item "${name}" not found`);
		const data = await res.json();
		if (thisId !== fetchId) return;
		detail = data;
	}

	async function loadAbility(name: string, thisId: number) {
		const res = await fetch(`https://pokeapi.co/api/v2/ability/${name.toLowerCase()}`);
		if (!res.ok) throw new Error(`Ability "${name}" not found`);
		const data = await res.json();
		if (thisId !== fetchId) return;
		detail = data;
	}

	// ── Back navigation ───────────────────────────────────────────
	function goBack() {
		if (history.length > 0) {
			const prev = history[history.length - 1];
			history = history.slice(0, -1);
			open(prev.cat, prev.name, false);
		} else {
			detail = null; detailCat = null; error = '';
		}
	}

	// ── Gen switch ────────────────────────────────────────────────
	function switchGen(g: '4'|'9') {
		if (selectedGen === g) return;
		selectedGen = g;
		loadAll(g);
		if (detail && detailCat === 'pokemon') open('pokemon', detail.name, false);
	}

	// ── Helpers ───────────────────────────────────────────────────
	function flattenEvoChain(node: any, chain: any[] = []): any[] {
		chain.push({ name: node.species.name, details: node.evolution_details?.[0] ?? null });
		for (const next of (node.evolves_to ?? [])) flattenEvoChain(next, chain);
		return chain;
	}

	function evoMethod(d: any): string {
		if (!d) return '';
		if (d.min_level)              return `Lv. ${d.min_level}`;
		if (d.item)                   return `Use ${fmt(d.item.name)}`;
		if (d.trigger?.name === 'trade') return d.held_item ? `Trade w/ ${fmt(d.held_item.name)}` : 'Trade';
		if (d.min_happiness)          return `Friendship${d.time_of_day ? ` (${d.time_of_day})` : ''}`;
		if (d.known_move)             return `Know ${fmt(d.known_move.name)}`;
		if (d.location)               return `At ${fmt(d.location.name)}`;
		if (d.min_beauty)             return `Beauty ${d.min_beauty}`;
		return fmt(d.trigger?.name ?? '');
	}

	function fmt(s: string): string {
		return (s ?? '').replace(/-/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
	}

	function engEffect(entries: any[]): string {
		return entries?.find((e: any) => e.language.name === 'en')?.short_effect ?? '—';
	}

	function engFlavorText(entries: any[]): string {
		const en = entries?.filter((e: any) => e.language.name === 'en') ?? [];
		return en[en.length - 1]?.flavor_text?.replace(/\f/g, ' ') ?? '';
	}

	function statPct(v: number) { return Math.min(100, Math.round((v/255)*100)); }
	function statColor(v: number) { return v >= 100 ? '#7AC74C' : v >= 70 ? '#F7D02C' : '#EE8130'; }
	function spriteUrl(p: any) {
		return shiny
			? (p.sprites?.front_shiny ?? p.sprites?.front_default ?? '')
			: (p.sprites?.front_default ?? '');
	}

	// Limit learned-by list to avoid huge renders
	function learnedByInGen(move: any): string[] {
		return (move.learned_by_pokemon ?? [])
			.map((p: any) => p.pokemon.name)
			.filter((n: string) => {
				const entry = allPokemon.find(p => p.name === n);
				return entry !== undefined;
			})
			.slice(0, 60);
	}

	// ── Init ─────────────────────────────────────────────────────
	loadAll(selectedGen);
</script>

<svelte:head><title>Pokédex — ScuffedDev</title></svelte:head>

<div class="page">

	<!-- Header -->
	<header class="hdr">
		<a href="/dev" class="hdr__back">← dev</a>
		<span class="hdr__title">pokédex</span>
		<div class="gen-tabs">
			{#each Object.entries(GENS) as [key, val]}
				<button class="gen-tab" class:gen-tab--on={selectedGen===key} onclick={() => switchGen(key as '4'|'9')}>
					{val.label}
				</button>
			{/each}
		</div>
	</header>

	<!-- Search bar always visible -->
	<div class="search-bar">
		<div class="search-input-wrap">
			<input
				class="search-input"
				type="text"
				placeholder={loadingList ? 'Loading…' : 'Search Pokémon, moves, items, abilities…'}
				bind:value={query}
				oninput={onInput}
				onfocus={onInput}
				onblur={() => setTimeout(() => showDrop = false, 160)}
				disabled={loadingList}
				autocomplete="off"
				spellcheck="false"
			/>
			{#if showDrop}
				<div class="dropdown">
					{#each suggestions as s}
						<button class="dd-item" onclick={() => open(s.cat, s.name)}>
							<span class="dd-cat dd-cat--{s.cat}">{CAT_LABELS[s.cat]}</span>
							{#if s.cat === 'pokemon'}
								<span class="dd-num">#{String(s.id).padStart(3,'0')}</span>
							{/if}
							<span class="dd-name">{fmt(s.name)}</span>
						</button>
					{/each}
				</div>
			{/if}
		</div>
		<button
			class="btn"
			onclick={() => {
				if (!allPokemon.length) return;
				const p = allPokemon[Math.floor(Math.random()*allPokemon.length)];
				open('pokemon', p.name);
			}}
			disabled={loadingList}
		>↻ random</button>
	</div>

	<!-- Empty state -->
	{#if !detail && !loading}
		<div class="empty-state">
			{#if error}
				<p class="error-txt">⚠ {error}</p>
			{:else}
				<p class="empty-hint">{loadingList ? `Loading ${GENS[selectedGen].label}…` : `${allPokemon.length} Pokémon · ${allMoves.length} moves · ${allItems.length} items · ${allAbilities.length} abilities`}</p>
			{/if}
		</div>
	{/if}

	<!-- Loading -->
	{#if loading}
		<div class="loading-state">
			<div class="spinner"></div>
			<p>Loading…</p>
		</div>
	{/if}

	<!-- Detail panel -->
	{#if detail && !loading}
	<div class="detail">
		<div class="detail__topbar">
			<button class="btn" onclick={goBack}>
				{history.length > 0 ? `← ${fmt(history[history.length-1].name)}` : '← back'}
			</button>
			<div class="detail__topbar-right">
				{#if detailCat === 'pokemon' && shinyUnlocked}
					<button class="btn btn--shiny" class:btn--shiny-on={shiny} onclick={() => shiny=!shiny}>✦ shiny</button>
				{/if}
				<span class="cat-badge cat-badge--{detailCat}">{CAT_LABELS[detailCat!]}</span>
				<span class="gen-badge">{GENS[selectedGen].label}</span>
			</div>
		</div>

		<!-- ── Pokémon detail ──────────────────────────────────── -->
		{#if detailCat === 'pokemon'}
		<div class="detail__body">
			<div class="col-left">
				<div class="sprite-box" class:sprite-box--shiny={shiny}>
					<img src={spriteUrl(detail)} alt={fmt(detail.name)} width="128" height="128" />
				</div>
				<div>
					<h1 class="mon-name">{fmt(detail.name)}</h1>
					<p class="mon-num">#{String(detail.id).padStart(3,'0')}</p>
				</div>
				<div class="type-row">
					{#each detail.types as t}
						<span class="type-badge" style={`background:${TYPE_COLORS[t.type.name]}`}>{t.type.name}</span>
					{/each}
				</div>

				<div class="stats">
					{#each detail.stats as s}
						<div class="stat-row">
							<span class="sl">{STAT_LABELS[s.stat.name] ?? s.stat.name}</span>
							<span class="sv">{s.base_stat}</span>
							<div class="bar-bg"><div class="bar-fill" style={`width:${statPct(s.base_stat)}%;background:${statColor(s.base_stat)}`}></div></div>
						</div>
					{/each}
					<div class="stat-row stat-row--bst">
						<span class="sl">BST</span>
						<span class="sv">{detail.stats.reduce((a:number,s:any)=>a+s.base_stat,0)}</span>
					</div>
				</div>

				<div class="block">
					<p class="bt">Abilities</p>
					{#each detail.abilities as a}
						<button class="link-btn" onclick={() => open('ability', a.ability.name)}>
							{fmt(a.ability.name)}{#if a.is_hidden} <span class="tag">hidden</span>{/if}
						</button>
					{/each}
				</div>

				{#if species}
				<div class="block">
					<p class="bt">Profile</p>
					<div class="kv">
						<span class="k">Height</span><span class="v">{(detail.height/10).toFixed(1)}m</span>
						<span class="k">Weight</span><span class="v">{(detail.weight/10).toFixed(1)}kg</span>
						<span class="k">Capture</span><span class="v">{species.capture_rate}</span>
						<span class="k">Base Exp</span><span class="v">{detail.base_experience ?? '—'}</span>
						<span class="k">Growth</span><span class="v">{fmt(species.growth_rate?.name ?? '—')}</span>
					</div>
				</div>
				{/if}
			</div>

			<div class="col-right">
				{#if evoChain.length > 1}
				<div class="block">
					<p class="bt">Evolution Chain</p>
					<div class="evo-chain">
						{#each evoChain as evo, i}
							<div class="evo-step">
								{#if i > 0 && evo.details}
									<span class="evo-arrow">→ {evoMethod(evo.details)}</span>
								{/if}
								<button class="evo-btn" class:evo-btn--on={evo.name===detail.name} onclick={() => open('pokemon', evo.name)}>
									{fmt(evo.name)}
								</button>
							</div>
						{/each}
					</div>
				</div>
				{/if}

				<div class="block moves-block">
					<div class="tabs">
						<button class="tab" class:tab--on={activeTab==='level'} onclick={() => activeTab='level'}>Level-up ({levelMoves.length})</button>
						<button class="tab" class:tab--on={activeTab==='tm'}    onclick={() => activeTab='tm'}>TM/HM ({tmMoves.length})</button>
					</div>
					<div class="list-scroll">
						{#if activeTab === 'level'}
							{#if !levelMoves.length}<p class="empty">No level-up moves in {GENS[selectedGen].label}</p>
							{:else}
								<div class="row-hdr"><span>Lv.</span><span>Move</span></div>
								{#each levelMoves as m}
									<div class="move-row">
										<span class="move-lv">{m.level||'—'}</span>
										<button class="link-btn move-name" onclick={() => open('move', m.name)}>{fmt(m.name)}</button>
									</div>
								{/each}
							{/if}
						{:else}
							{#if !tmMoves.length}<p class="empty">No TM/HM moves in {GENS[selectedGen].label}</p>
							{:else}
								{#each tmMoves as m}
									<div class="move-row move-row--tm">
										<button class="link-btn move-name" onclick={() => open('move', m.name)}>{fmt(m.name)}</button>
									</div>
								{/each}
							{/if}
						{/if}
					</div>
				</div>
			</div>
		</div>

		<!-- ── Move detail ─────────────────────────────────────── -->
		{:else if detailCat === 'move'}
		<div class="detail__body">
			<div class="col-left">
				<div>
					<h1 class="mon-name">{fmt(detail.name)}</h1>
				</div>
				<div class="type-row">
					<span class="type-badge" style={`background:${TYPE_COLORS[detail.type?.name]??'#888'}`}>{detail.type?.name}</span>
					<span class="type-badge" style={`background:${DMGCLASS_COLORS[detail.damage_class?.name]??'#888'}`}>{detail.damage_class?.name}</span>
				</div>

				<div class="block">
					<p class="bt">Stats</p>
					<div class="kv">
						<span class="k">Power</span>    <span class="v">{detail.power ?? '—'}</span>
						<span class="k">Accuracy</span> <span class="v">{detail.accuracy != null ? detail.accuracy+'%' : '—'}</span>
						<span class="k">PP</span>        <span class="v">{detail.pp ?? '—'}</span>
						<span class="k">Priority</span> <span class="v">{detail.priority ?? 0}</span>
						<span class="k">Target</span>   <span class="v">{fmt(detail.target?.name ?? '—')}</span>
					</div>
				</div>

				{#if detail.meta}
				<div class="block">
					<p class="bt">Battle Data</p>
					<div class="kv">
						{#if detail.meta.ailment?.name && detail.meta.ailment.name !== 'none'}
							<span class="k">Ailment</span><span class="v">{fmt(detail.meta.ailment.name)}</span>
						{/if}
						{#if detail.meta.drain}
							<span class="k">Drain</span><span class="v">{detail.meta.drain}%</span>
						{/if}
						{#if detail.meta.healing}
							<span class="k">Healing</span><span class="v">{detail.meta.healing}%</span>
						{/if}
						{#if detail.meta.crit_rate}
							<span class="k">Crit Rate</span><span class="v">+{detail.meta.crit_rate}</span>
						{/if}
						{#if detail.meta.flinch_chance}
							<span class="k">Flinch</span><span class="v">{detail.meta.flinch_chance}%</span>
						{/if}
						{#if detail.meta.stat_chance}
							<span class="k">Stat chance</span><span class="v">{detail.meta.stat_chance}%</span>
						{/if}
						{#if detail.meta.min_hits}
							<span class="k">Hits</span><span class="v">{detail.meta.min_hits}–{detail.meta.max_hits}</span>
						{/if}
					</div>
				</div>
				{/if}

				<div class="block">
					<p class="bt">Effect</p>
					<p class="effect-text">{engEffect(detail.effect_entries)}</p>
				</div>
			</div>

			<div class="col-right">
				<div class="block moves-block">
					<p class="bt">Learned by ({learnedByInGen(detail).length}{learnedByInGen(detail).length===60?'+':''})</p>
					<div class="list-scroll list-scroll--grid">
						{#each learnedByInGen(detail) as name}
							<button class="grid-btn" onclick={() => open('pokemon', name)}>{fmt(name)}</button>
						{/each}
					</div>
				</div>
			</div>
		</div>

		<!-- ── Item detail ─────────────────────────────────────── -->
		{:else if detailCat === 'item'}
		<div class="detail__body">
			<div class="col-left">
				<div class="sprite-box sprite-box--item">
					{#if detail.sprites?.default}
						<img src={detail.sprites.default} alt={fmt(detail.name)} width="64" height="64" />
					{:else}
						<span style="font-size:2rem">?</span>
					{/if}
				</div>
				<div>
					<h1 class="mon-name">{fmt(detail.name)}</h1>
					<p class="mon-num">{fmt(detail.category?.name ?? '')}</p>
				</div>

				<div class="block">
					<p class="bt">Data</p>
					<div class="kv">
						<span class="k">Cost</span>        <span class="v">{detail.cost ? `₽${detail.cost}` : '—'}</span>
						{#if detail.fling_power != null}
							<span class="k">Fling Power</span><span class="v">{detail.fling_power}</span>
						{/if}
						{#if detail.fling_effect?.name}
							<span class="k">Fling Effect</span><span class="v">{fmt(detail.fling_effect.name)}</span>
						{/if}
					</div>
				</div>

				<div class="block">
					<p class="bt">Effect</p>
					<p class="effect-text">{engEffect(detail.effect_entries)}</p>
				</div>

				{#if detail.flavor_text_entries?.length}
				<div class="block">
					<p class="bt">Description</p>
					<p class="effect-text">{engFlavorText(detail.flavor_text_entries)}</p>
				</div>
				{/if}

				{#if detail.attributes?.length}
				<div class="block">
					<p class="bt">Attributes</p>
					<div class="tag-row">
						{#each detail.attributes as a}
							<span class="attr-tag">{fmt(a.name)}</span>
						{/each}
					</div>
				</div>
				{/if}
			</div>

			<div class="col-right">
				{#if detail.held_by_pokemon?.length}
				<div class="block moves-block">
					<p class="bt">Held by ({detail.held_by_pokemon.length})</p>
					<div class="list-scroll list-scroll--grid">
						{#each detail.held_by_pokemon as h}
							<button class="grid-btn" onclick={() => open('pokemon', h.pokemon.name)}>{fmt(h.pokemon.name)}</button>
						{/each}
					</div>
				</div>
				{/if}
			</div>
		</div>

		<!-- ── Ability detail ──────────────────────────────────── -->
		{:else if detailCat === 'ability'}
		<div class="detail__body">
			<div class="col-left">
				<div>
					<h1 class="mon-name">{fmt(detail.name)}</h1>
					<p class="mon-num">Ability</p>
				</div>

				<div class="block">
					<p class="bt">Effect</p>
					<p class="effect-text">{engEffect(detail.effect_entries)}</p>
				</div>

				{#if detail.flavor_text_entries?.length}
				<div class="block">
					<p class="bt">Description</p>
					<p class="effect-text">{engFlavorText(detail.flavor_text_entries)}</p>
				</div>
				{/if}
			</div>

			<div class="col-right">
				{#if detail.pokemon?.length}
				<div class="block moves-block">
					<p class="bt">Pokémon with this ability ({detail.pokemon.length})</p>
					<div class="list-scroll list-scroll--grid">
						{#each detail.pokemon as p}
							{#if allPokemon.find(e => e.name === p.pokemon.name)}
								<button class="grid-btn" onclick={() => open('pokemon', p.pokemon.name)}>
									{fmt(p.pokemon.name)}{#if p.is_hidden} <span class="tag">H</span>{/if}
								</button>
							{/if}
						{/each}
					</div>
				</div>
				{/if}
			</div>
		</div>
		{/if}
	</div>
	{/if}

</div>

<style>
/* ── Shell ──────────────────────────────────────────────────────── */
.page {
	background: #2f2f2f;
	min-height: 100dvh;
	font-family: 'Cozette', monospace;
	color: #fff;
	display: flex;
	flex-direction: column;
}

/* ── Header ─────────────────────────────────────────────────────── */
.hdr {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 0.75rem 2rem;
	border-bottom: 1px solid rgba(255,255,255,0.06);
	gap: 1rem;
	flex-wrap: wrap;
}
.hdr__back { font-size: 0.75rem; color: rgba(255,255,255,0.4); text-decoration: none; transition: color 150ms; }
.hdr__back:hover { color: #fff; }
.hdr__title { font-size: 0.75rem; color: rgba(255,255,255,0.25); letter-spacing: 0.08em; }
.gen-tabs { display: flex; gap: 0.4rem; }
.gen-tab {
	font-family: 'Cozette', monospace;
	font-size: 0.7rem;
	padding: 0.3rem 0.7rem;
	border: 1px solid rgba(255,255,255,0.12);
	color: rgba(255,255,255,0.4);
	background: none;
	cursor: pointer;
	transition: all 150ms;
}
.gen-tab:hover { border-color: rgba(255,255,255,0.3); color: #fff; }
.gen-tab--on { border-color: rgba(255,255,255,0.5); color: #fff; background: rgba(255,255,255,0.06); }

/* ── Search bar ─────────────────────────────────────────────────── */
.search-bar {
	display: flex;
	gap: 0.5rem;
	padding: 0.75rem 2rem;
	border-bottom: 1px solid rgba(255,255,255,0.06);
	background: rgba(255,255,255,0.01);
}
.search-input-wrap { position: relative; flex: 1; max-width: 600px; }
.search-input {
	width: 100%;
	font-family: 'Cozette', monospace;
	font-size: 0.85rem;
	color: #fff;
	background: rgba(255,255,255,0.04);
	border: 1px solid rgba(255,255,255,0.12);
	padding: 0.55rem 0.9rem;
	outline: none;
	transition: border-color 150ms;
}
.search-input::placeholder { color: rgba(255,255,255,0.22); }
.search-input:focus { border-color: rgba(255,255,255,0.35); }
.search-input:disabled { opacity: 0.4; }

.dropdown {
	position: absolute;
	top: calc(100% + 2px);
	left: 0; right: 0;
	background: #1c1c1c;
	border: 1px solid rgba(255,255,255,0.1);
	z-index: 100;
	max-height: 300px;
	overflow-y: auto;
}
.dd-item {
	width: 100%;
	display: flex;
	align-items: center;
	gap: 0.6rem;
	padding: 0.42rem 0.9rem;
	font-family: 'Cozette', monospace;
	font-size: 0.78rem;
	color: rgba(255,255,255,0.68);
	background: none;
	border: none;
	cursor: pointer;
	text-align: left;
	transition: background 100ms;
}
.dd-item:hover { background: rgba(255,255,255,0.06); color: #fff; }
.dd-cat {
	font-size: 0.6rem;
	padding: 0.1rem 0.35rem;
	border: 1px solid rgba(255,255,255,0.15);
	color: rgba(255,255,255,0.35);
	min-width: 3.8rem;
	text-align: center;
}
.dd-cat--pokemon  { border-color: rgba(247,208,44,0.3);  color: rgba(247,208,44,0.7);  }
.dd-cat--move     { border-color: rgba(238,129,48,0.3);  color: rgba(238,129,48,0.7);  }
.dd-cat--item     { border-color: rgba(99,144,240,0.3);  color: rgba(99,144,240,0.7);  }
.dd-cat--ability  { border-color: rgba(122,199,76,0.3);  color: rgba(122,199,76,0.7);  }
.dd-num  { color: rgba(255,255,255,0.28); font-size: 0.68rem; min-width: 2.4rem; }
.dd-name { text-transform: capitalize; }

/* ── Buttons ────────────────────────────────────────────────────── */
.btn {
	font-family: 'Cozette', monospace;
	font-size: 0.78rem;
	color: rgba(255,255,255,0.75);
	border: 1px solid rgba(255,255,255,0.14);
	padding: 0.5rem 0.9rem;
	background: none;
	cursor: pointer;
	display: inline-flex;
	align-items: center;
	white-space: nowrap;
	text-decoration: none;
	transition: all 150ms;
}
.btn:hover:not(:disabled) { border-color: rgba(255,255,255,0.38); color: #fff; }
.btn:disabled { opacity: 0.35; cursor: default; }
.btn--shiny     { border-color: rgba(255,215,0,0.25); color: rgba(255,215,0,0.5); }
.btn--shiny-on  { border-color: gold; color: gold; background: rgba(255,215,0,0.05); }

/* ── Empty / loading ────────────────────────────────────────────── */
.empty-state {
	flex: 1;
	display: flex;
	align-items: center;
	justify-content: center;
	padding: 4rem 2rem;
}
.empty-hint  { font-size: 0.72rem; color: rgba(255,255,255,0.22); }
.error-txt   { font-size: 0.75rem; color: #EE8130; }
.loading-state {
	flex: 1;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	gap: 1rem;
	color: rgba(255,255,255,0.28);
	font-size: 0.78rem;
}
.spinner {
	width: 22px; height: 22px;
	border: 2px solid rgba(255,255,255,0.1);
	border-top-color: rgba(255,255,255,0.5);
	border-radius: 50%;
	animation: spin 0.8s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── Detail ─────────────────────────────────────────────────────── */
.detail {
	flex: 1;
	display: flex;
	flex-direction: column;
	padding: 1.25rem 2rem 2rem;
	gap: 1.25rem;
	max-width: 960px;
	width: 100%;
	margin-inline: auto;
}
.detail__topbar { display: flex; align-items: center; justify-content: space-between; }
.detail__topbar-right { display: flex; align-items: center; gap: 0.6rem; }
.gen-badge {
	font-size: 0.63rem;
	color: rgba(255,255,255,0.28);
	border: 1px solid rgba(255,255,255,0.1);
	padding: 0.18rem 0.55rem;
}
.cat-badge {
	font-size: 0.63rem;
	padding: 0.18rem 0.55rem;
	border: 1px solid rgba(255,255,255,0.15);
	color: rgba(255,255,255,0.35);
}
.cat-badge--pokemon  { border-color: rgba(247,208,44,0.3); color: rgba(247,208,44,0.7); }
.cat-badge--move     { border-color: rgba(238,129,48,0.3); color: rgba(238,129,48,0.7); }
.cat-badge--item     { border-color: rgba(99,144,240,0.3); color: rgba(99,144,240,0.7); }
.cat-badge--ability  { border-color: rgba(122,199,76,0.3); color: rgba(122,199,76,0.7); }

.detail__body {
	display: grid;
	grid-template-columns: 260px 1fr;
	gap: 2rem;
	align-items: start;
}

/* ── Left col ───────────────────────────────────────────────────── */
.col-left { display: flex; flex-direction: column; gap: 1.1rem; }

.sprite-box {
	display: flex; align-items: center; justify-content: center;
	background: rgba(255,255,255,0.03);
	border: 1px solid rgba(255,255,255,0.06);
	padding: 1.25rem;
}
.sprite-box--shiny  { border-color: rgba(255,215,0,0.25); background: rgba(255,215,0,0.04); }
.sprite-box--item   { padding: 1rem; }
.sprite-box img     { image-rendering: pixelated; }

.mon-name { font-size: 1.35rem; font-weight: normal; text-transform: capitalize; line-height: 1; }
.mon-num  { font-size: 0.68rem; color: rgba(255,255,255,0.28); margin-top: 0.2rem; text-transform: capitalize; }

.type-row { display: flex; gap: 0.35rem; flex-wrap: wrap; }
.type-badge {
	font-size: 0.66rem;
	padding: 0.18rem 0.5rem;
	border-radius: 2px;
	color: #fff;
	text-transform: capitalize;
	letter-spacing: 0.04em;
}

.stats { display: flex; flex-direction: column; gap: 0.32rem; }
.stat-row { display: grid; grid-template-columns: 2.6rem 2.2rem 1fr; align-items: center; gap: 0.45rem; }
.sl  { font-size: 0.65rem; color: rgba(255,255,255,0.35); }
.sv  { font-size: 0.7rem;  color: #fff; text-align: right; }
.bar-bg   { height: 4px; background: rgba(255,255,255,0.07); border-radius: 99px; overflow: hidden; }
.bar-fill { height: 100%; border-radius: 99px; transition: width 400ms ease; }
.stat-row--bst .sl { color: rgba(255,255,255,0.5); }
.stat-row--bst .sv { color: rgba(255,255,255,0.8); }

.block  { display: flex; flex-direction: column; gap: 0.42rem; }
.bt {
	font-size: 0.61rem;
	color: rgba(255,255,255,0.26);
	text-transform: uppercase;
	letter-spacing: 0.12em;
	padding-bottom: 0.28rem;
	border-bottom: 1px solid rgba(255,255,255,0.06);
}
.kv  { display: grid; grid-template-columns: auto 1fr; gap: 0.26rem 1rem; }
.k   { font-size: 0.66rem; color: rgba(255,255,255,0.3); }
.v   { font-size: 0.72rem; color: rgba(255,255,255,0.76); }

.effect-text {
	font-size: 0.78rem;
	color: rgba(255,255,255,0.65);
	line-height: 1.6;
	max-width: 100%;
}

.tag-row { display: flex; flex-wrap: wrap; gap: 0.35rem; }
.attr-tag {
	font-size: 0.62rem;
	padding: 0.15rem 0.45rem;
	border: 1px solid rgba(255,255,255,0.12);
	color: rgba(255,255,255,0.38);
}
.tag {
	font-size: 0.58rem;
	color: rgba(255,255,255,0.28);
	border: 1px solid rgba(255,255,255,0.12);
	padding: 0.08rem 0.28rem;
}

/* link buttons */
.link-btn {
	font-family: 'Cozette', monospace;
	font-size: 0.78rem;
	color: rgba(255,255,255,0.6);
	background: none;
	border: none;
	cursor: pointer;
	text-align: left;
	text-transform: capitalize;
	padding: 0;
	display: inline-flex;
	align-items: center;
	gap: 0.4rem;
	transition: color 120ms;
}
.link-btn:hover { color: #fff; text-decoration: underline; }

/* ── Right col ──────────────────────────────────────────────────── */
.col-right { display: flex; flex-direction: column; gap: 1.5rem; }

.evo-chain { display: flex; flex-wrap: wrap; align-items: center; gap: 0.4rem; }
.evo-step  { display: flex; align-items: center; gap: 0.4rem; }
.evo-arrow { font-size: 0.66rem; color: rgba(255,255,255,0.26); }
.evo-btn {
	font-family: 'Cozette', monospace;
	font-size: 0.76rem;
	color: rgba(255,255,255,0.58);
	background: rgba(255,255,255,0.03);
	border: 1px solid rgba(255,255,255,0.1);
	padding: 0.25rem 0.55rem;
	cursor: pointer;
	text-transform: capitalize;
	transition: all 150ms;
}
.evo-btn:hover { border-color: rgba(255,255,255,0.33); color: #fff; }
.evo-btn--on   { border-color: rgba(255,255,255,0.43); color: #fff; background: rgba(255,255,255,0.06); }

.moves-block { flex: 1; }
.tabs { display: flex; gap: 0.35rem; margin-bottom: 0.55rem; }
.tab {
	font-family: 'Cozette', monospace;
	font-size: 0.66rem;
	padding: 0.25rem 0.65rem;
	border: 1px solid rgba(255,255,255,0.1);
	color: rgba(255,255,255,0.36);
	background: none;
	cursor: pointer;
	transition: all 150ms;
}
.tab:hover { color: #fff; border-color: rgba(255,255,255,0.25); }
.tab--on    { color: #fff; border-color: rgba(255,255,255,0.4); background: rgba(255,255,255,0.05); }

.list-scroll {
	display: flex;
	flex-direction: column;
	max-height: 420px;
	overflow-y: auto;
	border: 1px solid rgba(255,255,255,0.06);
}
.list-scroll--grid {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
	gap: 0;
	flex-direction: unset;
}

.row-hdr {
	display: grid;
	grid-template-columns: 2.4rem 1fr;
	gap: 0.7rem;
	padding: 0.32rem 0.65rem;
	font-size: 0.6rem;
	color: rgba(255,255,255,0.26);
	text-transform: uppercase;
	letter-spacing: 0.08em;
	border-bottom: 1px solid rgba(255,255,255,0.06);
	background: rgba(255,255,255,0.02);
}
.move-row {
	display: grid;
	grid-template-columns: 2.4rem 1fr;
	gap: 0.7rem;
	padding: 0.36rem 0.65rem;
	font-size: 0.76rem;
	border-bottom: 1px solid rgba(255,255,255,0.04);
	transition: background 100ms;
	align-items: center;
}
.move-row--tm { grid-template-columns: 1fr; }
.move-row:last-child { border-bottom: none; }
.move-row:hover { background: rgba(255,255,255,0.03); }
.move-lv   { font-size: 0.68rem; color: rgba(255,255,255,0.3); }
.move-name { font-size: 0.76rem; }

.grid-btn {
	font-family: 'Cozette', monospace;
	font-size: 0.73rem;
	color: rgba(255,255,255,0.6);
	background: none;
	border: none;
	border-bottom: 1px solid rgba(255,255,255,0.04);
	border-right: 1px solid rgba(255,255,255,0.04);
	padding: 0.4rem 0.6rem;
	cursor: pointer;
	text-transform: capitalize;
	text-align: left;
	transition: background 100ms, color 100ms;
}
.grid-btn:hover { background: rgba(255,255,255,0.05); color: #fff; }

.empty { font-size: 0.72rem; color: rgba(255,255,255,0.24); padding: 0.85rem 0.65rem; }

/* ── Mobile ─────────────────────────────────────────────────────── */
@media (max-width: 660px) {
	.detail { padding: 1rem; }
	.detail__body { grid-template-columns: 1fr; }
	.list-scroll { max-height: 260px; }
	.hdr, .search-bar { padding: 0.75rem 1rem; }
}
</style>