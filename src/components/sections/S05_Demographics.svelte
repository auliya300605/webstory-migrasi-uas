<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import s05Data from "$data/s05_data.json";

	function hashStr(s) {
		let h = 2166136261 >>> 0;
		for (let i = 0; i < s.length; i++) {
			h ^= s.charCodeAt(i);
			h = Math.imul(h, 16777619);
		}
		return h >>> 0;
	}
	function rng(seed) {
		let a = seed >>> 0;
		return () => {
			a = (a + 0x6d2b79f5) >>> 0;
			let t = a;
			t = Math.imul(t ^ (t >>> 15), t | 1);
			t ^= t + Math.imul(t ^ (t >>> 7), t | 61);
			return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
		};
	}
	const r2 = (v) => Math.round(v * 100) / 100;

	const TITLE_S04 = "Siapa Itu Pekerja Migran Indonesia?";
	const titleWordsS04 = (() => {
		const rnd = rng(hashStr(TITLE_S04));
		return TITLE_S04.split(" ").map((w) =>
			[...w].map((ch) => ({
				ch,
				dy: r2(rnd() * 3.2),
				rot: r2(rnd() * 2.2 - 0.2)
			}))
		);
	})();


	/* ----------------------------------------------------
	   1. PENGATURAN & DATA LINE CHART (COMEBACK STYLE)
	---------------------------------------------------- */
	const width = 820;
	const height = 430;
	const margin = { top: 80, right: 190, bottom: 45, left: 55 };

	const trends = s05Data.trends;
	const LINE_CURVE = d3.curveLinear;
	const SHOW_AVATAR_RING = true;

	const X_START = trends[0].year;
	const X_END = trends[trends.length - 1].year;

	let currentYear = $state(X_START);
	let trackEl = $state(null);
	let chartWrapEl = $state(null);

	const xScale = d3.scaleLinear()
		.domain([X_START, X_END])
		.range([margin.left, width - margin.right]);

	const yScale = d3.scaleLinear()
		.domain([0, 100])
		.range([height - margin.bottom, margin.top]);

	const lineGeneratorFemale = d3.line()
		.curve(LINE_CURVE)
		.x((d) => xScale(d.year))
		.y((d) => yScale(d.female));

	const lineGeneratorInformal = d3.line()
		.curve(LINE_CURVE)
		.x((d) => xScale(d.year))
		.y((d) => yScale(d.informal));

	const areaGenerator = d3.area()
		.curve(LINE_CURVE)
		.x((d) => xScale(d.year))
		.y0((d) => yScale(d.informal))
		.y1((d) => yScale(d.female));

	const bisect = d3.bisector((d) => d.year).left;
	function interpolateData(yearVal) {
		const i = Math.min(Math.max(bisect(trends, yearVal), 1), trends.length - 1);
		const a = trends[i - 1];
		const b = trends[i];
		const t = Math.min(Math.max((yearVal - a.year) / (b.year - a.year), 0), 1);
		return {
			year: Math.floor(yearVal + 1e-6),
			female: a.female + (b.female - a.female) * t,
			informal: a.informal + (b.informal - a.informal) * t
		};
	}

	let activeValues = $derived(interpolateData(currentYear));
	let currentX = $derived(xScale(currentYear));

	const p2022 = trends.find((t) => t.year === 2022);
	const mid2023 = interpolateData(2023.5);

	function onLineScroll() {
		if (!trackEl) return;
		const r = trackEl.getBoundingClientRect();
		const stickyH = chartWrapEl ? chartWrapEl.offsetHeight : window.innerHeight;
		const scrollable = r.height - stickyH;
		const p = scrollable > 0 ? Math.min(Math.max(-r.top / scrollable, 0), 1) : 1;
		currentYear = X_START + p * (X_END - X_START);
	}

	/* ----------------------------------------------------
	   2. PENGATURAN & ANIMASI SWARM (TEENAGERS STYLE)
	---------------------------------------------------- */
	// Figur dirancang pada kotak 20 x 44. FIG_SCALE = ukuran tampil di peta.
	const FIG_SCALE = 0.66;

	// Palet: terakota, mustard, teal, biru, cokelat (sama dengan warna di bagian lain web story)
	const ageCategories = [
		{ key: "20-29", label: "20–29 Tahun", pct: "45,5%", count: 45, color: "#d95f43", highlight: true },
		{ key: "30-39", label: "30–39 Tahun", pct: "25,4%", count: 25, color: "#e79a3c", highlight: false },
		{ key: "40-49", label: "40–49 Tahun", pct: "17,4%", count: 18, color: "#2b847e", highlight: false },
		{ key: "<20", label: "<20 Tahun", pct: "7,0%", count: 7, color: "#0284c7", highlight: false },
		{ key: "50-59", label: "50–59 Tahun", pct: "4,3%", count: 4, color: "#78695e", highlight: false },
		{ key: "60+", label: "60+ Tahun", pct: "0,4%", count: 1, color: "#c2ae9b", highlight: false }
	];

    // Kategori Wilayah (Gambar 6 Susenas 2024: 55,36% Desa vs 44,64% Kota)
	const regionCategories = [
		{ key: "desa", label: "Perdesaan", pct: "55,4%", count: 55, color: "#2b847e" },
		{ key: "kota", label: "Perkotaan", pct: "44,6%", count: 45, color: "#d95f43" }
	];

	let activeHoverGroup = $state(null);
	let swarmProgress = $state(0);
	let swarmTrackEl = $state(null);
	let swarmStickyEl = $state(null);

	const pseudoRandom = (seed) => {
		const x = Math.sin(seed * 9999) * 10000;
		return x - Math.floor(x);
	};

	const peopleUnits = [];
	let runningIdx = 0;
	ageCategories.forEach((cat) => {
		for (let i = 0; i < cat.count; i++) {
			const id = runningIdx;
			const col = id % 10;
			const row = Math.floor(id / 10);

			const waffleX = col * 42 + 25;
			const waffleY = row * 35 + 14;

			// Susunan saat fase desa/kota: 55 pertama perdesaan, 45 sisanya perkotaan
			const isDesa = id < 55;
			const regionKey = isDesa ? "desa" : "kota";
			const regionColor = isDesa ? "#2b847e" : "#d95f43";
			// Offset vertikal kecil agar blok desa dan kota terbelah rapi secara visual
			const regionWaffleY = waffleY + (isDesa ? 0 : 8);

			const swarmX = 220 + (pseudoRandom(id * 13) - 0.5) * 380;
			const swarmY = 160 + (pseudoRandom(id * 29) - 0.5) * 220;

			peopleUnits.push({
				id,
				col,
				row,
				waffleX,
				waffleY,
				regionWaffleY,
				regionKey,
				regionColor,
				swarmX,
				swarmY,
				sx: 0.9 + pseudoRandom(id * 7) * 0.24,
				sy: 0.95 + pseudoRandom(id * 17) * 0.1,
				...cat
			});
			runningIdx++;
		}
	});

	function getPersonPos(p, progress) {
		if (progress <= 0.12) {
			// Fase 1: 1 orang di tengah
			return { x: 220, y: 150, opacity: p.id === 0 ? 1 : 0 };
		} else if (progress < 0.45) {
			// Fase 2: Ratusan orang berdatangan acak (Swarm)
			const t = (progress - 0.12) / 0.33;
			return {
				x: 220 + (p.swarmX - 220) * t,
				y: 150 + (p.swarmY - 150) * t,
				opacity: p.id === 0 ? 1 : Math.min(t * 1.5, 0.85)
			};
		} else if (progress < 0.75) {
			// Fase 3: Waffle Kelompok Umur
			const t = Math.min((progress - 0.45) / 0.25, 1);
			return {
				x: p.swarmX + (p.waffleX - p.swarmX) * t,
				y: p.swarmY + (p.waffleY - p.swarmY) * t,
				opacity: 1
			};
		} else {
			// Fase 4: Reorganisasi ke Waffle Desa vs Kota
			const t = Math.min((progress - 0.75) / 0.25, 1);
			return {
				x: p.waffleX,
				y: p.waffleY + (p.regionWaffleY - p.waffleY) * t,
				opacity: 1
			};
		}
	}

	function getPersonColor(p, progress) {
		if (progress > 0.75) {
			// Warna transisi ke Desa (Teal) vs Kota (Coral)
			return p.regionColor;
		}
		if (progress > 0.45) {
			return p.color; // Warna kelompok umur
		}
		return p.key === "20-29" ? "#d95f43" : "#a8a29e";
	}

	function onSwarmScroll() {
		if (!swarmTrackEl) return;
		const r = swarmTrackEl.getBoundingClientRect();
		const stickyH = swarmStickyEl ? swarmStickyEl.offsetHeight : window.innerHeight;
		const scrollable = r.height - stickyH;
		if (scrollable > 0) {
			swarmProgress = Math.min(Math.max(-r.top / scrollable, 0), 1);
		}
	}

	function handleAllScroll() {
		onLineScroll();
		onSwarmScroll();
	}

	onMount(handleAllScroll);
</script>

<svelte:window onscroll={handleAllScroll} onresize={handleAllScroll} />

<section id="section-5" class="section-container">
	<!-- Header Section -->
	<div class="section-intro">
		<div class="intro-prose-container">
		<div class="title-header-wrapper" style="text-align: center; margin-bottom: 2rem;">
			<h2 class="section-title" aria-label={TITLE_S04}>
				<span class="title-wrap" aria-hidden="true">
					{#each titleWordsS04 as word}
						<span class="word"
							>{#each word as l}<span
									class="letter"
									style="transform: translate(0, {l.dy}px) rotate({l.rot}deg);">{l.ch}</span
								>{/each}</span
						>
					{/each}
				</span>
			</h2>

		</div>
		<p class="lead-text">
			Lebih dari 60% PMI tercatat sebagai perempuan, dan mayoritas bekerja di sektor informal.
			Pergerakan persentase pekerja migran perempuan menunjukkan tren yang sejalan dengan dinamika sektor informal.
		</p>
		</div>
	</div>

	<!-- ====================================================
	     MODUL 1: DUAL LINE CHART (THE PUDDING COMEBACK)
	==================================================== -->
	<div class="scroll-track" bind:this={trackEl}>
		<div class="chart-wrapper" bind:this={chartWrapEl}>
			<div class="chart-header-row">
				<div class="title-group">
					<h3 class="chart-headline">
						Trend Gender dan Sektor Informal : 2020 - 2024<br />
					</h3>
				</div>
				{#if currentYear < X_START + 0.15}
					<span class="scroll-hint">Gulir ke bawah ↓</span>
				{/if}
			</div>

			<div class="svg-container">
				<svg viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
					<defs>
						<clipPath id="chart-reveal-clip">
							<rect x="0" y="0" width={currentX} height={height} />
						</clipPath>
						<clipPath id="circle-clip-gender" clipPathUnits="objectBoundingBox">
							<circle cx="0.5" cy="0.5" r="0.5" />
						</clipPath>
						<clipPath id="circle-clip-sektor" clipPathUnits="objectBoundingBox">
							<circle cx="0.5" cy="0.5" r="0.5" />
						</clipPath>
					</defs>

					<!-- Garis kisi Y -->
					{#each [25, 50, 75, 100] as tick}
						<line
							x1={margin.left}
							y1={yScale(tick)}
							x2={width - margin.right}
							y2={yScale(tick)}
							stroke="#e5e7eb"
							stroke-dasharray="3 3"
						/>
						<text x={margin.left - 8} y={yScale(tick) + 4} text-anchor="end" class="axis-tick-text">
							{tick}%
						</text>
					{/each}

					<!-- Garis jejak abu-abu -->
					<g opacity="0.12">
						<path d={lineGeneratorFemale(trends)} class="trend-line female-line" />
						<path d={lineGeneratorInformal(trends)} class="trend-line informal-line" />
					</g>

					<!-- Garis & area tersingkap -->
					<g clip-path="url(#chart-reveal-clip)">
						<path d={areaGenerator(trends)} class="shading-area" />
						<path d={lineGeneratorFemale(trends)} class="trend-line female-line" />
						<path d={lineGeneratorInformal(trends)} class="trend-line informal-line" />
					</g>

					<!-- Sumbu X -->
					{#each trends as t}
						<text x={xScale(t.year)} y={height - 12} text-anchor="middle" class="axis-year-text" class:highlight-year={Math.round(currentYear) === t.year}>
							{t.year}
						</text>
					{/each}

					<!-- Garis pemandu vertikal scanner -->
					<line
						x1={currentX}
						y1={12}
						x2={currentX}
						y2={height - margin.bottom}
						stroke="#1e293b"
						stroke-width="1.8"
					/>

					<!-- Callout angka dinamis -->
					<g transform="translate({currentX + 10}, 22)" pointer-events="none">
						<text class="svg-tag">TAHUN {activeValues.year}</text>
						<text y="32" class="svg-big">{Math.round(activeValues.female)}%<tspan class="svg-sub" dx="6">PMI Perempuan</tspan></text>
						<text y="50" class="svg-note">{Math.round(activeValues.informal)}% Sektor Informal</text>
					</g>

					<!-- Anotasi 2022 -->
					{#if currentYear >= 2022}
						<g transform="translate({xScale(2022)}, {yScale(p2022.informal)})" class="chart-annotation">
							<line x1="0" y1="6" x2="0" y2="30" stroke="#94a3b8" stroke-width="1" />
							<rect x="-95" y="30" width="190" height="24" rx="3" fill="#ffffff" stroke="#cbd5e1" />
							<text x="0" y="45.5" text-anchor="middle" class="ann-text">2022: Keduanya anjlok serempak</text>
						</g>
					{/if}

					<!-- Anotasi 2024 -->
					{#if currentYear >= 2023.6}
						<g transform="translate({xScale(2023.5)}, {yScale(mid2023.informal)})" class="chart-annotation">
							<line x1="0" y1="6" x2="0" y2="30" stroke="#94a3b8" stroke-width="1" />
							<rect x="-95" y="30" width="190" height="24" rx="3" fill="#ffffff" stroke="#cbd5e1" />
							<text x="0" y="45.5" text-anchor="middle" class="ann-text">2024: Kembali naik beriringan</text>
						</g>
					{/if}

					<!-- Avatar Perempuan -->
					<g transform="translate({currentX + 2}, {yScale(activeValues.female) - 16})">
						{#if SHOW_AVATAR_RING}
							<circle cx="16" cy="16" r="18" fill="#ffffff" stroke="#e11d48" stroke-width="2.5" />
						{/if}
						<image
							href="/assets/cewe_cowo.svg"
							x="0"
							y="0"
							width="32"
							height="32"
							preserveAspectRatio="xMidYMid slice"
							clip-path="url(#circle-clip-gender)"
						/>
						<text x="42" y="16" class="direct-label female-label">
							Pekerja Perempuan ({Math.round(activeValues.female)}%)
						</text>
					</g>

					<!-- Avatar Sektor Informal -->
					<g transform="translate({currentX + 2}, {yScale(activeValues.informal) - 16})">
						{#if SHOW_AVATAR_RING}
							<circle cx="16" cy="16" r="18" fill="#ffffff" stroke="#0284c7" stroke-width="2.5" />
						{/if}
						<image
							href="/assets/formal_formal.svg"
							x="0"
							y="0"
							width="32"
							height="32"
							preserveAspectRatio="xMidYMid slice"
							clip-path="url(#circle-clip-sektor)"
						/>
						<text x="42" y="16" class="direct-label informal-label">
							Sektor Informal ({Math.round(activeValues.informal)}%)
						</text>
					</g>
				</svg>
			</div>

			<div class="chart-source">
				Sumber: Kementerian Pelindungan Pekerja Migran Indonesia (KP2MI) & BPS, 2024
			</div>
		</div>
	</div>

	<!-- ====================================================
	     MODUL 2: SWARM -> WAFFLE CHART (THE PUDDING TEENAGERS)
	==================================================== -->
	<div class="swarm-track" bind:this={swarmTrackEl}>
		<div class="swarm-sticky-view" bind:this={swarmStickyEl}>
			<div class="swarm-header-row">
				<p class="chart-eyebrow">
					{swarmProgress > 0.75 ? "KLASIFIKASI ASAL WILAYAH PMI (SUSENAS 2024)" : "STRUKTUR KELOMPOK USIA PEKERJA MIGRAN"}
				</p>
				<h3 class="chart-headline">
					{#if swarmProgress > 0.75}
						Asal Wilayah: Desa vs Kota <br />
						<span class="headline-light">55 dari 100 PMI berasal dari keluarga di perdesaan</span>
					{:else}
						Usia Pekerja Migran <br />
						<span class="headline-light">1 figur = 1% dari total sampel penempatan (SUSENAS 2024)</span>
					{/if}
				</h3>
			</div>

			<div class="swarm-visual-layout">
				<div class="swarm-stage">
					<svg viewBox="0 0 480 400" preserveAspectRatio="xMidYMid meet" class="swarm-svg">
						{#if swarmProgress < 0.25}
							<polygon points="230,135 224,124 236,124" fill="#d95f43" class="bounce-arrow" />
						{/if}

						{#each peopleUnits as person (person.id)}
							{@const pos = getPersonPos(person, swarmProgress)}
							{@const isTargetGroup = person.key === "20-29"}
							{@const isHovered = activeHoverGroup === person.key}
							{@const isDimmed = activeHoverGroup !== null && !isHovered}

							<!-- svelte-ignore a11y_no_static_element_interactions -->
							<g
								transform="translate({pos.x}, {pos.y})"
								opacity={isDimmed ? 0.2 : pos.opacity}
								class="person-agent"
								onmouseenter={() => (activeHoverGroup = person.key)}
								onmouseleave={() => (activeHoverGroup = null)}
							>
								<g
                                    transform="scale({person.sx * FIG_SCALE}, {person.sy * FIG_SCALE}) translate(-10, 0)"
                                    fill={getPersonColor(person, swarmProgress)}
                                    class="person-body"
                                    style="transition: fill 0.6s ease;"
                                >
                                    <circle cx="10" cy="5" r="4.3" />
                                    <rect x="4.6" y="10.8" width="10.8" height="15" rx="3" />
                                    <rect x="1.3" y="11.6" width="2.9" height="14" rx="1.45" />
                                    <rect x="15.8" y="11.6" width="2.9" height="14" rx="1.45" />
                                    <rect x="5.4" y="25" width="4.2" height="19" rx="2" />
                                    <rect x="10.4" y="25" width="4.2" height="19" rx="2" />
                                </g>
							</g>
						{/each}
					</svg>
				</div>

				<div class="swarm-legend-panel" class:show-legend={swarmProgress > 0.45}>
					{#if swarmProgress > 0.75}
						<span class="legend-note">Klasifikasi Desa / Kota:</span>
						<div class="legend-list">
							{#each regionCategories as cat}
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<div
									class="legend-row"
									class:active={activeHoverGroup === cat.key}
									onmouseenter={() => (activeHoverGroup = cat.key)}
									onmouseleave={() => (activeHoverGroup = null)}
								>
									<span class="swatch" style="background-color: {cat.color};"></span>
									<span class="name">{cat.label}</span>
									<strong class="pct">{cat.pct}</strong>
								</div>
							{/each}
						</div>
					{:else}
						<span class="legend-note">Kelompok Umur Berangkat:</span>
						<div class="legend-list">
							{#each ageCategories as cat}
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<div
									class="legend-row"
									class:active={activeHoverGroup === cat.key}
									onmouseenter={() => (activeHoverGroup = cat.key)}
									onmouseleave={() => (activeHoverGroup = null)}
								>
									<span class="swatch" style="background-color: {cat.color};"></span>
									<span class="name">{cat.label}</span>
									<strong class="pct">{cat.pct}</strong>
								</div>
							{/each}
						</div>
					{/if}
					<p class="legend-subtext">
						Sumber: Badan Pusat Statistik, Susenas Maret 2024
					</p>
				</div>
			</div>
		</div>

		<!-- Kartu narasi bergulir -->
		<div class="swarm-cards-overlay">
			<div class="swarm-step-box">
				<p>
					Setiap figur mewakili satu pekerja migran. Saat dikelompokkan menurut usia, polanya mulai terlihat.
				</p>
			</div>

			<div class="swarm-step-box">
				<p>
					Dari sisi usia, mayoritas PMI berusia <strong class="hl-coral">20–29 tahun</strong>,
					fase awal karier di mana migrasi sering menjadi jalan menuju kemandirian ekonomi.
				</p>
			</div>

			<!-- Kartu Baru: Desa vs Kota -->
			<div class="swarm-step-box">
				<p>
					Tidak hanya usia muda, faktor spasial juga sangat kental.
					Hasil Susenas 2024 mencatat bahwa <strong class="hl-teal">55,4% PMI</strong> berasal dari keluarga yang tinggal di
					<strong class="hl-teal">perdesaan</strong> akibat keterbatasan lapangan kerja dan rendahnya upah domestik.
				</p>
			</div>
		</div>
	</div>
</section>

<style>
	.section-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.2rem, 5vw, 3.4rem);
		color: var(--color-primary, #78695e);
		text-align: center;
		letter-spacing: 0.05em;
		line-height: 1.15;
		margin: 0;
		font-weight: 400;
	}

	.word {
		display: inline-block;
		white-space: nowrap;
		margin: 0 0.2em;
	}

	.letter {
		display: inline-block;
	}
	
	/* 1. Header Pengantar Non-Scrolling */
	.intro-prose-container {
		width: 100%;
		max-width: 680px;
		margin: 0 auto;
		padding: 5.5rem 1.5rem 2.5rem;
		box-sizing: border-box;
	}

	.section-container {
		width: 100%;
		max-width: 900px;
		margin: 0 auto;
		padding: 5rem 1.5rem 0;
		box-sizing: border-box;
		background: #ffffff;
	}


	.lead-text {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.8;
		color: #374151;
		text-align: justify;
		margin: 0;
	}

	/* Scroll track untuk line chart */
	.scroll-track {
		height: 240vh;
		position: relative;
	}

	.chart-wrapper {
		position: sticky;
		top: 0;
		display: flex;
		flex-direction: column;
		justify-content: flex-start;
		width: 100%;
		background: #ffffff;
		padding: 1rem 1.8rem 1rem;
		box-sizing: border-box;
	}

	.chart-header-row {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		gap: 1.5rem;
		margin-bottom: 0.5rem;
	}

	.chart-eyebrow {
		font-family: var(--mono, monospace);
		font-size: 0.72rem;
		color: #6b7280;
		letter-spacing: 0.08em;
		margin: 0 0 0.3rem;
		font-weight: 600;
	}

	.chart-headline {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.35rem;
		font-weight: 700;
		color: #111827;
		margin: 0;
		line-height: 1.25;
	}

	.headline-light {
		font-weight: 400;
		font-size: 1.05rem;
		color: #4b5563;
	}

	.scroll-hint {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		color: #6b7280;
		white-space: nowrap;
		padding-top: 0.2rem;
		animation: nudge 1.4s ease-in-out infinite;
	}

	@keyframes nudge {
		0%, 100% { transform: translateY(0); }
		50% { transform: translateY(3px); }
	}

	.svg-container {
		width: 100%;
		height: auto;
	}

	svg {
		width: 100%;
		height: auto;
		display: block;
	}

	.shading-area {
		fill: rgba(225, 29, 72, 0.12);
	}

	.trend-line {
		fill: none;
		stroke-width: 2.5;
		stroke-linejoin: round;
		stroke-linecap: round;
	}

	.female-line { stroke: #78695e; }
	.informal-line { stroke: #0284c7; }

	.axis-tick-text {
		font-family: var(--mono, monospace);
		font-size: 9.5px;
		fill: #9ca3af;
	}

	.axis-year-text {
		font-family: var(--mono, monospace);
		font-size: 11px;
		font-weight: 600;
		fill: #374151;
	}

	.highlight-year {
		fill: #78695e !important;
		font-weight: 800;
	}

	.svg-tag {
		font-family: var(--mono, monospace);
		font-size: 11px;
		font-weight: 700;
		fill: #78695e;
		letter-spacing: 0.05em;
	}

	.svg-big {
		font-family: var(--mono, monospace);
		font-size: 30px;
		font-weight: 800;
		fill: #78695e;
	}

	.svg-sub {
		font-family: var(--sans, sans-serif);
		font-size: 12px;
		font-weight: 500;
		fill: #4b5563;
	}

	.svg-note {
		font-family: var(--mono, monospace);
		font-size: 11px;
		font-weight: 600;
		fill: #0284c7;
	}

	.chart-annotation rect {
		filter: drop-shadow(0 1px 3px rgba(0, 0, 0, 0.05));
	}

	.ann-text {
		font-family: var(--sans, sans-serif);
		font-size: 9px;
		font-weight: 600;
		fill: #475569;
	}

	.direct-label {
		font-family: var(--sans, sans-serif);
		font-size: 10.5px;
		font-weight: 700;
		dominant-baseline: middle;
	}

	.female-label { fill: #78695e; }
	.informal-label { fill: #0284c7; }

	.chart-source {
		margin-top: 0.8rem;
		font-family: var(--mono, monospace);
		font-size: 0.65rem;
		color: #9ca3af;
		text-align: center;
	}

	/* CSS Track Swarm */
	.swarm-track {
		position: relative;
		min-height: 300vh; 
	}

    .hl-teal { 
        color: #2b847e; 
        font-weight: 700; }

	.swarm-sticky-view {
		position: sticky;
		top: 0;
		height: 100vh;
		display: flex;
		flex-direction: column;
		justify-content: flex-start;
		align-items: center;
		padding-top: 2.5rem;
		box-sizing: border-box;
		background: #ffffff;
		overflow: hidden;
		z-index: 1;
	}

	.swarm-header-row {
	align-self: stretch;
	padding: 0 1.8rem;
	margin-bottom: 0.5rem;
	text-align: left;
}

	.swarm-visual-layout {
		display: grid;
		grid-template-columns: 500px 220px;
		gap: 1.5rem;
		align-items: center;
		width: 100%;
		max-width: 820px;
	}

	.swarm-stage {
		width: 100%;
	}

	.swarm-svg {
		width: 100%;
		height: auto;
		display: block;
	}

	.person-agent {
		cursor: pointer;
		transition: opacity 0.4s ease, transform 0.25s linear;
	}

	.person-agent:hover .person-body {
		filter: drop-shadow(0 2px 4px rgba(0,0,0,0.25));
	}

	.bounce-arrow {
		animation: arrow-bounce 1s ease-in-out infinite alternate;
	}

	@keyframes arrow-bounce {
		from { transform: translateY(0); }
		to { transform: translateY(-5px); }
	}

	.swarm-legend-panel {
		background: none;
		border: none;
		padding: 0;
		opacity: 0;
		transform: translateX(15px);
		transition: opacity 0.5s ease, transform 0.5s ease;
		pointer-events: none;
	}

	.swarm-legend-panel.show-legend {
		opacity: 1;
		transform: translateX(0);
		pointer-events: auto;
	}

	.legend-note {
		font-family: var(--mono, monospace);
		font-size: 0.65rem;
		color: #4b5563;
		text-transform: uppercase;
		display: block;
		margin-bottom: 0.6rem;
		font-weight: 600;
	}

	.legend-list {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
	}

	.legend-row {
		display: flex;
		align-items: center;
		gap: 0.45rem;
		font-family: var(--sans, sans-serif);
		font-size: 0.75rem;
		color: #374151;
		padding: 3px 6px;
		border-radius: 3px;
		cursor: pointer;
		transition: background-color 0.15s;
	}

	.legend-row:hover, .legend-row.active {
		background: #f1f5f9;
		font-weight: bold;
	}

	.swatch {
		width: 10px;
		height: 10px;
		border-radius: 2px;
		display: inline-block;
	}

	.name { flex-grow: 1; }
	.pct { font-family: var(--mono, monospace); font-size: 0.72rem; }

	.legend-subtext {
		font-family: var(--mono, monospace);
		font-size: 0.6rem;
		color: #9ca3af;
		margin: 0.8rem 0 0;
	}

	.swarm-cards-overlay {
		position: relative;
		z-index: 10;
		display: flex;
		flex-direction: column;
		align-items: center;
		margin-top: -100vh;
		padding: 14vh 1.5rem 2vh;
		pointer-events: none;
	}

	.swarm-step-box {
		pointer-events: auto;
		background: rgba(255, 255, 255, 0.92);
		backdrop-filter: blur(8px);
		-webkit-backdrop-filter: blur(8px);
		border: 1px solid rgba(229, 231, 235, 0.9);
		box-shadow: 0 8px 24px rgba(0, 0, 0, 0.05);
		max-width: 420px;
		padding: 1rem 1.4rem;
		border-radius: 6px;
		margin: 0 0 80vh;
	}

	.swarm-step-box:last-child {
		margin-bottom: 0;
	}

	.swarm-step-box p {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1rem;
		line-height: 1.75;
		color: #2b2b2b;
		margin: 0;
		text-align: justify;
	}

	.hl-coral { color: #d95f43; font-weight: 700; }

	@media (max-width: 768px) {
		.chart-header-row {
			flex-direction: column;
			align-items: flex-start;
		}
		.chart-wrapper {
			padding: 1rem 0.5rem;
		}
		.swarm-visual-layout {
			grid-template-columns: 1fr;
		}
		.swarm-legend-panel {
			display: none;
		}
	}
</style>