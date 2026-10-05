<script>
	import { onMount } from "svelte";

	let { activeSection = 0 } = $props();

	const totalSections = 8;
	const sectionNames = [
		"Judul",
		"Urgensi",
		"Wilayah Asal",
		"Konsistensi",
		"Profil PMI",
		"Negara Tujuan",
		"Dampak Ekonomi",
		"Kesimpulan"
	];

	const scrollToSection = (index) => {
		const targetId = `section-${index + 1}`;
		const el = document.getElementById(targetId);
		if (el) {
			el.scrollIntoView({ behavior: "smooth" });
		}
	};
</script>

<nav class="progress-nav" aria-label="Navigasi Cerita">
	{#each Array(totalSections) as _, i}
		<button
			type="button"
			class="pill-btn"
			class:active={activeSection === i}
			onclick={() => scrollToSection(i)}
			title={`Lompat ke Bab ${i + 1}: ${sectionNames[i]}`}
		>
			<span class="pill-number">{i + 1}</span>
		</button>
	{/each}
</nav>

<style>
	.progress-nav {
		position: fixed;
		top: 50%;
		left: 1.5rem;
		transform: translateY(-50%);
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		z-index: 100;
	}

	.pill-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 32px;
		height: 24px;
		background-color: var(--color-gray-200, #e5e5e5);
		border: 1px solid var(--color-gray-300, #d4d4d4);
		border-radius: 3px;
		cursor: pointer;
		padding: 0;
		transition: all 0.2s ease-in-out;
	}

	.pill-number {
		font-family: var(--mono, monospace);
		font-size: 0.75rem;
		color: var(--color-gray-700, #404040);
		font-weight: 500;
	}

	.pill-btn:hover {
		background-color: var(--color-gray-400, #a8a8a8);
	}

	.pill-btn.active {
		background-color: var(--color-accent-teal, #4ea59f);
		border-color: var(--color-black, #282828);
		transform: scale(1.1);
	}

	.pill-btn.active .pill-number {
		color: #ffffff;
		font-weight: 700;
	}

	@media (max-width: 768px) {
		.progress-nav {
			display: none; 
		}
	}
</style>