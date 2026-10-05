<script>
	// ==========================================================
	// SECTION 7: HASIL PEMODELAN (gaya Pudding "Cool Line Up")
	// Statis (tanpa sticky/scrollytelling): grafik + teks mengalir biasa mengikuti scroll.
	// Sumber: BPS, Cerita Data Statistik untuk Indonesia Vol. 2 No. 12 (2025),
	// Susenas Maret 2024 & Sakernas Agustus 2024, diolah dengan Propensity Score Matching.
	// Komponen ini sengaja tanpa dependensi d3 supaya ringan dan aman untuk SSR.
	// ==========================================================

	// ---------- RNG deterministik (agar tampilan "coret tangan" sama di server & browser) ----------
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

	// ---------- Judul dengan huruf "bergoyang" ala Pudding ----------
	const TITLE = "Pulang, Tapi Tidak Selalu Naik Kelas";
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
	const SQUIGGLE_D = "@@SQUIGGLE@@";

	// ---------- Format angka (koma desimal, tidak bergantung ICU server) ----------
	const fmt = (v, d = 3) => Math.abs(v).toFixed(d).replace(".", ",");
	const fmtSigned = (v) => (v > 0 ? "+" : v < 0 ? "\u2212" : "") + fmt(v);
	const fmtTick = (v) => (v === 0 ? "0" : (v < 0 ? "\u2212" : "") + fmt(v, 1));
	const fmtP = (p) => (p < 0.001 ? "< 0,001" : fmt(p, 3));

	// ---------- Data ATT (Gambar 9 & Tabel Hasil Perhitungan ATT, hlm. 17, 26-27) ----------
	// est = estimasi ATT, lo/hi = batas selang kepercayaan 95%, p = p-value
	const facets = {
		total: { label: "Semua purna PMI", color: "#1f2937" },
		gender: { label: "Jenis kelamin", color: "#b8333f" },
		edu: { label: "Pendidikan", color: "#8c5a2b" },
		sector: { label: "Sektor", color: "#245a1e" },
		status: { label: "Kedudukan dalam pekerjaan", color: "#3f6b82" },
		area: { label: "Wilayah tempat tinggal", color: "#7a5c8a" },
		idm: { label: "Status desa asal (IDM)", color: "#a0741a" }
	};

	const rowsByFacet = {
		total: [{ lines: ["Semua purna PMI"], est: 0.051, lo: 0.003, hi: 0.1, p: 0.038 }],
		gender: [
			{ lines: ["Perempuan"], est: 0.153, lo: 0.057, hi: 0.249, p: 0.002 },
			{ lines: ["Laki-laki"], est: 0.008, lo: -0.047, hi: 0.063, p: 0.772 }
		],
		edu: [
			{ lines: ["Tidak/belum tamat SD"], est: 0.115, lo: -0.001, hi: 0.231, p: 0.052 },
			{ lines: ["Pendidikan dasar"], est: 0.016, lo: -0.049, hi: 0.08, p: 0.631 },
			{ lines: ["Pendidikan menengah"], est: 0.045, lo: -0.048, hi: 0.138, p: 0.345 },
			{ lines: ["Pendidikan tinggi"], est: 0.265, lo: 0.035, hi: 0.494, p: 0.024 }
		],
		sector: [
			{ lines: ["Pertanian"], est: -0.108, lo: -0.193, hi: -0.023, p: 0.013 },
			{ lines: ["Industri"], est: 0.086, lo: -0.005, hi: 0.177, p: 0.065 },
			{ lines: ["Jasa"], est: 0.152, lo: 0.081, hi: 0.224, p: 0 }
		],
		status: [
			{ lines: ["Usaha + buruh tidak tetap/", "pekerja keluarga"], est: -0.042, lo: -0.145, hi: 0.061, p: 0.422 },
			{ lines: ["Usaha + buruh tetap"], est: 0.011, lo: -0.238, hi: 0.26, p: 0.932 },
			{ lines: ["Berusaha sendiri"], est: 0.055, lo: -0.04, hi: 0.149, p: 0.256 },
			{ lines: ["Buruh/karyawan/pegawai"], est: 0.167, lo: 0.096, hi: 0.238, p: 0 },
			{ lines: ["Pekerja bebas"], est: -0.062, lo: -0.203, hi: 0.079, p: 0.387 }
		],
		area: [
			{ lines: ["Desa"], est: 0.04, lo: -0.031, hi: 0.112, p: 0.268 },
			{ lines: ["Kota"], est: 0.063, lo: -0.001, hi: 0.126, p: 0.053 }
		],
		idm: [
			{ lines: ["Sangat tertinggal"], est: 0.346, lo: -0.317, hi: 1.01, p: 0.306 },
			{ lines: ["Tertinggal"], est: 0.497, lo: 0.09, hi: 0.904, p: 0.017 },
			{ lines: ["Berkembang"], est: 0.045, lo: -0.06, hi: 0.15, p: 0.404 },
			{ lines: ["Maju"], est: 0.051, lo: -0.021, hi: 0.123, p: 0.166 },
			{ lines: ["Mandiri"], est: 0.037, lo: -0.043, hi: 0.117, p: 0.36 }
		]
	};

	const figures = [
		{
			id: "siapa",
			title: "Selisih pendapatan purna PMI vs non-purna PMI, menurut jenis kelamin dan pendidikan",
			facets: ["total", "gender", "edu"]
		},
		{
			id: "mana",
			title: "Selisih pendapatan purna PMI vs non-purna PMI, menurut sektor dan kedudukan dalam pekerjaan",
			facets: ["sector", "status"]
		},
		{
			id: "asal",
			title: "Selisih pendapatan purna PMI vs non-purna PMI, menurut wilayah dan status desa asal",
			facets: ["area", "idm"]
		}
	];

	// ---------- Geometri grafik ----------
	const W = 668;
	const PLOT = { x0: 232, x1: 604 };
	const DOM = [-0.3, 0.6];
	const ticks = [-0.2, 0, 0.2, 0.4, 0.6];
	const xs = (v) => PLOT.x0 + ((v - DOM[0]) / (DOM[1] - DOM[0])) * (PLOT.x1 - PLOT.x0);

	// Titik "digambar tangan": lingkaran tidak sempurna yang dihaluskan (Catmull-Rom tertutup)
	function blobPath(cx, cy, rad, seed) {
		const rnd = rng(seed);
		const n = 14;
		const pts = [];
		for (let i = 0; i < n; i++) {
			const a = (i / n) * 2 * Math.PI + rnd() * 0.2;
			const rr = rad * (0.92 + rnd() * 0.18);
			pts.push([cx + Math.cos(a) * rr, cy + Math.sin(a) * rr]);
		}
		let d = `M${r2(pts[0][0])},${r2(pts[0][1])}`;
		for (let i = 0; i < n; i++) {
			const p0 = pts[(i - 1 + n) % n];
			const p1 = pts[i];
			const p2 = pts[(i + 1) % n];
			const p3 = pts[(i + 2) % n];
			const c1 = [p1[0] + (p2[0] - p0[0]) / 6, p1[1] + (p2[1] - p0[1]) / 6];
			const c2 = [p2[0] - (p3[0] - p1[0]) / 6, p2[1] - (p3[1] - p1[1]) / 6];
			d += `C${r2(c1[0])},${r2(c1[1])} ${r2(c2[0])},${r2(c2[1])} ${r2(p2[0])},${r2(p2[1])}`;
		}
		return d + "Z";
	}

	// Garis selang kepercayaan: sedikit bergoyang; bentuk "titik-titik" dibuat lewat stroke-dasharray di CSS
	function wobblyLine(xa, xb, y, seed) {
		const rnd = rng(seed);
		const steps = Math.max(2, Math.round((xb - xa) / 14));
		let d = "";
		for (let i = 0; i <= steps; i++) {
			const px = xa + ((xb - xa) * i) / steps;
			const py = y + (rnd() - 0.5) * 1.6;
			d += `${i === 0 ? "M" : "L"}${r2(px)},${r2(py)}`;
		}
		return d;
	}

	function layout(fig) {
		let y = 56;
		const groups = fig.facets.map((fk) => {
			const f = facets[fk];
			let header = null;
			if (fk !== "total") {
				header = { y: y + 9, label: f.label.toUpperCase() };
				y += 20;
			}
			const rows = rowsByFacet[fk].map((r, i) => {
				const h = r.lines.length > 1 ? 38 : 28;
				const cy = y + h / 2;
				y += h;
				const seed = hashStr(`${fk}-${i}-${r.lines[0]}`);
				const sig = r.p < 0.05;
				const clipL = r.lo < DOM[0];
				const clipR = r.hi > DOM[1];
				const xl = xs(Math.max(r.lo, DOM[0]));
				const xh = xs(Math.min(r.hi, DOM[1]));
				return {
					...r,
					cy,
					sig,
					clipL,
					clipR,
					labelY: r.lines.length > 1 ? cy - 2 : cy + 5,
					line: wobblyLine(xl, xh, cy, seed),
					arrL: `M${r2(xl + 6)},${r2(cy - 4.5)} L${r2(xl - 1)},${r2(cy)} L${r2(xl + 6)},${r2(cy + 4.5)}`,
					arrR: `M${r2(xh - 6)},${r2(cy - 4.5)} L${r2(xh + 1)},${r2(cy)} L${r2(xh - 6)},${r2(cy + 4.5)}`,
					dot: blobPath(xs(r.est), cy, fk === "total" ? 9 : 7.6, seed + 1),
					estTxt: fmtSigned(r.est),
					tip:
						`${r.lines.join(" ")}: ${fmtSigned(r.est)} ` +
						`(selang kepercayaan 95%: ${fmtSigned(r.lo)} s.d. ${fmtSigned(r.hi)}; p = ${fmtP(r.p)}; ` +
						`${sig ? "signifikan" : "tidak signifikan"})`
				};
			});
			y += 14;
			return { key: fk, color: f.color, header, rows };
		});
		const yEnd = y - 6;
		return { groups, yEnd, H: yEnd + 34 };
	}

	const charts = figures.map((f) => ({ ...f, ...layout(f) }));
</script>

<section id="section-7" class="s7">
	<!-- JUDUL SECTION -->
	<div class="s7-intro">
		<h2 class="s7-title" aria-label={TITLE}>
			<span class="title-wrap" aria-hidden="true">
				{#each titleWords as word}
					<span class="word">{#each word as l}<span
								class="letter"
								style="transform: translate(0, {l.dy}px) rotate({l.rot}deg);">{l.ch}</span
							>{/each}</span
					>
				{/each}
			</span>
			<svg class="squiggle" viewBox="0 0 142 22" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
				<path d={SQUIGGLE_D} stroke="currentColor" stroke-width="1.5" stroke-linecap="round" />
			</svg>
		</h2>
		<p class="s7-lead">
			Apakah keberadaan PMI membuat rumah tangga lebih sejahtera,
			dan apakah pengalaman kerja di luar negeri menaikkan pendapatan setelah pulang? BPS
			membandingkan mereka dengan rumah tangga dan pekerja yang karakteristiknya serupa.
		</p>
	</div>

	<!-- BAGIAN 1: RUMAH TANGGA -->
	<div class="s7-text">
		<span class="kicker">UANG YANG PULANG</span>
		<p>
			Data Susenas Maret 2024 menunjukkan rumah tangga dengan anggota PMI lebih sejahtera daripada rumah
			tangga serupa tanpa PMI. Pengeluaran per kapitanya signifikan lebih besar, dan arah belanjanya ikut
			bergeser.
		</p>

		<div class="callouts">
			<div class="callout">
				<span class="big">61%</span>
				<p>
					porsi pengeluaran non-makanan yang lebih besar, tanda belanja bergeser ke pendidikan, transportasi,
					dan barang tahan lama.
				</p>
			</div>
			<div class="callout">
				<span class="big">8%</span>
				<p>
					peluang yang lebih besar untuk menempati rumah layak huni, karena remitansi sering dipakai membangun
					atau merenovasi rumah keluarga.
				</p>
			</div>
		</div>

		<p>
			Sejalan dengan teori <em>New Economics of Labor Migration</em>, remitansi bukan sekadar tambahan
			pendapatan. Setelah kebutuhan makan dan minum terpenuhi, uang kiriman itu mengalir ke konsumsi dan
			aset produktif. Temuan serupa juga muncul dalam studi Kencana (2024) di Cilacap, Jawa Tengah.
		</p>
	</div>

	<!-- BAGIAN 2: PENDAPATAN PURNA PMI -->
	<div class="s7-text">
		<span class="kicker">PENDAPATAN PURNA PMI</span>
		<p>
			Cerita berubah setelah mereka pulang. Dengan data Sakernas Agustus 2024, purna PMI dibandingkan dengan
			pekerja non-migran yang berkarakteristik awal serupa. Secara keseluruhan, purna PMI berpendapatan
			<strong>sekitar 5 persen lebih tinggi</strong>. Namun rata-rata menyembunyikan banyak hal: siapa yang
			diuntungkan sangat bergantung pada karakteristiknya.
		</p>
	</div>

	{@render chart(charts[0])}

	<div class="methods">
		<details>
			<summary>Bagaimana membaca grafik ini?</summary>
			<ul>
				<li>
					<strong>Titik besar:</strong> perkiraan selisih pendapatan purna PMI terhadap pekerja non-migran yang
					karakteristiknya serupa (ATT). Angka 0,05 dibaca sekitar 5 persen lebih tinggi; nol berarti tidak ada beda.
				</li>
				<li>
					<strong>Garis titik-titik:</strong> selang kepercayaan 95%. Jika garis menyentuh nol, perbedaan tidak
					terbukti signifikan.
				</li>
				<li>
					<strong>Titik terisi atau kosong:</strong> terisi berarti signifikan (p &lt; 0,05); kosong berarti tidak
					signifikan.
				</li>
				<li>
					<strong>Tanda panah:</strong> garis dipotong di tepi grafik karena melampaui batas tampilan.
				</li>
			</ul>
		</details>
	</div>

	<div class="s7-text">
		<p>
			Menurut jenis kelamin, efek positif yang signifikan hanya terlihat pada <strong>perempuan</strong>; pada
			laki-laki, selisihnya tidak terbukti signifikan. Menurut pendidikan, keuntungan terutama dinikmati
			<strong>lulusan pendidikan tinggi</strong>. Pada kelompok berpendidikan dasar, menengah, maupun yang
			tidak/belum tamat SD, dampaknya tidak signifikan. Artinya, keterampilan dari luar negeri paling efektif
			menaikkan pendapatan bila ditopang modal pendidikan formal yang kuat.
		</p>
	</div>

	{@render chart(charts[1])}

	<div class="s7-text">
		<p>
			Di mana mereka bekerja juga menentukan. Purna PMI di <strong>sektor jasa</strong> memperoleh dampak positif
			yang signifikan, sedangkan di industri tidak ada perbedaan berarti dan di pertanian estimasinya justru
			negatif. Menurut kedudukan dalam pekerjaan, keuntungan signifikan hanya muncul pada
			<strong>buruh, karyawan, atau pegawai</strong>. Pekerja yang berusaha sendiri, dibantu buruh, pekerja
			keluarga, maupun pekerja bebas tidak menunjukkan keuntungan yang signifikan. Temuan ini menunjukkan bahwa
			pasar kerja formal lebih menghargai pengalaman kerja luar negeri dibanding sektor informal atau wirausaha.
		</p>
	</div>

	{@render chart(charts[2])}

	<div class="s7-text">
		<p>
			Antara desa dan kota, tidak ada beda yang signifikan. Dari sisi status desa asal (IDM), dampak positif yang
			signifikan hanya ditemukan pada mereka yang berasal dari <strong>desa tertinggal</strong>. Pada desa sangat
			tertinggal maupun yang berkembang, maju, dan mandiri, dampaknya tidak signifikan. Pengalaman migrasi
			tampak memberi keuntungan relatif lebih besar bagi yang berasal dari wilayah dengan keterbatasan kesempatan
			ekonomi. Garis untuk desa sangat tertinggal sangat lebar karena hanya sekitar 0,01% purna PMI berasal dari
			kelompok itu (Tabel 2).
		</p>
	</div>

	<!-- BAGIAN 3: PENUTUP -->
	<div class="s7-text">
		<span class="kicker">03 / JALAN PULANG</span>
		<p>
			Studi tentang pekerja migran yang kembali (Moreni 2024) mencatat banyak dari mereka justru masuk ke
			sektor informal atau jabatan berketerampilan rendah, sejalan dengan hasil di atas. Karena itu, kebijakan
			migrasi perlu juga memikirkan pemberdayaan ekonomi purna PMI: pelatihan keterampilan, akses pembiayaan, dan
			pendampingan usaha, agar pengalaman di luar negeri menjadi modal produktif di dalam negeri.
		</p>
	</div>

	<div class="methods">
		<details>
			<summary>Bagaimana angka ini dihitung?</summary>
			<ul>
				<li>
					<strong>Data:</strong> Susenas Maret 2024 (kesejahteraan rumah tangga), Sakernas Agustus 2024
					(pendapatan pekerja), dan Indeks Desa Membangun 2024 dari Kemendesa. PMI dan purna PMI di studi ini
					mencakup yang berangkat secara legal maupun tidak.
				</li>
				<li>
					<strong>Pencocokan:</strong> <em>Propensity Score Matching</em> dengan <em>nearest neighbor</em> dan
					<em>caliper</em>. Rumah tangga dicocokkan menurut daerah tempat tinggal, jumlah anggota, jenis kelamin,
					kemampuan baca tulis, kegiatan utama, dan pendidikan kepala rumah tangga. Purna PMI dicocokkan menurut
					jenis kelamin, umur, status perkawinan, pendidikan, status dalam rumah tangga, daerah tempat tinggal,
					dan jumlah anggota rumah tangga.
				</li>
				<li>
					<strong>Catatan kualitas:</strong> setelah pencocokan, umur, status perkawinan, dan status dalam rumah
					tangga masih belum sepenuhnya seimbang. Hasil tetap valid, tetapi perlu dibaca dengan hati-hati.
				</li>
			</ul>
		</details>
	</div>

	<p class="s7-source">
		Sumber: BPS, olahan Susenas Maret 2024 dan
		Sakernas Agustus 2024. Estimasi ATT; titik terisi = signifikan (p &lt; 0,05).
	</p>
</section>

{#snippet chart(c)}
	<figure class="chart-container">
		<div class="chart-title">{c.title}</div>
		<div class="legend">
			<span><i class="filled"></i>signifikan (p &lt; 0,05)</span>
			<span><i class="hollow"></i>tidak signifikan</span>
			<span><i class="dots"></i>selang kepercayaan 95%</span>
		</div>
		<div class="chart-scroll">
			<svg viewBox="0 0 {W} {c.H}" role="img" aria-label={c.title}>
				<!-- Garis kisi putus-putus + sumbu atas & bawah -->
				{#each ticks as t}
					<line class={t === 0 ? "zero" : "grid"} x1={xs(t)} x2={xs(t)} y1="46" y2={c.yEnd} />
					<text class="tick" x={xs(t)} y="38">{fmtTick(t)}</text>
					<text class="tick" x={xs(t)} y={c.yEnd + 20}>{fmtTick(t)}</text>
				{/each}
				<text class="note" x={xs(0) - 10} y="14" text-anchor="end">← lebih rendah</text>
				<text class="note" x={xs(0) + 10} y="14" text-anchor="start">lebih tinggi →</text>

				{#each c.groups as g}
					{#if g.header}
						<text class="facet-h" x="4" y={g.header.y} fill={g.color}>{g.header.label}</text>
						<line
							class="facet-rule"
							x1="4"
							x2={W - 4}
							y1={g.header.y + 6}
							y2={g.header.y + 6}
							stroke={g.color}
						/>
					{/if}

					{#each g.rows as r}
						<g class="row">
							<title>{r.tip}</title>
							{#each r.lines as ln, i}
								<text class="row-label" x={PLOT.x0 - 16} y={r.labelY + i * 14} fill={g.color}>{ln}</text>
							{/each}
							<path class="ci" d={r.line} stroke={g.color} stroke-opacity={r.sig ? 1 : 0.5} />
							{#if r.clipL}
								<path class="arrow" d={r.arrL} stroke={g.color} />
							{/if}
							{#if r.clipR}
								<path class="arrow" d={r.arrR} stroke={g.color} />
							{/if}
							<path
								class="dot"
								d={r.dot}
								fill={r.sig ? g.color : "#ffffff"}
								stroke={g.color}
								stroke-width={r.sig ? 1 : 2.4}
							/>
							<text class="val" class:sig={r.sig} x={W - 2} y={r.cy + 3.5}>{r.estTxt}</text>
						</g>
					{/each}
				{/each}
			</svg>
		</div>
		<figcaption class="sr-only">{c.title}. Setiap baris menampilkan estimasi ATT dan selang kepercayaan 95%.</figcaption>
	</figure>
{/snippet}

<style>
	.s7 {
		width: 100%;
		background: #ffffff;
		position: relative;
		padding-bottom: 4rem;
	}

	/* ---------- Judul (konsisten dengan section lain) ---------- */
	.s7-intro {
		max-width: 680px;
		margin: 0 auto;
		padding: 3.5rem 1.5rem 1.5rem;
		text-align: center;
		background: #ffffff;
	}

	.s7-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.2rem, 5vw, 3.4rem);
		color: var(--color-primary, #78695e);
		letter-spacing: 0.05em;
		line-height: 1.15;
		font-weight: 400;
		margin: 0 0 0.9rem;
        
	}

	.word {
		display: inline-block;
		white-space: nowrap;
		margin: 0 0.2em;
	}

	.letter {
		display: inline-block;
	}

	.squiggle {
		display: block;
		width: min(360px, 70%);
		height: auto;
		margin: 0.3rem auto 0;
		color: var(--color-primary, #78695e);
	}

	.s7-lead {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.8;
		color: #374151;
		margin: 0;
	}

	/* ---------- Teks ---------- */
	.s7-text {
		max-width: 640px;
		margin: 2.2rem auto 0;
		padding: 0 1.5rem;
        text-align: justify;
	}

	.s7-text p {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.02rem;
		line-height: 1.8;
		color: #334155;
		margin: 0 0 1.15rem;
        text-align: justify;
	}

	.s7-text strong {
		color: #0f172a;
	}

	.kicker {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
		letter-spacing: 0.08em;
		color: #64748b;
		display: inline-block;
		margin-bottom: 0.9rem;
		padding-bottom: 2px;
	}

	/* ---------- Angka besar (garis titik-titik di bawahnya) ---------- */
	.callouts {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 1.4rem;
		margin: 1.6rem 0 1.8rem;
	}

	.callout .big {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: 3.4rem;
		line-height: 1;
		color: var(--color-primary, #78695e);
		display: inline-block;
		padding-bottom: 0.45rem;
		background: radial-gradient(circle, currentColor 1.5px, transparent 2px) repeat-x left bottom / 8px 5px;
	}

	.callout p {
		font-size: 0.92rem;
		line-height: 1.6;
		margin: 0.7rem 0 0;
	}

	/* ---------- Grafik ---------- */
	.chart-container {
		max-width: 760px;
		margin: 2.4rem auto 1.2rem;
		padding: 0 1rem;
	}

	.chart-title {
		font-family: var(--mono, ui-monospace, monospace);
		font-weight: 700;
		font-size: 0.82rem;
		line-height: 1.55;
		text-align: center;
		color: #111827;
		max-width: 540px;
		margin: 0 auto 0.5rem;
	}

	.legend {
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		gap: 0.3rem 1.3rem;
		font-family: var(--mono, ui-monospace, monospace);
		font-size: 0.68rem;
		color: #475569;
		margin-bottom: 0.3rem;
	}

	.legend i {
		display: inline-block;
		margin-right: 0.4rem;
		vertical-align: -1px;
	}

	.legend i.filled {
		width: 10px;
		height: 10px;
		border-radius: 50%;
		background: #475569;
	}

	.legend i.hollow {
		width: 10px;
		height: 10px;
		border-radius: 50%;
		border: 2px solid #475569;
		box-sizing: border-box;
	}

	.legend i.dots {
		width: 26px;
		height: 4px;
		vertical-align: 1px;
		background: radial-gradient(circle, #475569 1.4px, transparent 1.8px) repeat-x left center / 6px 4px;
	}

	.chart-scroll {
		overflow-x: auto;
	}

	.chart-scroll svg {
		display: block;
		width: 100%;
		min-width: 560px;
		height: auto;
	}

	.grid {
		stroke: #d9d4c3;
		stroke-width: 1;
		stroke-dasharray: 4 4;
	}

	.zero {
		stroke: #6b6556;
		stroke-width: 1.4;
		stroke-dasharray: 1.5 3.5;
		stroke-linecap: round;
	}

	.tick {
		font-family: var(--mono, ui-monospace, monospace);
		font-weight: 700;
		font-size: 11px;
		fill: #111827;
		text-anchor: middle;
	}

	.note {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: 14px;
		fill: #6b7280;
	}

	.facet-h {
		font-family: var(--mono, ui-monospace, monospace);
		font-weight: 700;
		font-size: 9.5px;
		letter-spacing: 0.08em;
	}

	.facet-rule {
		stroke-width: 1.2;
		stroke-linecap: round;
		stroke-dasharray: 0.1 5;
		opacity: 0.55;
	}

	.row-label {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: 16px;
		text-anchor: end;
	}

	.ci {
		fill: none;
		stroke-width: 3.4;
		stroke-linecap: round;
		stroke-dasharray: 0.1 6.6;
	}

	.arrow {
		fill: none;
		stroke-width: 2;
		stroke-linecap: round;
		stroke-linejoin: round;
	}

	.dot {
		stroke-linejoin: round;
	}

	.val {
		font-family: var(--mono, ui-monospace, monospace);
		font-size: 9.5px;
		fill: #4b5563;
		text-anchor: end;
	}

	.val.sig {
		font-weight: 700;
		fill: #111827;
	}

	.row:hover .dot {
		stroke-width: 3;
	}

	/* ---------- "Details" ala Pudding ---------- */
	.methods {
		max-width: 640px;
		margin: 0.6rem auto 0;
		padding: 0 1.5rem;
	}

	.methods summary {
		font-family: var(--mono, ui-monospace, monospace);
		font-weight: 700;
		font-size: 0.8rem;
		cursor: pointer;
		color: #111827;
	}

	.methods ul {
		margin: 0.8rem 0 0;
		padding-left: 1.1rem;
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 0.95rem;
		line-height: 1.65;
		color: #334155;
	}

	.methods li {
		margin-bottom: 0.5rem;
	}

	.s7-source {
		max-width: 640px;
		margin: 2.2rem auto 0;
		padding: 0 1.5rem;
		font-family: var(--mono, ui-monospace, monospace);
		font-size: 0.68rem;
		line-height: 1.6;
		color: #94a3b8;
	}

	.sr-only {
		position: absolute;
		width: 1px;
		height: 1px;
		margin: -1px;
		padding: 0;
		overflow: hidden;
		clip: rect(0, 0, 0, 0);
		white-space: nowrap;
		border: 0;
	}

	@media (max-width: 560px) {
		.callouts {
			grid-template-columns: 1fr;
		}
	}
</style>