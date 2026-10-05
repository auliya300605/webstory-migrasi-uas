<script>
	import { onMount } from "svelte";
	import Footer from "$components/Footer.svelte";
	import Progress from "$components/ui/Progress.svelte";

	// Import 8 Section
	import S01Hero from "$components/sections/S01_Hero.svelte";
	import S02Importance from "$components/sections/S02_Importance.svelte";
	import S03OriginMap from "$components/sections/S03_OriginMap.svelte";
	import S04Consistency from "$components/sections/S04_Consistency.svelte";
	import S05Demographics from "$components/sections/S05_Demographics.svelte";
	import S06DestinationFlow from "$components/sections/S06_DestinationFlow.svelte";
	import S07EconomicReturn from "$components/sections/S07_EconomicReturn.svelte";
	import S08Conclusion from "$components/sections/S08_Conclusion.svelte";

	let activeSection = $state(0);

	onMount(() => {
		const observer = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					if (entry.isIntersecting) {
						const id = entry.target.id;
						const index = parseInt(id.replace("section-", "")) - 1;
						if (!isNaN(index)) {
							activeSection = index;
						}
					}
				});
			},
			{ threshold: 0.4 } // Berubah warna ketika 40% section masuk layar
		);

		for (let i = 1; i <= 8; i++) {
			const el = document.getElementById(`section-${i}`);
			if (el) observer.observe(el);
		}

		return () => observer.disconnect();
	});
</script>

<svelte:boundary onerror={(e) => console.error("Error rendering story:", e)}>
	<!-- Navigasi Tombol Angka Melayang di Kiri -->
	<Progress {activeSection} />

	<article class="story-container">
		<S01Hero />
		<div id="section-2"><S02Importance /></div>
		<div id="section-3"><S03OriginMap /></div>
		<div id="section-4"><S04Consistency /></div>
		<div id="section-5"><S05Demographics /></div>
		<div id="section-6"><S06DestinationFlow /></div>
		<div id="section-7"><S07EconomicReturn /></div>
		<div id="section-8"><S08Conclusion /></div>
	</article>

	<Footer recirc={false} />
</svelte:boundary>

<style>
	.story-container {
		width: 100%;
		min-height: 100vh;
		background-color: var(--color-bg-clean, #ffffff);
		color: var(--color-black, #282828);
		overflow-x: clip;
	}
</style>