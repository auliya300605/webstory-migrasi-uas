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

	const TITLE = "Meniti Asa di Negeri Orang";
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

<section id="section-1" class="hero-container">
	<div class="hero-content">
		<!-- Area Visual Ilustrasi -->
		<div class="visual-wrapper">
			<img
				src="/assets/hero.png"
				alt="Ilustrasi Perjalanan Pekerja Migran Indonesia Meniti Asa"
				class="hero-illustration"
			/>
		</div>

		<!-- Judul Bergoyang ala S07 -->
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
			<svg class="squiggle" viewBox="0 0 142 22" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
				<path d={SQUIGGLE_D} stroke="currentColor" stroke-width="1.5" stroke-linecap="round" />
			</svg>
		</h1>

		<!-- Subjudul dengan Highlight Warna -->
		<p class="subtitle">
			PERJALANAN <span class="hl-terracotta">PEKERJA MIGRAN INDONESIA</span> MENINGKATKAN 
			<span class="hl-mustard">KESEJAHTERAAN KELUARGA</span> DAN TANTANGAN 
			<span class="hl-teal">REINTEGRASI PASCA MIGRASI</span>
		</p>

		<!-- Byline -->
		<div class="byline">
			By <strong>Auliya</strong>
		</div>
	</div>

	<!-- Scroll indicator -->
	<div class="scroll-hint">
		<span class="scroll-arrow">↓</span>
		<span>Gulir ke bawah</span>
	</div>
</section>

<style>
	.hero-container {
		width: 100%;
		padding: 0.5rem 1.5rem 3rem;
		box-sizing: border-box;
		background-color: var(--color-bg-clean, #ffffff);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: flex-start;
		position: relative;
		text-align: center;
	}

	.hero-content {
		max-width: 820px;
		display: flex;
		flex-direction: column;
		align-items: center;
		margin-top: 0;
	}

	.visual-wrapper {
		width: 100%;
		max-width: 580px;
		margin-bottom: 1.25rem;
	}

	.hero-illustration {
		width: 100%;
		height: auto;
		object-fit: contain;
		display: block;
		border: none;
		box-shadow: none;
	}

	/* Judul Tulisan Tangan Sue Ellen Bergoyang */
	.main-title {
		font-family: "Sue Ellen San Francisco", var(--handwriting), cursive !important;
		font-size: clamp(2.8rem, 6vw, 4.4rem);
		color: var(--color-primary, #78695e);
		letter-spacing: 0.05em;
		line-height: 1.15;
		margin: 0.5rem 0 1.2rem;
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

	.squiggle {
		display: block;
		width: min(340px, 65%);
		height: auto;
		margin: 0.4rem auto 0;
		color: var(--color-primary, #78695e);
	}

	/* Subjudul Sans-serif Editorial */
	.subtitle {
		font-family: var(--sans);
		text-transform: uppercase;
		font-size: clamp(0.85rem, 1.8vw, 1.02rem);
		line-height: 1.6;
		color: var(--color-gray-700, #555555);
		letter-spacing: 0.06em;
		max-width: 720px;
		margin: 0 0 1.25rem;
		padding: 0 1rem;
	}

	.hl-terracotta {
		color: #d95f43;
		font-weight: 700;
		background-color: rgba(217, 95, 67, 0.08);
		padding: 0.1rem 0.35rem;
		border-radius: 2px;
	}

	.hl-mustard {
		color: #b87b1e;
		font-weight: 700;
		background-color: rgba(231, 154, 60, 0.12);
		padding: 0.1rem 0.35rem;
		border-radius: 2px;
	}

	.hl-teal {
		color: #2b847e;
		font-weight: 700;
		background-color: rgba(78, 165, 159, 0.12);
		padding: 0.1rem 0.35rem;
		border-radius: 2px;
	}

	.byline {
		font-family: var(--sans, sans-serif);
		font-size: 0.85rem;
		color: #666666;
		letter-spacing: 0.03em;
	}

	.byline strong {
		color: #222222;
		font-weight: 600;
	}

	.scroll-hint {
		margin-top: 2rem;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.2rem;
		color: #888888;
		font-family: var(--mono, monospace);
		font-size: 0.7rem;
		letter-spacing: 0.04em;
	}

	.scroll-arrow {
		animation: bounce 1.8s infinite;
	}

	@keyframes bounce {
		0%, 20%, 50%, 80%, 100% {
			transform: translateY(0);
		}
		40% {
			transform: translateY(5px);
		}
		60% {
			transform: translateY(3px);
		}
	}
</style>