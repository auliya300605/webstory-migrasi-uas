<script>
	// ---------- Generator Huruf Bergoyang (Identik S07) ----------
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

	const TITLE = "Mengapa Pekerja Migran Penting Dibahas?";
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

<section id="section-2" class="importance-section">
	<div class="prose-wrapper">
		<!-- Kicker & Judul Bergoyang ala S07 -->
		<div class="section-intro">
			<span class="kicker"></span>
			<h2 class="section-title" aria-label={TITLE}>
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

		<!-- Paragraf 1 -->
		<p class="paragraph">
			Migrasi kerja internasional bukan sekadar perpindahan tenaga kerja, tetapi juga cerminan dinamika
			sosial dan ekonomi Indonesia yang terus berkembang. Selama beberapa dekade terakhir, Pekerja Migran
			Indonesia (PMI) telah menjadi bagian penting dari perekonomian nasional. Pada tahun 2024, tercatat
			sekitar <span class="hl-terracotta">297 ribu penempatan PMI</span> dengan remitansi yang dikirimkan ke tanah air mencapai 
			<span class="hl-mustard">USD 15,7 miliar</span>, setara <span class="hl-mustard">Rp 263,8 triliun</span> yang menjadikan PMI sebagai 
			<span class="hl-teal">sumber devisa terbesar kedua</span> setelah sektor migas.
		</p>

		<!-- Paragraf 2 -->
		<p class="paragraph">
			Namun, di balik kontribusinya yang besar tersebut, fenomena PMI menyimpan pertanyaan yang lebih
			mendasar: wilayah seperti apa yang mendorong seseorang memutuskan untuk bermigrasi, dan
			bagaimana keputusan tersebut pada akhirnya memengaruhi kesejahteraan keluarga yang ditinggalkan.
		</p>

		<!-- Paragraf 3 -->
		<p class="paragraph">
			Berdasarkan uraian di atas, <em>web story</em> ini berupaya untuk mengulas perjalanan PMI secara menyeluruh,
			mulai dari <span class="hl-terracotta">karakteristik wilayah asal</span>, arah migrasi ke negara tujuan, hingga dampaknya terhadap
			<span class="hl-mustard">kesejahteraan keluarga</span> dan tantangan <span class="hl-teal">reintegrasi</span> setelah kembali ke tanah air.
		</p>
	</div>
</section>

<style>
	.importance-section {
		width: 100%;
		background-color: var(--color-bg-clean, #ffffff);
		padding: 5rem 1.5rem 6rem;
		box-sizing: border-box;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.prose-wrapper {
		width: 100%;
		max-width: 680px;
		margin: 0 auto;
	}

	.section-intro {
		text-align: center;
		margin-bottom: 2.4rem;
	}

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

	/* Judul Section Bergoyang ala S07 */
	.section-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive;
		font-size: clamp(2.2rem, 5vw, 3.4rem);
		color: var(--color-primary, #78695e);
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


	.paragraph {
		font-family: "Jost", var(--sans), sans-serif;
		font-size: clamp(1.05rem, 1.2vw, 1.15rem);
		line-height: 1.8;
		color: #333333;
		text-align: justify;
		margin: 0 0 1.6rem;
		letter-spacing: 0.01em;
		hyphens: auto;
	}

	.paragraph:last-child {
		margin-bottom: 0;
	}

	.hl-terracotta {
		color: #1f1f1f;
		font-weight: 600;
		border-bottom: 2px solid #d95f43;
		padding-bottom: 1px;
	}

	.hl-mustard {
		color: #1f1f1f;
		font-weight: 600;
		border-bottom: 2px solid #e79a3c;
		padding-bottom: 1px;
	}

	.hl-teal {
		color: #1f1f1f;
		font-weight: 600;
		border-bottom: 2px solid #4ea59f;
		padding-bottom: 1px;
	}

	@media (max-width: 600px) {
		.importance-section {
			padding: 4rem 1.2rem 4.5rem;
		}

		.paragraph {
			text-align: left;
			line-height: 1.7;
		}
	}
</style>