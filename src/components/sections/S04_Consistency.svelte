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

	const TITLE_S04 = "Apakah Pola Ini Konsisten di Semua Provinsi?";
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

	const SQUIGGLE_D =
		"M3 13.5C22.5 7.5 45.2 6.8 65.5 12.2C85.8 17.6 106.1 18.2 125.8 11.8C131.2 10.1 136.5 7.8 141 5";
	import * as d3 from "d3";
	import Scrolly from "$components/helpers/Scrolly.svelte";
	import pcaData from "$data/s04_data.json";

	// Komponen Pendukung
	import PcaBiplot from "./PcaBiplot.svelte";
	import ParallelCoordinates from "./ParallelCoordinates.svelte";
	import ClusteredHeatmap from "./ClusteredHeatmap.svelte";

	// Indeks Scrollytelling (0, 1, 2, 3, 4)
	let activeStep = $state(0);

	// Shared State Linking & Brushing dua arah
	let selectedIds = $state([]);
	let hoveredId = $state(null);

	let activeTooltip = $state(null);
	let tooltipPos = $state({ x: 0, y: 0 });

	const width = 900;
	const height = 560;
	const margin = { top: 40, right: 60, bottom: 60, left: 60 };

	const xScale = d3.scaleLinear().domain([-6, 8]).range([margin.left, width - margin.right]);
	const yScale = d3.scaleLinear().domain([-5, 3.5]).range([height - margin.bottom, margin.top]);
	const rScale = d3.scaleSqrt().domain([0, 70000]).range([4.5, 26]);
	const vectorMultiplier = 7.5;

	function getPointStyle(p, step) {
		const isStep0 = step === 0 || step === undefined;
		const isStep1 = step === 1;
		const isStep2 = step === 2;
		const isStep3 = step === 3;

		if (isStep0) {
			return { fill: "#9ca3af", stroke: "#4b5563", opacity: 0.65, r: rScale(p.pmi_count), highlight: false };
		}
		if (isStep1) {
			if (p.group === "makmur") {
				return { fill: "#0d9488", stroke: "#115e59", opacity: 1, r: rScale(p.pmi_count) * 1.25, highlight: true };
			}
			return { fill: "#d1d5db", stroke: "#9ca3af", opacity: 0.18, r: rScale(p.pmi_count) * 0.8, highlight: false };
		}
		if (isStep2) {
			if (p.group === "sentra") {
				return { fill: "#e11d48", stroke: "#9f1239", opacity: 1, r: rScale(p.pmi_count) * 1.35, highlight: true };
			}
			return { fill: "#d1d5db", stroke: "#9ca3af", opacity: 0.15, r: rScale(p.pmi_count) * 0.75, highlight: false };
		}
		if (isStep3) {
			if (p.group === "tertinggal") {
				return { fill: "#64748b", stroke: "#334155", opacity: 0.95, r: Math.max(rScale(p.pmi_count), 6), highlight: true };
			}
			return { fill: "#d1d5db", stroke: "#9ca3af", opacity: 0.15, r: rScale(p.pmi_count) * 0.75, highlight: false };
		}
		return { fill: "#d1d5db", stroke: "#9ca3af", opacity: 0.5, r: rScale(p.pmi_count), highlight: false };
	}
</script>

<section id="section-4" class="pca-section-container">
	<!-- 1. PENGANTAR NON-SCROLLING (FORMAT KONTEMPLASI MIRIP SECTION 2) -->
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

		<p class="paragraph">
			Kemiskinan dan PDRB per kapita memberikan gambaran awal mengenai karakteristik wilayah asal PMI,
			tetapi kedua indikator tersebut belum cukup untuk menjelaskan fenomena migrasi secara utuh.
			Kerangka yang disusun <em>International Organization for Migration</em> (IOM, 2015) menyebutkan bahwa
			migrasi internasional juga dipengaruhi oleh kondisi lapangan pekerjaan, tingkat pendidikan, struktur
			sektor ekonomi, infrastruktur, serta ketimpangan di wilayah asal.
		</p>

		<p class="paragraph">
			Dengan demikian, dilakukan reduksi dimensi menggunakan <strong>Principal Component Analysis (PCA)</strong>
			terhadap sembilan indikator sosial-ekonomi tingkat provinsi: PDRB per kapita, persentase penduduk miskin,
			Indeks Pembangunan Manusia (IPM), tingkat pengangguran terbuka (TPT), persentase pekerja sektor pertanian,
			rata-rata lama sekolah (RLS), persentase pekerja informal, upah minimum provinsi (UMP), dan rasio ketergantungan penduduk.
		</p>

		<div class="pca-dimensions-box">
			<h4 class="dim-box-title">Tiga Dimensi Utama Pembangunan Daerah (Kumulatif 80.1% Varians):</h4>
			<ol class="dim-list">
				<li><strong>PC1 (53,23% varians):</strong> Wajah kesejahteraan ekonomi modern (PDRB, IPM, UMP) versus kerentanan informal dan sektor pertanian.</li>
				<li><strong>PC2 (16,21% varians):</strong> Tekanan dinamika pasar kerja lokal dan tingkat pengangguran terbuka.</li>
				<li><strong>PC3 (10,66% varians):</strong> Struktur demografi dan rasio beban ketergantungan penduduk.</li>
			</ol>
		</div>
	</div>

	<!-- 2. SATU KESATUAN STICKY VIEWPORT & SCROLLYTELLING TERPADU -->
	<div class="unified-scrolly-wrapper">
		<!-- Wadah Sticky di Latar Belakang Layar Penuh -->
		<div class="sticky-stage">
			<!-- TAMPILAN A (STEP 0 - 3): BIPLOT PENUH DI TENGAH LAYAR -->
			<div class="guided-biplot-view" class:hidden-mode={activeStep >= 4}>
				<div class="chart-header-tag">
					<span class="tag-title">BIPLOT PCA (PC1 vs PC2) & JUMLAH PENEMPATAN PMI</span>
					<span class="tag-desc">Sumbu X: PC1 (53.2%) | Sumbu Y: PC2 (16.2%) | Ukuran Lingkaran: Volume PMI Resmi</span>
				</div>

				<div class="svg-stage">
					<svg viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
						<defs>
							<marker id="arrow" viewBox="0 0 10 10" refX="6" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
								<path d="M 0 1.5 L 8 5 L 0 8.5 z" fill="#6b7280" />
							</marker>
						</defs>

						<line x1={xScale(-6)} y1={yScale(0)} x2={xScale(8)} y2={yScale(0)} stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 4" />
						<line x1={xScale(0)} y1={yScale(-5)} x2={xScale(0)} y2={yScale(3.5)} stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 4" />

						<text x={xScale(7.8)} y={yScale(0) - 8} text-anchor="end" class="axis-label-guide">→ Modern, IPM & Upah Tinggi (PC1+)</text>
						<text x={xScale(-5.8)} y={yScale(0) - 8} text-anchor="start" class="axis-label-guide">← Sektor Pertanian & Kemiskinan (PC1-)</text>
						<text x={xScale(0) + 8} y={yScale(3.3)} text-anchor="start" class="axis-label-guide">↑ Pengangguran & Tekanan Kerja (PC2+)</text>

						<g class="loading-vectors" opacity={activeStep === 0 ? 0.85 : 0.25} style="transition: opacity 0.5s ease;">
							{#each pcaData.loadings as vec}
								{@const vx = xScale(vec.loading_pc1 * vectorMultiplier)}
								{@const vy = yScale(vec.loading_pc2 * vectorMultiplier)}
								<line x1={xScale(0)} y1={yScale(0)} x2={vx} y2={vy} stroke="#4b5563" stroke-width="1.2" marker-end="url(#arrow)" />
								<text x={vx + (vec.loading_pc1 >= 0 ? 5 : -5)} y={vy + (vec.loading_pc2 >= 0 ? -4 : 10)} text-anchor={vec.loading_pc1 >= 0 ? "start" : "end"} class="vector-label">
									{vec.name}
								</text>
							{/each}
						</g>

						<g class="provinces-layer">
							{#each pcaData.provinces as p}
								{@const style = getPointStyle(p, activeStep)}
								{@const cx = xScale(p.pc1)}
								{@const cy = yScale(p.pc2)}
								<!-- svelte-ignore a11y_no_static_element_interactions -->
								<circle
									{cx}
									{cy}
									r={style.r}
									fill={style.fill}
									stroke={style.stroke}
									stroke-width={style.highlight ? 2 : 0.8}
									opacity={style.opacity}
									class="prov-bubble"
									class:glow={style.highlight}
									style="transition: all 0.6s cubic-bezier(0.16, 1, 0.3, 1);"
									onmousemove={(e) => {
										activeTooltip = { title: p.provinsi, pmi: p.pmi_count.toLocaleString("id-ID"), pc1: p.pc1, pc2: p.pc2 };
										tooltipPos = { x: e.clientX, y: e.clientY };
									}}
									onmouseleave={() => (activeTooltip = null)}
								/>

								{#if style.highlight || (activeStep === 0 && ["DKI Jakarta", "Jawa Timur", "Jawa Barat", "Papua"].includes(p.provinsi))}
									<text x={cx} y={cy - style.r - 4} text-anchor="middle" class="bubble-label" class:highlight-label={style.highlight}>
										{p.provinsi}
									</text>
								{/if}
							{/each}
						</g>
					</svg>
				</div>

				<div class="chart-attribution">
					Sumber data: BPS & BP2MI, 2024 | Analisis Reduksi Dimensi: PCA 9 Indikator Sosio-Ekonomi
				</div>
			</div>

			<!-- TAMPILAN B (STEP 4): DASHBOARD MULTIVARIAT TERPADU (BIPLOT KIRI ATAS, PCP KANAN ATAS, HEATMAP FULL-WIDTH BAWAH) -->
			<div class="free-exploration-dashboard" class:visible-mode={activeStep >= 4}>
				<div class="dashboard-top-row">
                    <div class="top-col-left">
                        <PcaBiplot data={pcaData} bind:selectedIds bind:hoveredId width={560} height={320} />
                    </div>
                    <div class="top-col-right">
                        <ParallelCoordinates data={pcaData} bind:selectedIds bind:hoveredId width={560} height={320} />
                    </div>
                </div>

				<div class="dashboard-bottom-row">
					<ClusteredHeatmap data={pcaData} bind:selectedIds bind:hoveredId />
				</div>
			</div>
		</div>

		<!-- KARTU NARASI SCROLLING YANG MELAYANG -->
		<div class="scrolly-narrative-overlay">
			<Scrolly bind:value={activeStep} top={120} bottom={120}>
				<!-- Step 0 -->
				<div class="step-card" class:active={activeStep === 0}>
					<p>
						Ketika kita memetakan posisi seluruh provinsi dan memadukannya dengan ukuran lingkaran yang
						merepresentasikan jumlah penempatan PMI, sebuah pola menarik mulai terlihat.
						Migrasi masif ternyata <strong>tidak terjadi di wilayah yang paling miskin ekstrem</strong>,
						ataupun di wilayah yang paling kaya raya.
					</p>
				</div>

				<!-- Step 1 -->
				<div class="step-card" class:active={activeStep === 1}>
					<p>
						Perhatikan kuadran kanan atas: <strong class="hl-teal">DKI Jakarta</strong> (1.496 PMI),
						<strong class="hl-teal">Kalimantan Timur</strong> (137 PMI), <strong class="hl-teal">Kepulauan Riau</strong> (904 PMI),
						dan <strong class="hl-teal">Banten</strong> (3.183 PMI) mencatat skor PC1 yang sangat tinggi.
						Wilayah ber-PDRB dan IPM tinggi ini memiliki angka penempatan kecil karena pasar kerja formal lokal
						sudah sangat menarik dengan upah yang kompetitif.
					</p>
				</div>

				<!-- Step 2 -->
				<div class="step-card" class:active={activeStep === 2}>

					<p>
						Sebaliknya, <strong class="hl-coral">Jawa Timur</strong> (68.069 PMI),
						<strong class="hl-coral">Jawa Tengah</strong> (59.009 PMI), <strong class="hl-coral">Jawa Barat</strong> (52.961 PMI),
						hingga <strong class="hl-coral">Nusa Tenggara Barat</strong> (33.949 PMI).
						Tekanan demografi usia kerja yang padat berpadu dengan ketatnya pasar kerja lokal memaksa tenaga kerjanya
						mencari penghidupan ke luar negeri.
					</p>
				</div>

				<!-- Step 3 -->
				<div class="step-card" class:active={activeStep === 3}>
					<p>
						Di ujung kiri, wilayah dengan tantangan terberat seperti <strong class="hl-slate">Papua</strong> (hanya 47 PMI),
						<strong class="hl-slate">Papua Barat</strong> (15 PMI), dan <strong class="hl-slate">Sulawesi Barat</strong> (221 PMI)
						mencatatkan skor PC1 sangat negatif. Namun penempatan formalnya justru kecil.
						Ini membuktikan <em>migrasi internasional butuh modal</em>: hambatan biaya awal, informasi, dan jalur resmi
						membuat daerah tertinggal tidak otomatis menjadi kantong PMI.
					</p>
				</div>

				<!-- Step 4: KOTAK NARASI EKSPLORASI MANDIRI -->
				<div class="step-card" class:active={activeStep === 4}>
                    <p>
                        Sekarang Anda dapat menjelajahi data secara mandiri.
                        Gunakan <em>brush</em> pada sumbu vertikal untuk menyaring kriteria, atau klik provinsi pada Biplot maupun Heatmap untuk meneliti profilnya secara mendalam.
                    </p>
                </div>

				<!-- Spacer ekstra di bawah agar pembaca bisa leluasa menjelajah dashboard setelah kartu narasi lewat -->
				<div class="exploration-free-space"></div>
			</Scrolly>
		</div>
	</div>

	<!-- Tooltip Mengambang -->
	{#if activeTooltip}
		<div class="chart-tooltip" style="top: {tooltipPos.y + 12}px; left: {tooltipPos.x + 12}px;">
			<div class="tt-header">{activeTooltip.title}</div>
			<div class="tt-row">Penempatan PMI: <strong>{activeTooltip.pmi} orang</strong></div>
			<div class="tt-sub">Skor PC1: {activeTooltip.pc1.toFixed(2)} | Skor PC2: {activeTooltip.pc2.toFixed(2)}</div>
		</div>
	{/if}
</section>

<style>
	/* Update warna .section-title agar selaras dengan palette S07 */
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

	.pca-section-container {
		width: 100%;
		background-color: #ffffff;
		position: relative;
	}

	/* 1. Header Pengantar Non-Scrolling */
	.intro-prose-container {
		width: 100%;
		max-width: 680px;
		margin: 0 auto;
		padding: 5.5rem 1.5rem 2.5rem;
		box-sizing: border-box;
	}

	.section-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.3rem, 5vw, 3.5rem);
		color: var(--color-primary, #d95f43);
		text-align: center;
		letter-spacing: 0.05em;
		line-height: 1.15;
		margin: 0 0 2.2rem;
		font-weight: 400;
	}

	.paragraph {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: clamp(1.05rem, 1.2vw, 1.15rem);
		line-height: 1.85;
		color: #2b2b2b;
		text-align: justify;
		margin: 0 0 1.5rem;
	}

	.pca-dimensions-box {
		background: none;
		padding: 1.2rem 1.5rem;
		margin: 2rem 0 1rem;
	}

	.dim-box-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.95rem;
		color: #1f1f1f;
		margin: 0 0 0.75rem;
		font-weight: 600;
	}

	.dim-list {
		margin: 0;
		padding-left: 1.25rem;
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 0.92rem;
		line-height: 1.7;
		color: #4b5563;
	}

	.dim-list li {
		margin-bottom: 0.35rem;
	}

	/* 2. Unified Sticky Scrollytelling Setup */
	.unified-scrolly-wrapper {
		position: relative;
		width: 100%;
	}

	.sticky-stage {
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

	/* TAMPILAN A: Biplot Terpusat (Step 0 - 3) */
	.guided-biplot-view {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		transition: opacity 0.8s cubic-bezier(0.16, 1, 0.3, 1), transform 0.8s cubic-bezier(0.16, 1, 0.3, 1);
		opacity: 1;
		transform: scale(1);
	}

	.guided-biplot-view.hidden-mode {
		opacity: 0;
		transform: scale(0.92);
		pointer-events: none;
	}

	.chart-header-tag {
		position: absolute;
		top: 1.2rem;
		left: 50%;
		transform: translateX(-50%);
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.2rem;
		z-index: 10;
		
		backdrop-filter: blur(4px);
		padding: 0.45rem 1rem;
	
	}

	.tag-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.75rem;
		font-weight: 700;
		color: #1f1f1f;
		letter-spacing: 0.05em;
	}

	.tag-desc {
		font-family: var(--mono, monospace);
		font-size: 0.65rem;
		color: #6b7280;
	}

	.svg-stage {
		width: 100%;
		max-width: 960px;
		height: 80vh;
		margin-top: 1.5rem;
	}

	svg {
		width: 100%;
		height: 100%;
	}

	.axis-label-guide {
		font-family: var(--sans, sans-serif);
		font-size: 10px;
		fill: #9ca3af;
		font-weight: 500;
		letter-spacing: 0.03em;
	}

	.vector-label {
		font-family: var(--sans, sans-serif);
		font-size: 9.5px;
		fill: #374151;
		font-weight: 600;
	}

	.prov-bubble {
		cursor: pointer;
	}

	.prov-bubble.glow {
		filter: drop-shadow(0px 0px 6px rgba(0, 0, 0, 0.3));
	}

	.bubble-label {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 10px;
		font-weight: 600;
		fill: #1f1f1f;
		paint-order: stroke;
		stroke: #ffffff;
		stroke-width: 2.5px;
		pointer-events: none;
	}

	.highlight-label {
		font-size: 11px;
		font-weight: 700;
	}

	.chart-attribution {
		position: absolute;
		bottom: 1rem;
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		color: #9ca3af;
	}

	/* TAMPILAN B: Dashboard Transisi (Step 4) */
	.free-exploration-dashboard {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		max-width: 1260px;
		left: 50%;
		transform: translateX(-50%) translateY(20px);
		box-sizing: border-box;
		padding: 1.2rem 1.5rem 1rem;
		display: flex;
		flex-direction: column;
		justify-content: center;
		gap: 0.9rem;
		opacity: 0;
		pointer-events: none;
		transition: opacity 0.8s cubic-bezier(0.16, 1, 0.3, 1), transform 0.8s cubic-bezier(0.16, 1, 0.3, 1);
		z-index: 5;
	}

	.free-exploration-dashboard.visible-mode {
		opacity: 1;
		transform: translateX(-50%) translateY(0);
		pointer-events: auto;
	}

	.dashboard-top-row {
		display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 2.5rem;
        align-items: start;
	}

	.dashboard-bottom-row {
		width: 100%;
	}

	/* Layer Kartu Narasi Melayang */
	.scrolly-narrative-overlay {
		position: relative;
		z-index: 10;
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 20vh 1.5rem 10vh;
		pointer-events: none;
	}

	.step-card {
		pointer-events: auto;
		background: rgba(255, 255, 255, 0.9);
		backdrop-filter: blur(10px);
		-webkit-backdrop-filter: blur(10px);
		border: 1px solid rgba(229, 231, 235, 0.9);
		box-shadow: 0 8px 30px rgba(0, 0, 0, 0.06);
		max-width: 480px;
		padding: 1.6rem 2rem;
		border-radius: 6px;
		margin: 60vh 0;
		transition: transform 0.3s ease, border-color 0.3s ease;
	}

	.step-card.active {
		border-color: #d1d5db;
		transform: scale(1.02);
	}

	

	.step-card p {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.8;
		color: #2b2b2b;
		margin: 0;
		text-align: justify;
	}

	/* Ruang scroll ekstra agar pengguna bisa bebas berinteraksi di Step 4 */
	.exploration-free-space {
		height: 90vh;
	}

	.hl-teal { color: #0d9488; font-weight: 600; }
	.hl-coral { color: #e11d48; font-weight: 600; }
	.hl-slate { color: #475569; font-weight: 600; }

	/* Tooltip */
	.chart-tooltip {
		position: fixed;
		pointer-events: none;
		z-index: 1000;
		background: #1f2937;
		color: #f9fafb;
		padding: 0.5rem 0.8rem;
		border-radius: 4px;
		font-family: var(--sans, sans-serif);
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
	}

	.tt-header { font-weight: 700; font-size: 0.82rem; margin-bottom: 2px; }
	.tt-row { font-size: 0.75rem; color: #f3f4f6; }
	.tt-sub { font-size: 0.68rem; color: #9ca3af; margin-top: 3px; font-family: var(--mono); }

	@media (max-width: 992px) {
		.dashboard-top-row {
			grid-template-columns: 1fr;
		}
	}
</style>
