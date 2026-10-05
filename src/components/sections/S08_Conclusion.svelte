<script>
	// ==========================================================
	// SECTION 8: PENUTUP / KESIMPULAN
	// Gaya konsisten dengan Section 7 (Sue Ellen San Francisco + squiggle)
	// Tanpa dependensi d3, ringan, dan aman SSR.
	// ==========================================================

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

	const TITLE = "Dari Pengalaman ke Kebijakan: Jalan Pulang yang Lebih Sejahtera";

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
</script>

<section id="section-8" class="s8">
	<div class="s8-intro">
		<h2 class="s8-title" aria-label={TITLE}>
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
		</h2>
	</div>

	<div class="s8-content">
		<p class="conclusion-paragraph">
			Data menunjukkan bahwa keberhasilan migrasi tenaga kerja tidak berhenti saat kontrak kerja di luar negeri selesai, melainkan ditentukan oleh keberlanjutan ekonomi pekerja setelah kembali ke daerah asal. Remitansi terbukti mampu meningkatkan konsumsi dan kepemilikan aset rumah tangga, namun pengalaman kerja internasional belum otomatis mengangkat pendapatan individu jika pekerja terserap kembali ke sektor informal atau pekerjaan berketerampilan rendah tanpa pengakuan keahlian. Oleh karena itu, kebijakan ketenagakerjaan perlu menyeimbangkan fokus antara perlindungan saat penempatan dengan program reintegrasi yang terukur, termasuk sertifikasi keterampilan, akses permodalan produktif, dan pendampingan usaha berkelanjutan—agar masa kerja di luar negeri benar-benar menjadi bekal mobilitas ekonomi jangka panjang di dalam negeri.
		</p>
	</div>
</section>

<style>
	.s8 {
		width: 100%;
		background: #ffffff;
		position: relative;
		padding: 3rem 1.5rem 6rem;
	
	}

	.s8-intro {
		max-width: 720px;
		margin: 0 auto;
		text-align: center;
	}

	.s8-title {
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

	.s8-content {
		max-width: 640px;
		margin: 2.4rem auto 0;
		padding: 0 0.5rem;
	}

	.conclusion-paragraph {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: 1.05rem;
		line-height: 1.85;
		color: #334155;
		margin: 0;
		text-align: justify;
	}
</style>