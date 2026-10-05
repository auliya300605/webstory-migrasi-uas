<script>
	// Tambahkan fungsi RNG deterministik & pemecah huruf ala S07
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

	const TITLE_S03 = "Dari Mana Wilayah Asal Pekerja Migran Indonesia?";
	const titleWordsS03 = (() => {
		const rnd = rng(hashStr(TITLE_S03));
		return TITLE_S03.split(" ").map((w) =>
			[...w].map((ch) => ({
				ch,
				dy: r2(rnd() * 3.2),
				rot: r2(rnd() * 2.2 - 0.2)
			}))
		);
	})();

	const SQUIGGLE_D =
		"M3 13.5C22.5 7.5 45.2 6.8 65.5 12.2C85.8 17.6 106.1 18.2 125.8 11.8C131.2 10.1 136.5 7.8 141 5";
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import mapData from "$data/s03_data.json";

	let currentStep = $state(0);
	let selectedMonth = $state("JUMLAH");
	let activeTooltip = $state(null);
	let tooltipPos = $state({ x: 0, y: 0 });

	// $state.raw: GeoJSON besar tidak perlu dibungkus proxy reaktif (lebih ringan)
	let geoData = $state.raw(null);
	let kabGeoData = $state.raw(null);
	const width = 960;
	const height = 520;

	// ===== PENGATURAN ZOOM & TAMPILAN (ubah di sini kalau perlu) =====
	// true  -> zoom hanya ke 19 sentra di Pulau Jawa (6 sentra luar Jawa bisa terpotong)
	// false -> zoom memuat ke-25 sentra
	const FIT_ONLY_JAVA = false;
	// Ruang kosong di tepi area peta (satuan viewBox 960x520).
	// "top" dibuat besar karena ada header bulan + legenda di atas peta.
	// Kalau kartu narasi menutupi peta, besarkan "left" lalu geser kartu ke kiri lewat CSS.
	const FIT_PADDING = { top: 90, right: 50, bottom: 40, left: 50 };
	const MAX_ZOOM = 9;
	const BUBBLE_RANGE = [4, 16]; // radius bubble dalam piksel layar (bukan satuan peta)
	const LABEL_TOP_N = 8; // jumlah kab/kota yang diberi label nama

	const months = [
		{ key: "JUMLAH", label: "Total 2023" },
		{ key: "JAN", label: "Jan" },
		{ key: "FEB", label: "Feb" },
		{ key: "MAR", label: "Mar" },
		{ key: "APR", label: "Apr" },
		{ key: "MEI", label: "Mei" },
		{ key: "JUN", label: "Jun" },
		{ key: "JUL", label: "Jul" },
		{ key: "AGU", label: "Agu" },
		{ key: "SEP", label: "Sep" },
		{ key: "OKT", label: "Okt" },
		{ key: "NOV", label: "Nov" },
		{ key: "DES", label: "Des" }
	];

	// Palet warna Country Radio Editorial (Colorblind Safe)
	const colorChoroplethPMI = d3.scaleThreshold()
		.domain([1.5, 5, 12, 20])
		.range(["#f2ede4", "#e8c8a7", "#e79a3c", "#d95f43", "#9e3424"]);

	const colorKemiskinan = d3.scaleThreshold()
		.domain([8, 10, 12, 14])
		.range(["#e5f5f3", "#a7dbd5", "#4ea59f", "#2b847e", "#134e4a"]);

	const colorPDRB = d3.scaleThreshold()
		.domain([25, 40, 60, 90])
		.range(["#fef3df", "#fbd38d", "#e79a3c", "#c05621", "#7b341e"]);

	const projection = d3.geoMercator().center([118, -2.5]).scale(1100).translate([width / 2, height / 2]);
	const path = d3.geoPath().projection(projection);

	const fmt = (v) => Number(v).toLocaleString("id-ID", { maximumFractionDigits: 1 });

	// Apakah data kab/kota punya kolom rasio PMI? Kalau tidak, bubble & warna step 1
	// memakai jumlah PMI sebagai cadangan.
	   const hasKabRatio =
   	        mapData.kabupaten.length > 0 &&
        	mapData.kabupaten.every((k) => typeof k.populasi_2023 === "number");

	onMount(() => {
		async function loadJson(url) {
			try {
				const res = await fetch(url);
				return await res.json();
			} catch (e) {
				console.warn("Gagal memuat GeoJSON:", url, e);
				return null;
			}
		}
		loadJson("/data/indonesia-prov.json").then((d) => (geoData = d));
		loadJson("/data/indonesia-kab.json").then((d) => (kabGeoData = d));

		// Observer scrollytelling card step
		const cards = document.querySelectorAll(".scrolly-step-card");
		const stepObserver = new IntersectionObserver((entries) => {
			entries.forEach((entry) => {
				if (entry.isIntersecting) {
					currentStep = parseInt(entry.target.dataset.step);
				}
			});
		}, { rootMargin: "-30% 0px -30% 0px", threshold: 0.2 });

		cards.forEach((card) => stepObserver.observe(card));

		return () => stepObserver.disconnect();
	});

	/* ---------- Pencocokan nama provinsi ---------- */
	function getProvRate(provName) {
		const clean = (provName || "").toLowerCase().replace(/[^a-z]/g, "");
		const match = mapData.provinsi.find(p => {
			const target = p.provinsi.toLowerCase().replace(/[^a-z]/g, "");
			return clean.includes(target) || target.includes(clean);
		});
		if (!match) return { rate: 0, total: 0 };
		const rate = selectedMonth === "JUMLAH" ? match.rasio_per_10rb : match.monthly_rasio[selectedMonth];
		const total = selectedMonth === "JUMLAH" ? match.total : match.monthly[selectedMonth];
		return { rate, total };
	}

	/* ---------- Pencocokan kab/kota: data PMI <-> polygon GeoJSON ---------- */
	// Kunci properti nama kab/kota yang dicoba berurutan (sesuaikan kalau GeoJSON-mu beda).
	const KAB_NAME_KEYS = ["KABUPATEN", "NAME_2", "WADMKK", "KABKOT", "kabkot", "KAB_KOTA", "kab_kota", "NAMOBJ", "nama_kab", "name", "NAME", "Kabupaten", "kabupaten", "regency", "district"];

	function kabFeatureName(f) {
		const p = f.properties || {};
		for (const k of KAB_NAME_KEYS) if (p[k]) return String(p[k]);
		return "";
	}

	// "Kota Bandung" dan "Kabupaten Bandung" TIDAK boleh dianggap sama -> simpan base + penanda kota.
	function parseKabName(raw, props = {}) {
		const s = String(raw || "").toLowerCase();
		const t = String(props.TYPE_2 || props.ENGTYPE_2 || props.type || "").toLowerCase();
		const isKota = /\bkota\b|\(kota\)/.test(s) || t.includes("kota") || t.includes("city");
        const base = s.replace(/\(.*?\)/g, "").replace(/kabupaten|\bkab\b\.?|kota administrasi|kota adm\.?|\bkota\b|[^a-z]/g, "");
		return { base, isKota };
	}

	// Gabungan: data PMI + polygon + titik pusat (dalam koordinat peta)
	let sentra = $derived.by(() => {
		if (!kabGeoData) return [];

		const feats = kabGeoData.features.map((f) => ({
			f,
			...parseKabName(kabFeatureName(f), f.properties)
		}));
		if (feats.length && !feats[0].base) {
			console.warn("Nama kab/kota tidak terbaca. Properti GeoJSON:", Object.keys(kabGeoData.features[0].properties || {}));
		}
		const byBase = new Map();
		feats.forEach((x) => {
			if (!byBase.has(x.base)) byBase.set(x.base, []);
			byBase.get(x.base).push(x);
		});

		const out = [];
		const unmatched = [];
		for (const d of mapData.kabupaten) {
			const { base, isKota } = parseKabName(d.kabupaten);
			const cands = byBase.get(base) || [];
			let pick = null;
			if (cands.length === 1) pick = cands[0];
			else if (cands.length > 1) pick = cands.find((c) => c.isKota === isKota) ?? cands[0];
			if (!pick) {
				unmatched.push(d.kabupaten);
				continue;
			}
			out.push({ data: d, f: pick.f, c: path.centroid(pick.f) });
		}
		if (unmatched.length) console.warn("Kab/kota TIDAK ketemu di GeoJSON:", unmatched);
		console.info(`Sentra cocok: ${out.length} dari ${mapData.kabupaten.length}`);
		return out;
	});

	/* ---------- Nilai metrik kab/kota ---------- */
	function kabTotal(k) {
		return selectedMonth === "JUMLAH" ? k.total : (k.monthly?.[selectedMonth] ?? 0);
	}
	// Metrik utama: rasio PMI per 10rb penduduk kalau ada, kalau tidak jumlah PMI
	function kabMetric(k) {
		if (hasKabRatio) {
   	        const n = kabTotal(k);
   	        return (n / k.populasi_2023) * 10000;
        }
	}

	// Skala warna step 1: batas kelas dihitung dari kuantil 25 sentra (supaya variasinya terlihat)
	let colorKabPMI = $derived.by(() => {
		const vals = sentra.map((s) => kabMetric(s.data)).filter((v) => Number.isFinite(v));
		const palette = colorChoroplethPMI.range();
		if (vals.length < 5) return d3.scaleThreshold().domain([]).range([palette[0]]);
		const breaks = d3.scaleQuantile().domain(vals).range(palette).quantiles();
		return d3.scaleThreshold().domain(breaks).range(palette);
	});

	// Skala radius bubble (piksel layar)
	let rScale = $derived.by(() => {
		const max = d3.max(sentra, (s) => kabMetric(s.data)) || 1;
		return d3.scaleSqrt().domain([0, max]).range(BUBBLE_RANGE);
	});

	function kabFill(s) {
		if (currentStep === 1) return colorKabPMI(kabMetric(s.data));
		if (currentStep === 2) return colorKemiskinan(s.data.p0_kemiskinan);
		return colorPDRB(s.data.pdrb_perkapita);
	}

	/* ---------- Zoom otomatis ke 25 sentra ---------- */
	let focusFeatures = $derived(
		sentra.filter((s) => !FIT_ONLY_JAVA || s.data.is_java).map((s) => s.f)
	);

	let zoom = $derived.by(() => {
		if (currentStep === 0) return { k: 1, tx: 0, ty: 0 };
		if (!focusFeatures.length) return { k: 2.4, tx: -305, ty: -600 }; // cadangan: nilai lama
		const [[x0, y0], [x1, y1]] = path.bounds({ type: "FeatureCollection", features: focusFeatures });
		const w = width - FIT_PADDING.left - FIT_PADDING.right;
		const h = height - FIT_PADDING.top - FIT_PADDING.bottom;
		const k = Math.min(w / Math.max(x1 - x0, 1), h / Math.max(y1 - y0, 1), MAX_ZOOM);
		const cx = (x0 + x1) / 2;
		const cy = (y0 + y1) / 2;
		return {
			k,
			tx: FIT_PADDING.left + w / 2 - k * cx,
			ty: FIT_PADDING.top + h / 2 - k * cy
		};
	});

	let mapTransform = $derived(`translate(${zoom.tx}px, ${zoom.ty}px) scale(${zoom.k})`);
	// Koordinat peta -> koordinat layar (untuk label yang ukurannya tidak ikut membesar)
	const sx = (x) => x * zoom.k + zoom.tx;
	const sy = (y) => y * zoom.k + zoom.ty;

	let bubbles = $derived(
		[...sentra].sort((a, b) => kabMetric(b.data) - kabMetric(a.data)) // besar dulu, kecil di atas
	);
	let topLabels = $derived(
		[...sentra]
			.sort((a, b) => kabMetric(b.data) - kabMetric(a.data))
			.filter((s) => !(currentStep === 3 && /karawang/i.test(s.data.kabupaten)))
			.slice(0, LABEL_TOP_N)
	);
	let karawang = $derived(sentra.find((s) => /karawang/i.test(s.data.kabupaten)));

	/* ---------- Legenda otomatis dari skala (selalu sinkron dengan warna peta) ---------- */
	function legendFor(scale) {
		const dom = scale.domain();
		const rng = scale.range();
		if (!dom.length) return [];
		return rng.map((c, i) => ({
			color: c,
			dark: d3.lab(c).l < 55,
			label:
				i === 0 ? `<${fmt(dom[0])}`
				: i === rng.length - 1 ? `>${fmt(dom[dom.length - 1])}`
				: `${fmt(dom[i - 1])}–${fmt(dom[i])}`
		}));
	}

	let monthLabel = $derived(months.find((m) => m.key === selectedMonth)?.label ?? "");
	let sizeNote = hasKabRatio ? "rasio PMI" : "jumlah PMI";

	let legend = $derived(
		currentStep === 0 ? legendFor(colorChoroplethPMI)
		: currentStep === 1 ? legendFor(colorKabPMI)
		: currentStep === 2 ? legendFor(colorKemiskinan)
		: legendFor(colorPDRB)
	);
	let legendTitle = $derived(
		currentStep === 0 ? `Rasio PMI / 10.000 penduduk, provinsi (${monthLabel}):`
		: currentStep === 1 ? (hasKabRatio ? `Rasio PMI / 10.000 penduduk, kab/kota sentra (${monthLabel}):` : `Jumlah PMI, kab/kota sentra (${monthLabel}):`)
		: currentStep === 2 ? `Warna: kemiskinan P0 (%) | Lingkaran: ${sizeNote} (${monthLabel})`
		: `Warna: PDRB/kapita (juta Rp) | Lingkaran: ${sizeNote} (${monthLabel})`
	);

	/* ---------- Tooltip ---------- */
	function showKabTip(e, s) {
		const kab = s.data;
		const parts = [];
		if (currentStep === 2) parts.push(`Kemiskinan: ${fmt(kab.p0_kemiskinan)}%`);
		if (currentStep === 3) parts.push(`PDRB/Kapita: Rp ${fmt(kab.pdrb_perkapita)} Juta`);
		if (hasKabRatio) parts.push(`Rasio PMI: ${fmt(kabMetric(kab))} / 10k pddk`);
		parts.push(`PMI: ${kabTotal(kab).toLocaleString("id-ID")} orang`);
		activeTooltip = {
			title: `${kab.kabupaten} (${kab.provinsi})`,
			info: parts.join(" | "),
			source: "BPS / BP2MI, 2023"
		};
		tooltipPos = { x: e.clientX, y: e.clientY };
	}
</script>

<section id="section-3" class="map-section-container">
	<!-- Judul Section Pembuka -->
	<div class="section-header">
	<span class="kicker"></span>
	<h2 class="section-title" aria-label={TITLE_S03}>
		<span class="title-wrap" aria-hidden="true">
			{#each titleWordsS03 as word}
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

	<!-- Kontainer Sticky Scrollytelling -->
	<div class="scrolly-wrapper">
		<!-- Elemen Peta yang Terkunci Sticky di Layar -->
		<div class="sticky-map">
			<!-- Header Kontrol Waktu & Legenda Dinamis -->
			<div class="map-ui-header">
				<div class="month-selector">
					<span class="ctrl-label">Waktu:</span>
					<div class="btn-group">
						{#each months as m}
							<button
								type="button"
								class="month-btn"
								class:active={selectedMonth === m.key}
								onclick={() => selectedMonth = m.key}
							>
								{m.label}
							</button>
						{/each}
					</div>
				</div>

				<div class="legend-box">
					<span class="legend-title">{legendTitle}</span>
					<div class="legend-items">
						{#each legend as item}
							<span style="background: {item.color}; color: {item.dark ? '#fff' : '#222'}">{item.label}</span>
						{/each}
					</div>
				</div>
			</div>

			<!-- Canvas SVG Peta D3 -->
			<div class="svg-container">
				<svg viewBox="0 0 960 520" preserveAspectRatio="xMidYMid meet">
					<g class="map-stage" style="transform: {mapTransform}; transform-origin: 0 0; transition: transform 1.2s cubic-bezier(0.16, 1, 0.3, 1);">
						<!-- Latar: poligon provinsi (step 0 = choropleth PMI, step 1-3 = abu-abu netral sebagai konteks) -->
						{#if geoData}
							{#each geoData.features as feature}
								{@const provName = feature.properties.Propinsi || feature.properties.name || feature.properties.NAME_1 || ""}
								{@const pData = currentStep === 0 ? getProvRate(provName) : null}
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<path
									d={path(feature)}
									class="prov-path"
									class:no-hit={currentStep >= 1}
									fill={currentStep === 0 ? colorChoroplethPMI(pData.rate) : "#ececec"}
									stroke="#ffffff"
									stroke-width="0.75"
									onmousemove={(e) => {
										if (currentStep !== 0) return;
										activeTooltip = {
											title: provName || "Provinsi",
											info: `Rasio: ${pData.rate} / 10k pddk | Total: ${pData.total.toLocaleString('id-ID')} orang`,
											source: "BPS / BP2MI, 2023"
										};
										tooltipPos = { x: e.clientX, y: e.clientY };
									}}
									onmouseleave={() => activeTooltip = null}
								/>
							{/each}
						{/if}

						<!-- Step 1-3: batas wilayah kab/kota sentra sebagai choropleth -->
						{#if currentStep >= 1}
							{#each sentra as s (s.data.kabupaten)}
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<path
									d={path(s.f)}
									class="kab-poly"
									fill={kabFill(s)}
									onmousemove={(e) => showKabTip(e, s)}
									onmouseleave={() => activeTooltip = null}
								/>
							{/each}
						{/if}

						<!-- Step 2-3: proportional symbol (ukuran = PMI), warna polygon tetap terlihat -->
						{#if currentStep >= 2}
							{#each bubbles as s (s.data.kabupaten)}
								<circle
									cx={s.c[0]}
									cy={s.c[1]}
									r={rScale(kabMetric(s.data)) / zoom.k}
									class="kab-bubble"
								/>
							{/each}
						{/if}
					</g>

					<!-- Lapisan anotasi: koordinat layar, jadi ukuran teks tidak ikut membesar saat zoom -->
					{#if currentStep >= 1}
						<g class="overlay" pointer-events="none">
							{#each topLabels as s (s.data.kabupaten)}
								<text x={sx(s.c[0]) + 8} y={sy(s.c[1]) - 8} class="kab-label">{s.data.kabupaten}</text>
							{/each}

							<!-- Label Khusus Karawang (Pencilan Step 3) -->
							{#if currentStep === 3 && karawang}
								<g transform="translate({sx(karawang.c[0])}, {sy(karawang.c[1])})" class="callout">
									<line x1="0" y1="0" x2="0" y2="-36" stroke="#1f1f1f" stroke-width="0.8" />
									<rect x="-82" y="-62" width="164" height="26" fill="#ffffff" stroke="#d95f43" rx="2" />
									<text x="-74" y="-45" font-size="11" font-family="sans-serif" font-weight="bold" fill="#1f1f1f">Karawang: Rp {fmt(karawang.data.pdrb_perkapita)} Jt</text>
								</g>
							{/if}
						</g>
					{/if}
				</svg>
			</div>
		</div>

		<!-- Tumpukan Kartu Narasi Scrollytelling yang Melayang di Atas Peta -->
		<div class="scrolly-cards-overlay">
			<!-- Step 0 -->
			<div class="scrolly-step-card" data-step="0">
				<span class="card-step-badge"></span>
				<p>
					Terdapat beberapa wilayah yang tercatat mendominasi arus migrasi pekerja migran ke luar negeri pada tahun 2023.
					Berdasarkan data penempatan PMI, <strong class="hl-terracotta">Jawa Timur</strong>, <strong class="hl-terracotta">Jawa Tengah</strong>,
					dan <strong class="hl-terracotta">Jawa Barat</strong> tercatat sebagai tiga provinsi dengan penempatan tertinggi.
					Pulau Jawa tercatat sebagai sentra utama dengan kontribusi mencapai <strong class="hl-mustard">67,7 persen</strong> dari seluruh penempatan nasional.
				</p>
			</div>

			<!-- Step 1 -->
			<div class="scrolly-step-card" data-step="1">
	
				<p>
					Dominasi ini tidak hanya terlihat pada tingkat provinsi, tetapi tergambar semakin jelas ketika ditelusuri hingga tingkat kabupaten/kota.
					Pada tahun 2023, migran internasional terbanyak berasal dari <strong class="hl-terracotta">25 kabupaten/kota tertentu</strong>,
					dan <strong class="hl-terracotta">19 di antaranya berada di Pulau Jawa</strong>. Pola ini menunjukkan arus migrasi tidak tersebar merata,
					melainkan terkonsentrasi di kantong-kantong pengirim dengan karakteristik serupa.
				</p>
			</div>

			<!-- Step 2 -->
			<div class="scrolly-step-card" data-step="2">

				<p>
					Migrasi internasional didorong oleh kombinasi faktor ekonomi dan nonekonomi. Simpson (2017) menggarisbawahi kemiskinan dan tingkat pendapatan
					yang rendah sebagai pendorong kuat dari sisi ekonomi makro. Sejalan dengan hal tersebut, sebagian besar PMI berasal dari wilayah dengan tingkat kemiskinan
					berkisar <strong class="hl-teal">6,5 persen hingga 15,8 persen</strong> (rata-rata sekitar 11 persen).
				</p>
			</div>

			<!-- Step 3 -->
			<div class="scrolly-step-card" data-step="3">
				<span class="card-step-badge"></span>
				<p>
					Dari sisi PDRB per kapita, wilayah-wilayah sentra ini sebagian besar berada pada kisaran Rp 17,7 juta hingga Rp 65,9 juta.
					Namun terdapat satu pengecualian menarik: <strong class="hl-terracotta">Karawang</strong> memiliki PDRB per kapita jauh lebih tinggi
					(<strong class="hl-terracotta">Rp 115,2 juta</strong>) akibat basis industri manufaktur, namun kontribusinya terhadap penempatan PMI justru relatif kecil.
					Temuan ini menunjukkan kemakmuran ekonomi agregat semata belum cukup menjelaskan intensitas migrasi.
				</p>
			</div>
		</div>
	</div>

	<!-- Tooltip Mengambang -->
	{#if activeTooltip}
		<div class="map-tooltip" style="top: {tooltipPos.y + 12}px; left: {tooltipPos.x + 12}px;">
			<div class="tt-title">{activeTooltip.title}</div>
			<div class="tt-info">{activeTooltip.info}</div>
			<div class="tt-src">{activeTooltip.source}</div>
		</div>
	{/if}
</section>

<style>
	/* Update pada .section-title: ubah color jadi var(--color-primary, #78695e) */
	.section-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.2rem, 5vw, 3.4rem);
		color: var(--color-primary, #78695e);
		letter-spacing: 0.05em;
		line-height: 1.15;
		margin: 0 0 0.9rem;
		font-weight: 400;
	}

	/* Tambahkan kelas pembantu wobble & squiggle berikut: */
	.kicker {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		letter-spacing: 0.08em;
		color: #64748b;
		display: inline-block;
		margin-bottom: 0.8rem;
		border-bottom: 1px solid #e2e8f0;
		padding-bottom: 2px;
	}

	.word {
		display: inline-block;
		white-space: nowrap;
		margin: 0 0.2em;
	}

	.letter {
		display: inline-block;
	}


	.map-section-container {
		width: 100%;
		background-color: #ffffff;
		position: relative;
	}

	.section-header {
		text-align: center;
		padding: 4.5rem 1.5rem 2rem;
		max-width: 820px;
		margin: 0 auto;
	}

	.section-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.3rem, 5vw, 3.6rem);
		color: var(--color-primary, #d95f43);
		letter-spacing: 0.05em;
		line-height: 1.1;
		margin: 0 0 0.5rem;
		font-weight: 400;
	}

	/* Pembungkus Scrollytelling Utama */
	.scrolly-wrapper {
		position: relative;
		width: 100%;
	}

	/* Wadah Peta Sticky */
	.sticky-map {
		position: sticky;
		top: 0;
		left: 0;
		width: 100%;
		height: 100vh;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		background: #fafafa;
		overflow: hidden;
		z-index: 1;
	}

	.map-ui-header {
		position: absolute;
		top: 1rem;
		left: 50%;
		transform: translateX(-50%);
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.5rem;
		z-index: 20;
		backdrop-filter: blur(4px);
		padding: 0.5rem 1rem;
		border-radius: 4px;
		border: none;
	}

	.month-selector {
		display: flex;
		align-items: center;
		gap: 0.4rem;
	}

	.ctrl-label {
		font-family: var(--sans, sans-serif);
		font-size: 0.7rem;
		font-weight: 600;
		text-transform: uppercase;
		color: #555;
	}

	.btn-group {
		display: flex;
		gap: 2px;
	}

	.month-btn {
		font-family: var(--mono, monospace);
		font-size: 0.65rem;
		padding: 2px 5px;
		background: #f0f0f0;
		border: 1px solid #ccc;
		border-radius: 2px;
		cursor: pointer;
		transition: all 0.2s;
	}

	.month-btn.active {
		background: #d95f43;
		color: white;
		border-color: #d95f43;
		font-weight: bold;
	}

	.legend-box {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.legend-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.68rem;
		color: #444;
		font-weight: 500;
	}

	.legend-items {
		display: flex;
		gap: 1px;
	}

	.legend-items span {
		font-family: var(--mono, monospace);
		font-size: 0.6rem;
		padding: 2px 4px;
		border-radius: 1px;
	}

	/* Peta dibuat lebih lebar (dulu 960px) supaya kab/kota lebih dekat ke audiens */
	.svg-container {
		width: 100%;
		max-width: 1280px;
		height: 85vh;
		margin-top: 2rem;
	}

	svg {
		width: 100%;
		height: 100%;
	}

	/* vector-effect: garis tetap tipis & tajam walaupun peta di-zoom */
	.prov-path {
		vector-effect: non-scaling-stroke;
		transition: fill 0.4s ease;
	}

	.prov-path.no-hit {
		pointer-events: none;
	}

	.kab-poly {
		stroke: #1f1f1f;
		stroke-width: 0.9;
		stroke-linejoin: round;
		vector-effect: non-scaling-stroke;
		cursor: pointer;
		transition: fill 0.5s ease;
		animation: fade-in 0.6s ease 0.5s both;
	}

	.kab-poly:hover {
		stroke-width: 2.2;
	}

	/* Isi bubble dibuat transparan agar warna choropleth di bawahnya tetap terbaca */
	.kab-bubble {
		fill: rgba(178, 62, 196, 0.1);
		stroke: #ff05c5;
		stroke-width: 2;
		vector-effect: non-scaling-stroke;
		pointer-events: none;
		animation: fade-in 0.5s ease 0.2s both;
	}

	.kab-label {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 10.5px;
		font-weight: 600;
		fill: #1f1f1f;
		paint-order: stroke;
		stroke: #ffffff;
		stroke-width: 3px;
		stroke-linejoin: round;
		animation: fade-in 0.5s ease 0.9s both;
	}

	.callout {
		animation: fade-in 0.5s ease 0.9s both;
	}

	@keyframes fade-in {
		from { opacity: 0; }
		to { opacity: 1; }
	}

	/* Layer Kartu Narasi Scrollytelling Melayang */
	.scrolly-cards-overlay {
		position: relative;
		z-index: 10;
		display: flex;
		flex-direction: column;
		align-items: center;
		/* Padding dan margin tinggi agar pengguna memiliki ruang scroll antar-step */
		padding: 10vh 1.5rem 50vh;
		pointer-events: none;
	}

	.scrolly-step-card {
		pointer-events: auto;
		background: rgba(255, 255, 255, 0.88);
		backdrop-filter: blur(10px);
		-webkit-backdrop-filter: blur(10px);
		border: 1px solid rgba(0, 0, 0, 0.12);
		box-shadow: 0 6px 24px rgba(0, 0, 0, 0.08);
		max-width: 580px;
		padding: 1.6rem 2.2rem;
		border-radius: 4px;
		margin: 65vh 0; /* Memberi jarak scroll lega */
		transition: opacity 0.3s ease, transform 0.3s ease;
	}

	.card-step-badge {
		display: inline-block;
		font-family: var(--mono, monospace);
		font-size: 0.72rem;
		letter-spacing: 0.08em;
		color: #777;
		margin-bottom: 0.6rem;
		border-bottom: 1px solid #ddd;
		padding-bottom: 2px;
	}

	.scrolly-step-card p {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.8;
		color: #2b2b2b;
		margin: 0;
		text-align: justify;
	}

	.hl-terracotta { color: #d95f43; font-weight: 600; }
	.hl-mustard { color: #b87b1e; font-weight: 600; }
	.hl-teal { color: #2b847e; font-weight: 600; }

	/* Tooltip */
	.map-tooltip {
		position: fixed;
		pointer-events: none;
		z-index: 1000;
		background: #1f1f1f;
		color: white;
		padding: 0.5rem 0.8rem;
		border-radius: 3px;
		font-size: 0.75rem;
		font-family: var(--sans, sans-serif);
		box-shadow: 0 2px 10px rgba(0,0,0,0.25);
	}

	.tt-title { font-weight: bold; margin-bottom: 2px; }
	.tt-info { color: #f2ede4; }
	.tt-src { font-size: 0.65rem; color: #aaa; margin-top: 4px; font-style: italic; }

	@media (max-width: 768px) {
		.scrolly-step-card {
			margin: 35vh 0;
			padding: 1.2rem 1.4rem;
		}

		.map-ui-header {
			width: 92%;
			padding: 0.4rem;
		}
	}

	@media (prefers-reduced-motion: reduce) {
		.kab-poly, .kab-bubble, .kab-label, .callout {
			animation: none;
		}
	}
</style>