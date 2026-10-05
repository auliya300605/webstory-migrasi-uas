<script>

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

	const TITLE = "Ke Mana Pekerja Migran Berangkat?";
	const titleWords = (() => {
		const rnd = rng(hashStr(TITLE));
		return TITLE.split(" ").map((w) =>
			[...w].map((ch) => ({
				ch,
				dy: r2(rnd() * 3.2),
				rot: r2(rnd() * 2.2 - 0.2)
			}))
		);
	})();

	const SQUIGGLE_D =
		"M3 13.5C22.5 7.5 45.2 6.8 65.5 12.2C85.8 17.6 106.1 18.2 125.8 11.8C131.2 10.1 136.5 7.8 141 5";

	import { onMount, untrack } from "svelte";
	import * as d3 from "d3";
	import Scrolly from "$components/helpers/Scrolly.svelte";
	import flowData from "$data/s06_data.json";

	// Posisi balok pendidikan relatif terhadap peta: "top" atau "bottom".
	// Ganti satu kata ini untuk membandingkan keduanya.
	const BAR_SIDE = "top";

	let scrollValue = $state(0);
	let pastEnd = $state(false);
	let lastCardEl = $state(null);
	let worldGeo = $state(null);
	let mapError = $state(false);
	let hoveredDest = $state(null);
	let hoveredEdu = $state(null);
	let hoveredLink = $state(null);
	let tooltipPos = $state({ x: 0, y: 0 });

	const width = 1000;
	const height = 560;

	// Kamera Sankey: digeser ke kanan agar area kiri (Amerika/Atlantik) kosong untuk kartu cerita.
	// Mode "top" butuh ruang ekstra di atas untuk balok + label.
	const sankeyCam = BAR_SIDE === "top"
		? { x: 280, y: 80, w: 760, h: 440 }
		: { x: 280, y: 100, w: 760, h: 430 };

	// Viewbox kamera per scroll step
	const cameraViews = [
		{ x: 290, y: 50, w: 720, h: 480 },  // Step 0: Global Overview
		{ x: 420, y: 70, w: 520, h: 360 },  // Step 1: Fokus Asia Timur & Tenggara
		{ x: 500, y: 100, w: 460, h: 340 }, // Step 2: Fokus Jepang & Korsel
		{ x: 260, y: 70, w: 540, h: 380 },  // Step 3: Fokus Eropa & Timur Tengah
		{ x: 290, y: 50, w: 720, h: 480 },  // Step 4: Transisi Overview
		// STEP 5-8: SANKEY LINKED MAP
		{ ...sankeyCam }, // Step 5: Sankey lengkap
		{ ...sankeyCam }, // Step 6: Fokus pendidikan dasar (SD/SMP)
		{ ...sankeyCam }, // Step 7: Fokus pendidikan menengah & tinggi
		{ ...sankeyCam }  // Step 8: Penutup, semua pita menyala lagi
	];

	const LAST_STEP = cameraViews.length - 1;

	let activeStep = $derived(
		pastEnd
			? LAST_STEP
			: Number.isInteger(scrollValue) && scrollValue >= 0
				? Math.min(scrollValue, LAST_STEP)
				: 0
	);

	function updatePastEnd() {
		if (!lastCardEl) return;
		pastEnd = lastCardEl.getBoundingClientRect().bottom < 240;
	}

	// Kamera bergerak mulus (tween) antar-step. viewBox tidak bisa dianimasikan lewat CSS,
	// jadi diinterpolasi manual dengan d3.timer.
	let view = $state({ ...cameraViews[0] });
	let viewTimer = null;

	$effect(() => {
		const target = cameraViews[activeStep] || cameraViews[0];
		const from = untrack(() => ({ x: view.x, y: view.y, w: view.w, h: view.h }));
		const interp = d3.interpolateObject(from, target);
		const reduce = window.matchMedia?.("(prefers-reduced-motion: reduce)").matches;
		const duration = reduce ? 0 : 900;

		viewTimer?.stop();
		viewTimer = d3.timer((elapsed) => {
			const t = duration === 0 ? 1 : Math.min(1, elapsed / duration);
			view = interp(d3.easeCubicInOut(t));
			if (t >= 1) viewTimer.stop();
		});
		return () => viewTimer?.stop();
	});

	let zoomK = $derived(width / view.w);
	let isSankeyMode = $derived(activeStep >= 5);

	// Reset hover saat berpindah antar mode agar tooltip lama tidak "nyangkut"
	$effect(() => {
		isSankeyMode;
		hoveredDest = null;
		hoveredEdu = null;
		hoveredLink = null;
	});

	// Proyeksi peta datar
	const projection = d3.geoNaturalEarth1()
		.scale(195)
		.translate([width / 2, height / 2 + 10]);

	const pathGenerator = d3.geoPath().projection(projection);
	let originPos = $derived(projection(flowData.origin.coords));

	const destById = Object.fromEntries(flowData.destinations.map((d) => [d.id, d]));

	const fmtInt = (v) => Math.round(v).toLocaleString("id-ID");
	const fmtPct = (v) => v.toLocaleString("id-ID", { minimumFractionDigits: 2, maximumFractionDigits: 2 });

	// Skala tinggi batang 3D
	const spikeHeightScale = d3.scaleLinear()
		.domain([400, 85000])
		.range([8, 90]);

	// Garis lengkung penerbangan mode 1 (Flight Arcs)
	function getFlightCurve(source, target) {
		const s = projection(source);
		const t = projection(target);
		if (!s || !t) return "";

		const dx = t[0] - s[0];
		const dy = t[1] - s[1];
		const dist = Math.hypot(dx, dy);
		const cx = (s[0] + t[0]) / 2;
		const cy = (s[1] + t[1]) / 2 - Math.min(dist * 0.42, 110);

		return `M ${s[0]},${s[1]} Q ${cx},${cy} ${t[0]},${t[1]}`;
	}

	function getStepCategory(destId) {
		if (["TWN", "MYS", "HKG"].includes(destId)) return "top3";
		if (["JPN", "KOR"].includes(destId)) return "skilled";
		if (["ITA", "POL", "HUN", "TUR", "SAU", "ARE", "KWT"].includes(destId)) return "europe-mideast";
		return "other";
	}

	function getDestStyle(dest, step) {
		const isHovered = hoveredDest?.id === dest.id;
		const cat = getStepCategory(dest.id);

		if (isHovered) {
			return { color: "#0f172a", opacity: 1, active: true, width: 3 };
		}

		if (step === 1) {
			return cat === "top3"
				? { color: "#e11d48", opacity: 1, active: true, width: 2.8 }
				: { color: "#94a3b8", opacity: 0.18, active: false, width: 1 };
		}
		if (step === 2) {
			return cat === "skilled"
				? { color: "#0891b2", opacity: 1, active: true, width: 2.8 }
				: { color: "#94a3b8", opacity: 0.18, active: false, width: 1 };
		}
		if (step === 3) {
			return cat === "europe-mideast"
				? { color: "#8b5cf6", opacity: 1, active: true, width: 2.8 }
				: { color: "#94a3b8", opacity: 0.18, active: false, width: 1 };
		}

		const defaultColors = {
			top3: "#e11d48",
			skilled: "#0891b2",
			"europe-mideast": "#8b5cf6",
			other: "#0d9488"
		};
		return { color: defaultColors[cat], opacity: 0.85, active: true, width: 1.8 };
	}

	// Label negara di peta: step 1-3 hanya kelompok yang sedang dibahas,
	// step 0 & 4 hanya negara besar. Yang di-hover selalu tampil.
	function showLabel(dest, style, step) {
		if (hoveredDest?.id === dest.id) return true;
		if (step >= 1 && step <= 3) return style.active;
		return dest.count >= 9000;
	}

	const barScale = d3.scaleSqrt().domain([0, 85000]).range([0, 100]);
	let sortedDestinations = $derived(
		[...flowData.destinations].sort((a, b) => b.count - a.count)
	);

	// ==========================================
	// PETA DUNIA (dimuat dari /static/data/world-countries.json)
	// ==========================================
	onMount(async () => {
		try {
			const res = await fetch("/data/world-countries.json");
			if (!res.ok) throw new Error(`HTTP ${res.status}`);
			worldGeo = await res.json();
		} catch (err) {
			console.error("Gagal membaca peta dunia:", err);
			mapError = true;
		}
	});

	// Pencocokan polygon -> kode negara. Dicoba lewat kode ISO3 dulu, lalu lewat nama,
	// supaya tahan terhadap perbedaan struktur properti antar-dataset geojson.
	const NAME_ALIASES = {
		IDN: ["indonesia"],
		MYS: ["malaysia"],
		TWN: ["taiwan"],
		SAU: ["saudi arabia", "arab saudi"],
		HKG: ["hong kong"],
		SGP: ["singapore", "singapura"],
		JPN: ["japan", "jepang"]
	};

	function matchFeature(f) {
		const p = f.properties || {};
		const keys = [f.id, p.iso_a3, p.ISO_A3, p.ADM0_A3, p.adm0_a3, p.ISO3, p.cca3, p["ISO3166-1-Alpha-3"]]
			.filter(Boolean)
			.map((s) => String(s).toUpperCase());
		const names = [p.name, p.admin, p.NAME, p.ADMIN, p.name_en, p.NAME_EN, p.sovereignt]
			.filter(Boolean)
			.map((s) => String(s).toLowerCase());

		if (keys.includes("ATA") || names.includes("antarctica")) return "ATA";
		for (const [code, aliases] of Object.entries(NAME_ALIASES)) {
			if (keys.includes(code)) return code;
			if (names.some((n) => aliases.some((a) => n === a))) return code;
		}
		return null;
	}

	// Path polygon dihitung sekali saja (bukan di setiap render). Antartika dibuang
	// agar area selatan peta bersih untuk balok pendidikan.
	let countryShapes = $derived.by(() => {
		if (!worldGeo?.features) return [];
		return worldGeo.features
			.map((f) => ({ d: pathGenerator(f), code: matchFeature(f) }))
			.filter((s) => s.d && s.code !== "ATA");
	});

	// Kode negara yang polygon-nya ada di dataset (dataset kecil sering tidak memuat HK / Singapura)
	let matchedCodes = $derived(new Set(countryShapes.map((s) => s.code).filter(Boolean)));

	// ==========================================
	// DATA SANKEY LINKED MAP
	// Persentase jenjang pendidikan: BPS, Long Form SP2020, Gambar 12.
	// Jumlah pekerja per negara + koordinat: s06_data.json (BP2MI 2023).
	// Korea Selatan SENGAJA tidak ada: tidak dirinci di Gambar 12 BPS.
	// Kalau punya sumbernya, cukup tambahkan baris KOR di bawah.
	// ==========================================
	const eduCategories = [
		{ id: "SD", label: "≤ SD", color: "#0284c7" },
		{ id: "SMP", label: "SMP", color: "#38bdf8" },
		{ id: "SMA", label: "SMA/SMK", color: "#f97316" },
		{ id: "PT", label: "PT", color: "#1e3a8a" }
	];

	const eduDataByCountry = {
		MYS: { SD: 43.08, SMP: 26.57, SMA: 29.02, PT: 1.34 },
		TWN: { SD: 18.25, SMP: 39.87, SMA: 40.41, PT: 1.47 },
		SAU: { SD: 52.32, SMP: 29.91, SMA: 15.62, PT: 2.15 },
		HKG: { SD: 20.89, SMP: 44.58, SMA: 33.31, PT: 1.22 },
		SGP: { SD: 25.53, SMP: 39.55, SMA: 29.23, PT: 5.69 },
		JPN: { SD: 0.63, SMP: 2.66, SMA: 83.42, PT: 13.29 }
	};

	const ORIGIN_FILL = "#fcd34d";
	const OTHER_FILL = "#d5dce5";

	// Jenjang dominan = jenjang dengan persentase tertinggi
	function dominantEdu(code) {
		const p = eduDataByCountry[code];
		return eduCategories.reduce((best, e) => (p[e.id] > p[best.id] ? e : best), eduCategories[0]);
	}

	// Fokus sorotan per step: step 6 = SD/SMP di Malaysia & Arab Saudi, step 7 = SMA/SMK/PT di Jepang
	const stepFocus = {
		6: { edu: ["SD", "SMP"], countries: ["MYS", "SAU"] },
		7: { edu: ["SMA", "PT"], countries: ["JPN"] }
	};

	function isCountryFocused(code) {
		const f = stepFocus[activeStep];
		return !f || f.countries.includes(code);
	}

	function polygonStyle(code) {
		if (!isSankeyMode) return { fill: "#e2e8f0", opacity: 1 };
		if (code === "IDN") return { fill: ORIGIN_FILL, opacity: 1 };
		if (code && eduDataByCountry[code]) {
			const hovered = hoveredDest?.id === code || hoveredLink?.countryId === code;
			return {
				fill: dominantEdu(code).color,
				opacity: hovered ? 1 : isCountryFocused(code) ? 0.85 : 0.28
			};
		}
		return { fill: OTHER_FILL, opacity: 1 };
	}

	// Tata letak: 4 balok pendidikan berjajar di bawah peta (samudra selatan),
	// pita naik langsung ke titik negara tujuan. Tidak ada kolom tengah.
	const BAR = { x: 600, y: BAR_SIDE === "top" ? 108 : 480, h: 14, innerW: 330, gap: 14, taper: 0.16 };
	const markerScale = d3.scaleSqrt().domain([0, 85000]).range([3, 8]);

	const sankeyLayout = $derived.by(() => {
		const dests = flowData.destinations
			.filter((d) => eduDataByCountry[d.id])
			.map((d) => ({ ...d, pt: projection(d.coords) }))
			.filter((d) => d.pt);

		// 1. Nilai tiap pita = jumlah pekerja negara x persentase jenjang
		const eduTotals = Object.fromEntries(eduCategories.map((e) => [e.id, 0]));
		const vals = new Map();
		dests.forEach((d) =>
			eduCategories.forEach((e) => {
				const v = (d.count * eduDataByCountry[d.id][e.id]) / 100;
				vals.set(`${e.id}|${d.id}`, v);
				eduTotals[e.id] += v;
			})
		);
		const grand = d3.sum(Object.values(eduTotals));
		const k = BAR.innerW / grand;

		// 2. Balok pendidikan (horizontal, lebar sebanding jumlah pekerja)
		let cursor = BAR.x;
		const eduNodes = eduCategories.map((e) => {
			const w = eduTotals[e.id] * k;
			const node = {
				id: e.id,
				label: e.label,
				color: e.color,
				total: Math.round(eduTotals[e.id]),
				share: (eduTotals[e.id] / grand) * 100,
				x0: cursor,
				x1: cursor + w,
				w,
				y0: BAR.y,
				y1: BAR.y + BAR.h
			};
			cursor += w + BAR.gap;
			return node;
		});

		// 3. Pita: tiap balok dipecah per negara (urut barat -> timur agar silangan minimal)
		const destsByX = [...dests].sort((a, b) => a.pt[0] - b.pt[0]);
		const links = [];
		eduNodes.forEach((en) => {
			let sx = en.x0;
			destsByX.forEach((d) => {
				const val = vals.get(`${en.id}|${d.id}`);
				const w = val * k;
				links.push({
					eduId: en.id,
					eduLabel: en.label,
					color: en.color,
					countryId: d.id,
					countryName: d.name,
					countryTotal: d.count,
					pct: eduDataByCountry[d.id][en.id],
					val,
					w,
					sx0: sx,
					sx1: sx + w
				});
				sx += w;
			});
		});

		// 4. Ujung pita menyempit & bertumpuk rapi di titik negara
		const endW = (w) => Math.max(w * BAR.taper, 1.6);
		const endTotal = {};
		links.forEach((l) => (endTotal[l.countryId] = (endTotal[l.countryId] || 0) + endW(l.w)));
		const endCursor = Object.fromEntries(
			dests.map((d) => [d.id, d.pt[0] - endTotal[d.id] / 2])
		);
		const ptById = Object.fromEntries(dests.map((d) => [d.id, d.pt]));

		links.forEach((l) => {
			const [, ty] = ptById[l.countryId];
			const we = endW(l.w);
			const tx0 = endCursor[l.countryId];
			const tx1 = tx0 + we;
			endCursor[l.countryId] += we;

			const yb = BAR_SIDE === "top" ? BAR.y + BAR.h : BAR.y;
			const c = (yb - ty) * 0.55;
			l.path =
				`M${l.sx0},${yb} C${l.sx0},${yb - c} ${tx0},${ty + c} ${tx0},${ty} ` +
				`L${tx1},${ty} C${tx1},${ty + c} ${l.sx1},${yb - c} ${l.sx1},${yb} Z`;
		});

		const destNodes = dests.map((d) => ({
			...d,
			x: d.pt[0],
			y: d.pt[1],
			r: markerScale(d.count),
			dominant: dominantEdu(d.id)
		}));

		return { eduNodes, links, destNodes };
	});

	// Pita besar digambar duluan supaya pita kecil tetap kelihatan di atasnya
	let renderLinks = $derived([...sankeyLayout.links].sort((a, b) => b.w - a.w));

	function linkOpacity(l) {
		if (hoveredLink) {
			return hoveredLink.eduId === l.eduId && hoveredLink.countryId === l.countryId ? 0.95 : 0.07;
		}
		if (hoveredEdu) return hoveredEdu === l.eduId ? 0.85 : 0.07;
		if (hoveredDest) return hoveredDest.id === l.countryId ? 0.85 : 0.07;
		const f = stepFocus[activeStep];
		if (!f) return 0.55;
		return f.edu.includes(l.eduId) && f.countries.includes(l.countryId) ? 0.8 : 0.08;
	}

	// Posisi label negara supaya tidak saling tumpuk (HK/Taiwan, Malaysia/Singapura berdekatan)
	const labelCfg = {
		TWN: { dx: 11, dy: -4, anchor: "start" },
		HKG: { dx: -10, dy: 4, anchor: "end" },
		MYS: { dx: -11, dy: 3, anchor: "end" },
		SGP: { dx: 9, dy: 11, anchor: "start" },
		JPN: { dx: 10, dy: 3, anchor: "start" },
		SAU: BAR_SIDE === "top" ? { dx: 0, dy: 17, anchor: "middle" } : { dx: 0, dy: -12, anchor: "middle" }
	};
</script>

<svelte:window onscroll={updatePastEnd} onresize={updatePastEnd} />
<section id="section-6" class="flow-map-section">
	<!-- JUDUL SECTION -->
	<div class="section-intro">
		<h1 class="main-title" aria-label={TITLE}>
			<span class="title-wrap" aria-hidden="true">
				{#each titleWords as word}
					<span class="word"
						>{#each word as l}<span
								class="letter"
								style="transform: translate(0, {l.dy}px) rotate({l.rot}deg);">{l.ch}</span
							>{/each}</span
					>
				{/each}
			</span>
			
		</h1>

		<p class="lead-text">
			Telusuri koridor migrasi dan bagaimana pasar kerja dunia menyerap latar belakang sekolah pekerja kita.
		</p>
	</div>

	<div class="scrolly-container">
		<!-- VIEWPORT MAP KANVAS UTAMA -->
		<div class="sticky-map-viewport">
			<div class="map-top-bar">
				<div class="header-titles">
					<span class="badge-tag">
						{#if isSankeyMode}
							SANKEY LINKED MAP: LATAR BELAKANG PENDIDIKAN & NEGARA TUJUAN
						{:else}
							PETA MIGRATION FLOW & 3D VOLUME BARS
						{/if}
					</span>
					<h3 class="view-title">
						{#if isSankeyMode}
							Aliran Kualifikasi Ijazah ke Pasar Kerja Global (BPS SP2020)
						{:else}
							Arus Penempatan PMI ke 15 Negara Utama (2023)
						{/if}
					</h3>
				</div>

				<div class="map-legend-pills">
					{#if isSankeyMode}
						<span class="legend-pill"><span class="dot" style="background: #0284c7;"></span> ≤ SD</span>
						<span class="legend-pill"><span class="dot" style="background: #38bdf8;"></span> SMP</span>
						<span class="legend-pill"><span class="dot" style="background: #f97316;"></span> SMA/SMK</span>
						<span class="legend-pill"><span class="dot" style="background: #1e3a8a;"></span> Perguruan Tinggi</span>
						<span class="legend-pill"><span class="dot" style="background: #fcd34d;"></span> Indonesia (asal)</span>
					{:else}
						<span class="legend-pill"><span class="dot top3"></span> 3 Terbesar Asia Timur/Tenggara</span>
						<span class="legend-pill"><span class="dot skilled"></span> Vokasi G to G (Jepang & Korsel)</span>
						<span class="legend-pill"><span class="dot europe"></span> Eropa & Timur Tengah</span>
					{/if}
				</div>
			</div>

			<div class="canvas-split-layout">
				<!-- PETA SVG UTAMA -->
				<div class="svg-container">
					<svg
						viewBox="{view.x} {view.y} {view.w} {view.h}"
						preserveAspectRatio="xMidYMid meet"
					>
						<defs>
							<pattern id="grid" width="40" height="40" patternUnits="userSpaceOnUse">
								<path d="M 40 0 L 0 0 0 40" fill="none" stroke="#f1f5f9" stroke-width="0.8" />
							</pattern>
							<filter id="spike-shadow" x="-30%" y="-30%" width="160%" height="160%">
								<feDropShadow dx="1" dy="3" stdDeviation="2" flood-opacity="0.25" flood-color="#0f172a" />
							</filter>
						</defs>

						<rect width="100%" height="100%" fill="url(#grid)" />

						<!-- LAYER DARATAN DUNIA (polygon negara tujuan diwarnai saat Sankey Mode) -->
<g class="countries-layer">
	{#each countryShapes as shape}
		{@const st = polygonStyle(shape.code)}
		{@const interactive = isSankeyMode && !!shape.code && !!eduDataByCountry[shape.code]}
		<!-- svelte-ignore a11y_no_static_element_interactions -->
		<path
			d={shape.d}
			fill={st.fill}
			fill-opacity={st.opacity}
			stroke="#ffffff"
			stroke-width="0.6"
			class="country-polygon"
			class:interactive
			onmouseenter={interactive
				? (e) => {
						hoveredDest = destById[shape.code];
						tooltipPos = { x: e.clientX, y: e.clientY };
					}
				: undefined}
			onmousemove={interactive ? (e) => (tooltipPos = { x: e.clientX, y: e.clientY }) : undefined}
			onmouseleave={interactive ? () => (hoveredDest = null) : undefined}
		/>
	{/each}
</g>

{#if mapError}
	<text x={view.x + view.w / 2} y={view.y + view.h / 2} text-anchor="middle" class="map-error">
		Peta dunia gagal dimuat (cek /static/data/world-countries.json)
	</text>
{/if}

						<!-- ============================================== -->
						<!-- MODE 1: FLOW ARCS & 3D BARS (STEP 0 - 4)        -->
						<!-- ============================================== -->
						<g class="mode-flow-layer" style="opacity: {isSankeyMode ? 0 : 1}; pointer-events: {isSankeyMode ? 'none' : 'auto'}; transition: opacity 0.6s ease;">
							<!-- Flight Arcs -->
							<g class="flight-arcs-layer">
								{#each flowData.destinations as dest}
									{@const curve = getFlightCurve(flowData.origin.coords, dest.coords)}
									{@const style = getDestStyle(dest, activeStep)}
									{#if curve}
										<path
											d={curve}
											fill="none"
											stroke={style.color}
											stroke-width={style.width}
											stroke-opacity={style.opacity}
											class="flight-path"
											class:active-flow={style.active}
										/>
									{/if}
								{/each}
							</g>

							<!-- Titik Asal Jakarta -->
							{#if originPos}
								<g transform="translate({originPos[0]}, {originPos[1]})" class="origin-pin">
									<circle r="7" fill="#f97316" opacity="0.3" class="pulse-ring" />
									<circle r="4" fill="#f97316" stroke="#ffffff" stroke-width="1.5" />
									<text y="14" text-anchor="middle" class="pin-label origin-text">Indonesia</text>
								</g>
							{/if}

							<!-- Batang Spikes 3D Menjulang -->
							<g class="spikes-layer">
								{#each flowData.destinations as dest}
									{@const pt = projection(dest.coords)}
									{@const barH = spikeHeightScale(dest.count)}
									{@const style = getDestStyle(dest, activeStep)}
									{#if pt}
										<!-- svelte-ignore a11y_no_static_element_interactions -->
										<g
											transform="translate({pt[0]}, {pt[1]})"
											class="country-spike-node"
											opacity={style.opacity > 0.25 ? 1 : 0.2}
											onmouseenter={(e) => {
												hoveredDest = dest;
												tooltipPos = { x: e.clientX, y: e.clientY };
											}}
											onmouseleave={() => (hoveredDest = null)}
										>
											<ellipse cx="0" cy="0" rx="4" ry="2" fill="rgba(15,23,42,0.3)" />
											<polygon
												points="-3,0 0,{-barH} 0,0"
												fill={style.color}
												opacity="0.82"
												filter="url(#spike-shadow)"
											/>
											<polygon
												points="0,{-barH} 3,0 0,0"
												fill={style.color}
												opacity="1"
												filter="url(#spike-shadow)"
											/>
											<circle cx="0" cy={-barH} r="2.5" fill="#ffffff" stroke={style.color} stroke-width="1" />

											{#if showLabel(dest, style, activeStep)}
												<text
													x="5"
													y={-barH - 4}
													class="spike-label"
													class:prominent={style.active && activeStep >= 1 && activeStep <= 3}
													style="font-size: {(style.active && activeStep >= 1 && activeStep <= 3 ? 8.5 : 7.5) / zoomK}px; stroke-width: {2.5 / zoomK}px;"
												>
													{dest.name}
												</text>
											{/if}
										</g>
									{/if}
								{/each}
							</g>
						</g>

						<!-- ============================================== -->
<!-- MODE 2: SANKEY LINKED MAP (STEP 5, 6, 7)        -->
<!-- Balok pendidikan di bawah peta -> pita langsung -->
<!-- ke lokasi negara tujuan (tanpa kolom tengah)    -->
<!-- ============================================== -->
<g
	class="mode-sankey-layer"
	style="opacity: {isSankeyMode ? 1 : 0}; pointer-events: {isSankeyMode ? 'auto' : 'none'}; transition: opacity 0.6s ease {isSankeyMode ? '0.25s' : '0s'};"
>
	<!-- 1. Pita aliran: jenjang pendidikan -> titik negara tujuan -->
	<g class="sankey-links">
		{#each renderLinks as link}
			{@const op = linkOpacity(link)}
			<!-- svelte-ignore a11y_no_static_element_interactions -->
			<path
				d={link.path}
				fill={link.color}
				fill-opacity={op}
				stroke={link.color}
				stroke-width="0.4"
				stroke-opacity={op * 0.6}
				class="sankey-ribbon"
				onmouseenter={(e) => {
					hoveredLink = link;
					tooltipPos = { x: e.clientX, y: e.clientY };
				}}
				onmousemove={(e) => (tooltipPos = { x: e.clientX, y: e.clientY })}
				onmouseleave={() => (hoveredLink = null)}
			/>
		{/each}
	</g>

	<!-- 2. Label Indonesia sebagai asal -->
	{#if originPos}
		<text x={originPos[0]} y={originPos[1] + 4} text-anchor="middle" class="origin-label">
			Indonesia (asal)
		</text>
	{/if}

	<!-- 3. Penanda negara tujuan di peta -->
	<g class="sankey-dest-markers">
		{#each sankeyLayout.destNodes as n}
			{@const cfg = labelCfg[n.id] ?? { dx: 0, dy: -(n.r + 5), anchor: "middle" }}
			{@const dim = !(isCountryFocused(n.id) || hoveredDest?.id === n.id)}
			<!-- svelte-ignore a11y_no_static_element_interactions -->
			<g
				transform="translate({n.x}, {n.y})"
				class="dest-marker"
				opacity={dim ? 0.35 : 1}
				onmouseenter={(e) => {
					hoveredDest = destById[n.id];
					tooltipPos = { x: e.clientX, y: e.clientY };
				}}
				onmousemove={(e) => (tooltipPos = { x: e.clientX, y: e.clientY })}
				onmouseleave={() => (hoveredDest = null)}
			>
				{#if !matchedCodes.has(n.id)}
					<!-- Polygon negara kecil tidak ada di dataset peta: beri halo berwarna -->
					<circle r={n.r + 7} fill={n.dominant.color} opacity="0.3" />
				{/if}
				<circle r={n.r} fill="#ffffff" stroke={n.dominant.color} stroke-width="2" />
				<circle r={n.r * 0.45} fill={n.dominant.color} />
				<text x={cfg.dx} y={cfg.dy} text-anchor={cfg.anchor} class="sankey-country-label">
					{n.name}
				</text>
			</g>
		{/each}
	</g>

	<!-- 4. Balok jenjang pendidikan (sumber pita) -->
	<g class="sankey-edu-nodes">
		{#each sankeyLayout.eduNodes as node}
			<!-- svelte-ignore a11y_no_static_element_interactions -->
			<g
				class="edu-node-group"
				onmouseenter={(e) => {
					hoveredEdu = node.id;
					tooltipPos = { x: e.clientX, y: e.clientY };
				}}
				onmousemove={(e) => (tooltipPos = { x: e.clientX, y: e.clientY })}
				onmouseleave={() => (hoveredEdu = null)}
			>
				<!-- area hover diperlebar (balok PT sangat tipis) -->
				<rect
					x={node.x0 - 5}
					y={BAR_SIDE === "top" ? node.y0 - 30 : node.y0 - 4}
					width={node.w + 10}
					height={node.y1 - node.y0 + 34}
					fill="transparent"
				/>
				<rect
					x={node.x0}
					y={node.y0}
					width={node.w}
					height={node.y1 - node.y0}
					fill={node.color}
					rx="2"
				/>
				<text x={(node.x0 + node.x1) / 2} y={BAR_SIDE === "top" ? node.y0 - 20 : node.y1 + 12} text-anchor="middle" class="sankey-edu-label">
					{node.label}
				</text>
				<text x={(node.x0 + node.x1) / 2} y={BAR_SIDE === "top" ? node.y0 - 11 : node.y1 + 21} text-anchor="middle" class="sankey-edu-sub">
					{node.share.toLocaleString("id-ID", { maximumFractionDigits: 1 })}%
				</text>
			</g>
		{/each}
	</g>
</g>
					</svg>
				</div>

				<!-- PANEL MINI DIAGRAM BATANG SISI KANAN (Hanya aktif di Mode 1) -->
				<div
					class="sidebar-ranking-panel"
					class:collapsed={isSankeyMode}
				>
					<div class="panel-header">
						<h4>Top 15 Negara Tujuan</h4>
						<span class="subtext">Orang, 2023 · skala akar</span>
					</div>
					<div class="ranking-list">
						{#each sortedDestinations as dest}
							{@const style = getDestStyle(dest, activeStep)}
							{@const pct = Math.max(barScale(dest.count), 3)}
							<!-- svelte-ignore a11y_no_static_element_interactions -->
							<div
								class="rank-row"
								class:is-active={style.active && activeStep !== 0}
								class:is-dimmed={style.opacity < 0.3}
								onmouseenter={(e) => {
									hoveredDest = dest;
									tooltipPos = { x: e.clientX, y: e.clientY };
								}}
								onmouseleave={() => (hoveredDest = null)}
							>
								<span class="rank-name">{dest.name}</span>
								<div class="rank-bar-track">
									<div
										class="rank-bar-fill"
										style="width: {pct}%; background-color: {style.color};"
									></div>
								</div>
							</div>
						{/each}
					</div>
				</div>
			</div>

			<div class="map-footer-note">
				{#if isSankeyMode}
				Sumber: BPS (Long Form SP2020) untuk persentase jenjang, BP2MI 2023 untuk jumlah pekerja; lebar pita = jumlah x persentase (perkiraan), enam negara yang dirinci BPS | Warna negara = jenjang terbanyak | Hover pita, balok, atau negara
			{:else}
				Sumber Data: Penempatan BP2MI 2023 & BPS | Interaksi: Hover negara atau diagram batang untuk rincian
			{/if}
			</div>
		</div>

		<!-- KARTU CERITA SCROLLYTELLING -->
		<div class="scrolly-overlay">
			<Scrolly bind:value={activeStep} top={140} bottom={140}>
				<!-- TAHAP 1: PETA & ARUS PENEMPATAN -->
				<div class="story-card" class:active={activeStep === 0}>
					<h4 class="card-title">Peta Sebaran & Menara Migrasi</h4>
					<p>
						Sepanjang tahun 2023, sebanyak <strong>274.965 Pekerja Migran Indonesia (PMI)</strong> berangkat
						ke berbagai penjuru dunia.
						<strong>Lebih dari 98% pekerja hanya terserap di 15 negara tujuan utama ini</strong>.
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 1}>
				
					<h4 class="card-title">Dominasi 80% di Asia Timur & Tenggara</h4>
					<p>
						Tiga destinasi mendominasi hampir 80% dari seluruh penempatan:
						<strong class="color-red">Taiwan</strong> (83.216),
						<strong class="color-red">Malaysia</strong> (72.260), dan
						<strong class="color-red">Hong Kong</strong> (65.916).
						Tingginya permintaan perawat lansia dan pekerja pabrik manufaktur menjadi daya pikat utama kawasan ini.
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 2}>
					<h4 class="card-title">Jepang & Korsel</h4>
					<p>
						Bergeser ke utara, <strong class="color-cyan">Korea Selatan</strong> (12.580 PMI) dan
						<strong class="color-cyan">Jepang</strong> (9.673 PMI) menawarkan upah tinggi melalui skema
						resmi antar-pemerintah (G to G) serta visa keahlian khusus (SSW) di sektor manufaktur dan medis.
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 3}>
					<span class="card-badge hl-purple">EROPA & TIMUR TENGAH</span>
			
					<p>
						Jalur migrasi kini berkembang ke benua Eropa seperti
						<strong class="color-purple">Italia</strong> (3.519 PMI),
						<strong class="color-purple">Polandia</strong> (1.524 PMI), hingga
						<strong class="color-purple">Hungaria</strong> (447 PMI) di bidang perhotelan dan spa,
						di samping arus pekerja formal di <strong class="color-purple">Arab Saudi</strong> (6.310 PMI) dan <strong class="color-purple">Turki</strong> (2.289 PMI).
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 4}>
	
					<p>
						Mari kita lihat negara mana yang paling banyak menyerap lulusan pendidikan dasar, dan negara mana yang mensyaratkan ijazah vokasi atau tinggi?
					</p>
				</div>

				<!-- TAHAP 2: SANKEY LINKED TO MAP (PENDIDIKAN -> NEGARA -> PETA) -->
				<div class="story-card" class:active={activeStep === 5}>
	
					<h4 class="card-title">Peta Aliran Berdasarkan Jenjang Pendidikan</h4>
					<p>
						Berdasarkan hasil survei <em>Long Form Sensus Penduduk 2020 BPS</em>, profil migran Indonesia terbagi tegas
						menjadi koridor low-skilled dan skilled. Setiap pita mengalir dari jenjang sekolah di bawah peta langsung ke negara tujuannya. Semakin tebal pita, semakin banyak pekerjanya, dan warna negara menunjukkan jenjang yang paling dominan di sana.
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 6}>
					<span class="card-badge hl-blue">KORIDOR PENDIDIKAN DASAR</span>
					<h4 class="card-title">Arab Saudi & Malaysia Menyerap Lulusan SD–SMP</h4>
					<p>
						Lebih dari <strong class="color-blue">52,3% PMI di Arab Saudi</strong> dan
						<strong class="color-blue">43,1% di Malaysia</strong> berpendidikan maksimal SD.
						Bersama lulusan SMP, kedua negara ini menyerap proporsi terbesar pekerja domestik, perkebunan sawit, dan konstruksi fisik.
					</p>
				</div>

				<div class="story-card" class:active={activeStep === 7}>
					<span class="card-badge hl-orange">KORIDOR MENENGAH & TINGGI</span>
					<h4 class="card-title">Jepang: 96,7% Lulusan SMA/SMK & Sarjana</h4>
					<p>
						Sebaliknya, <strong class="color-orange">83,4% pekerja di Jepang</strong> berlatar belakang SMA/SMK,
						serta <strong>13,3% dari Perguruan Tinggi</strong>. Standar sertifikasi bahasa dan vokasi teknis menjadikan Jepang pusat serapan tenaga terdidik.
					</p>
				</div>
			<div class="story-card" class:active={activeStep === 8}>
		
					<h4 class="card-title">Ijazah Menentukan Arah Berangkat</h4>
					<p>
						<strong class="color-blue">Malaysia</strong> dan
						<strong class="color-blue">Arab Saudi</strong> banyak menyerap lulusan pendidikan dasar,
						<strong>Taiwan</strong> dan <strong>Hong Kong</strong> didominasi lulusan SMP hingga SMA/SMK,
						sedangkan <strong class="color-orange">Jepang</strong> hampir seluruhnya lulusan SMA/SMK dan perguruan tinggi.
					</p>
				</div>
			</Scrolly>
		</div>
	</div>

	<!-- TOOLTIP -->
{#if isSankeyMode && hoveredLink}
	<div class="flow-tooltip" style="top: {tooltipPos.y + 12}px; left: {tooltipPos.x + 14}px;">
		<div class="tt-header">{hoveredLink.eduLabel} → {hoveredLink.countryName}</div>
		<div class="tt-val">{fmtInt(hoveredLink.val)} Pekerja</div>
		<div class="tt-meta">{fmtPct(hoveredLink.pct)}% dari PMI di {hoveredLink.countryName}</div>
	</div>
{:else if isSankeyMode && hoveredEdu}
	{@const en = sankeyLayout.eduNodes.find((n) => n.id === hoveredEdu)}
	{#if en}
		<div class="flow-tooltip" style="top: {tooltipPos.y + 12}px; left: {tooltipPos.x + 14}px;">
			<div class="tt-header">Jenjang {en.label}</div>
			<div class="tt-val">{fmtInt(en.total)} Pekerja</div>
			<div class="tt-meta">{fmtPct(en.share)}% dari PMI enam negara tujuan</div>
		</div>
	{/if}
{:else if hoveredDest}
	<div class="flow-tooltip" style="top: {tooltipPos.y + 12}px; left: {tooltipPos.x + 14}px;">
		<div class="tt-header">{hoveredDest.name} ({hoveredDest.city})</div>
		<div class="tt-val">{hoveredDest.count.toLocaleString("id-ID")} Pekerja</div>
		{#if isSankeyMode && eduDataByCountry[hoveredDest.id]}
			{@const p = eduDataByCountry[hoveredDest.id]}
			<div class="tt-edu-breakdown">
				{#each eduCategories as e}
					<div class="edu-line">
						<span class="edu-dot" style="background:{e.color}"></span>
						{e.label}: <strong>{fmtPct(p[e.id])}%</strong>
					</div>
				{/each}
			</div>
		{:else}
			<div class="tt-meta">Kawasan: {hoveredDest.region}</div>
			<div class="tt-meta">Sektor: {hoveredDest.sector}</div>
			<div class="tt-gender">♀ {hoveredDest.female_pct}% Perempuan | ♂ {hoveredDest.male_pct}% Laki-laki</div>
		{/if}
	</div>
{/if}
</section>

<style>


	.word {
		display: inline-block;
		white-space: nowrap;
		margin: 0 0.2em;
	}

	.letter {
		display: inline-block;
	}
	
	.flow-map-section {
		width: 100%;
		background: #f8fafc;
		position: relative;
	}

	.section-intro {
		max-width: 680px;
		margin: 0 auto;
		text-align: center;
		background: #ffffff;
	}

	.lead-text {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.8;
		color: #374151;
		margin: 0;
	}

	.scrolly-container {
		position: relative;
		width: 100%;
	}

	.sticky-map-viewport {
		position: sticky;
		top: 0;
		left: 0;
		width: 100%;
		height: 100vh;
		display: flex;
		flex-direction: column;
		background: #ffffff;
		overflow: hidden;
		z-index: 1;

	}

	.map-top-bar {
		padding: 0.9rem 2rem 0.5rem;
		display: flex;
		justify-content: space-between;
		align-items: flex-end;
		background: #ffffff;
		border-bottom: 1px solid #f1f5f9;
		z-index: 5;
	}

	.badge-tag {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		font-weight: 700;
		color: #64748b;
		letter-spacing: 0.06em;
	}

	.view-title {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.15rem;
		font-weight: 700;
		color: #0f172a;
		margin: 0.2rem 0 0;
	}

	.map-legend-pills {
		display: flex;
		gap: 0.8rem;
	}

	.legend-pill {
		font-size: 0.72rem;
		font-family: var(--sans, sans-serif);
		color: #475569;
		display: flex;
		align-items: center;
		gap: 0.35rem;
	}

	.dot {
		width: 9px;
		height: 9px;
		border-radius: 50%;
		display: inline-block;
	}
	.dot.top3 { background: #e11d48; }
	.dot.skilled { background: #0891b2; }
	.dot.europe { background: #8b5cf6; }

	.canvas-split-layout {
		display: flex;
		flex: 1;
		width: 100%;
		height: calc(100vh - 110px);
		position: relative;
	}

	.svg-container {
		flex: 1;
		height: 100%;
		position: relative;
	}

	svg {
		width: 100%;
		height: 100%;
	}

	.country-polygon {
		transition: fill 0.45s ease, stroke 0.3s ease;
	}

	/* Flight Arc Styles */
	.flight-path {
		transition: stroke-opacity 0.5s, stroke-width 0.5s, stroke 0.5s;
		pointer-events: none;
	}

	.flight-path.active-flow {
		stroke-dasharray: 6 3;
		animation: flightFlow 1.6s linear infinite;
	}

	@keyframes flightFlow {
		to {
			stroke-dashoffset: -18;
		}
	}

	.origin-pin .pulse-ring {
		animation: pulseJakarta 2s ease-out infinite;
	}

	@keyframes pulseJakarta {
		0% { r: 4; opacity: 0.7; }
		100% { r: 16; opacity: 0; }
	}

	.pin-label {
		font-family: var(--sans, sans-serif);
		font-size: 10px;
		font-weight: 700;
		fill: #f97316;
		paint-order: stroke;
		stroke: #ffffff;
		stroke-width: 2.5px;
	}

	.country-spike-node {
		cursor: pointer;
		transition: opacity 0.5s;
	}

	.spike-label {
		font-family: var(--sans, sans-serif);
		fill: #1e293b;
		font-weight: 600;
		paint-order: stroke;
		stroke: #ffffff;
		pointer-events: none;
	}

	.spike-label.prominent {
		font-weight: 800;
		fill: #0f172a;
	}

	/* SANKEY STYLES */
	.country-polygon.interactive {
		cursor: pointer;
	}

	.sankey-ribbon {
		cursor: pointer;
		transition: fill-opacity 0.35s ease, stroke-opacity 0.35s ease;
	}

	.dest-marker,
	.edu-node-group {
		cursor: pointer;
		transition: opacity 0.4s ease;
	}

	.sankey-edu-label {
		font-family: var(--sans, sans-serif);
		font-size: 8.5px;
		font-weight: 700;
		fill: #1e293b;
	}

	.sankey-edu-sub {
		font-family: var(--mono, monospace);
		font-size: 6.5px;
		fill: #64748b;
	}

	.sankey-country-label {
		font-family: var(--sans, sans-serif);
		font-size: 8px;
		font-weight: 700;
		fill: #0f172a;
		paint-order: stroke;
		stroke: #ffffff;
		stroke-width: 2.5px;
		stroke-linejoin: round;
		pointer-events: none;
	}

	.origin-label {
		font-family: var(--sans, sans-serif);
		font-size: 7px;
		font-weight: 700;
		fill: #92400e;
		paint-order: stroke;
		stroke: #ffffff;
		stroke-width: 2.2px;
		pointer-events: none;
	}

	.map-error {
		font-family: var(--sans, sans-serif);
		font-size: 10px;
		fill: #b91c1c;
	}

	/* Sidebar Ranking Mini Chart (menyusut mulus saat masuk Sankey Mode) */
	.sidebar-ranking-panel {
		width: 240px;
		flex-shrink: 0;
		height: 100%;
		padding: 0.9rem 1rem;
		display: flex;
		flex-direction: column;
		z-index: 5;
		overflow: hidden;
		transition: width 0.6s cubic-bezier(0.2, 0.8, 0.2, 1), padding 0.6s cubic-bezier(0.2, 0.8, 0.2, 1),
			opacity 0.4s ease, border-width 0.6s ease;
	}

	.sidebar-ranking-panel > * {
		min-width: 208px;
	}

	.sidebar-ranking-panel.collapsed {
		width: 0;
		padding-left: 0;
		padding-right: 0;
		border-left-width: 0;
		opacity: 0;
		pointer-events: none;
	}

	.panel-header h4 {
		font-size: 0.85rem;
		font-weight: 700;
		color: #1e293b;
		margin: 0;
	}

	.panel-header .subtext {
		font-size: 0.68rem;
		color: #94a3b8;
		font-family: var(--mono, monospace);
		display: block;
		margin-bottom: 0.7rem;
	}

	.ranking-list {
		flex: 1;
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		gap: 3px;
		overflow-y: auto;
	}

	.rank-row {
		display: flex;
		align-items: center;
		gap: 6px;
		font-size: 0.72rem;
		padding: 2px 4px;
		border-radius: 3px;
		cursor: pointer;
		transition: background 0.2s, opacity 0.3s;
	}

	.rank-row:hover,
	.rank-row.is-active {
		background: #f1f5f9;
		font-weight: 700;
	}

	.rank-row.is-dimmed {
		opacity: 0.25;
	}

	.rank-name {
		width: 104px;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		color: #334155;
	}

	.rank-bar-track {
		flex: 1;
		height: 8px;
		background: #e9eef4;
		border-radius: 2px;
		overflow: hidden;
	}

	.rank-bar-fill {
		height: 100%;
		border-radius: 2px;
		transition: width 0.4s ease;
	}

	.map-footer-note {
		padding: 0.4rem 2rem;
		font-size: 0.68rem;
		font-family: var(--mono, monospace);
		color: #94a3b8;
		background: #ffffff;
		border-top: 1px solid #f1f5f9;
	}

	/* Scrolly Card Overlay */
	.scrolly-overlay {
		position: relative;
		z-index: 10;
		display: flex;
		flex-direction: column;
		align-items: flex-start;
		margin-top: -100vh;
		padding: 0 3rem 40vh 6.5rem;
		pointer-events: none;
	}

	.story-card {
		pointer-events: auto;
		background: rgba(255, 255, 255, 0.94);
		backdrop-filter: blur(10px);
		-webkit-backdrop-filter: blur(10px);
		border: 1px solid rgba(226, 232, 240, 0.9);
		box-shadow: 0 10px 28px rgba(0, 0, 0, 0.07);
		max-width: 440px;
		padding: 1.5rem 1.8rem;
		border-radius: 6px;
		margin: 55vh 0;
		transition: transform 0.3s, border-color 0.3s;
	}

	.story-card.active {
		border-color: #94a3b8;
		transform: scale(1.02);
	}

	.card-badge {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		letter-spacing: 0.08em;
		color: #64748b;
		display: inline-block;
		margin-bottom: 0.4rem;
		border-bottom: 1px solid #e2e8f0;
		padding-bottom: 2px;
	}

	.hl-purple { color: #7c3aed !important; border-color: #7c3aed !important; }
	.hl-blue { color: #0284c7 !important; border-color: #0284c7 !important; }
	.hl-orange { color: #f97316 !important; border-color: #f97316 !important; }

	.card-title {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.22rem;
		font-weight: 700;
		color: #0f172a;
		margin: 0 0 0.65rem;
	}

	.story-card p {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 0.98rem;
		line-height: 1.75;
		color: #334155;
		margin: 0;
		text-align: justify;
	}

	.color-red { color: #e11d48; font-weight: 700; }
	.color-cyan { color: #0891b2; font-weight: 700; }
	.color-purple { color: #7c3aed; font-weight: 700; }
	.color-blue { color: #0284c7; font-weight: 700; }
	.color-orange { color: #f97316; font-weight: 700; }

	/* Tooltip */
	.flow-tooltip {
		position: fixed;
		pointer-events: none;
		z-index: 1000;
		background: #0f172a;
		color: #f8fafc;
		padding: 0.65rem 0.95rem;
		border-radius: 5px;
		font-family: var(--sans, sans-serif);
		box-shadow: 0 8px 24px rgba(0, 0, 0, 0.28);
	}

	.tt-header { font-weight: 700; font-size: 0.85rem; color: #38bdf8; margin-bottom: 2px; }
	.tt-val { font-size: 0.9rem; font-weight: 800; color: #ffffff; margin-bottom: 4px; }
	.tt-meta { font-size: 0.73rem; color: #cbd5e1; margin-bottom: 2px; }

	.tt-edu-breakdown {
		margin-top: 6px;
		padding-top: 6px;
		border-top: 1px solid #334155;
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 3px 8px;
		font-size: 0.7rem;
	}
	.edu-line {
		display: flex;
		align-items: center;
		gap: 4px;
		color: #cbd5e1;
	}
	.edu-dot {
		width: 6px;
		height: 6px;
		border-radius: 50%;
		display: inline-block;
	}
</style>