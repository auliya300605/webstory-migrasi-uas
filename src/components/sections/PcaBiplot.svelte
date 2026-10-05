<script>
	import * as d3 from "d3";

	let {
		data,
		selectedIds = $bindable([]),
		hoveredId = $bindable(null),
		width = 460,
		height = 420
	} = $props();

	const margin = { top: 30, right: 30, bottom: 40, left: 40 };

	// Jadikan reaktif dengan $derived untuk menghilangkan warning
	let xScale = $derived(
		d3.scaleLinear()
			.domain([-6, 8])
			.range([margin.left, width - margin.right])
	);

	let yScale = $derived(
		d3.scaleLinear()
			.domain([-5, 3.5])
			.range([height - margin.bottom, margin.top])
	);

	const rScale = d3.scaleSqrt()
		.domain([0, 70000])
		.range([4, 18]);

	const vectorMultiplier = 6.8;

	function getColor(p) {
		if (p.group === "makmur") return "#0d9488";
		if (p.group === "sentra") return "#e11d48";
		if (p.group === "tertinggal") return "#64748b";
		return "#9ca3af";
	}

	function isHighlighted(p) {
		if (hoveredId !== null) return p.id === hoveredId;
		if (selectedIds.length > 0) return selectedIds.includes(p.id);
		return true;
	}

	function toggleSelect(id) {
		if (selectedIds.includes(id)) {
			selectedIds = selectedIds.filter((x) => x !== id);
		} else {
			selectedIds = [...selectedIds, id];
		}
	}
</script>

<div class="biplot-panel">
	<div class="panel-header">
		<span class="panel-title">PCA BIPLOT (PC1 vs PC2)</span>
		<span class="panel-hint">Klik titik untuk memilih wilayah</span>
	</div>

	<svg viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
		<defs>
			<marker id="arrow-biplot" viewBox="0 0 10 10" refX="6" refY="5" markerWidth="5" markerHeight="5" orient="auto-start-reverse">
				<path d="M 0 1.5 L 8 5 L 0 8.5 z" fill="#6b7280" />
			</marker>
		</defs>

		<!-- Sumbu 0 -->
		<line x1={xScale(-6)} y1={yScale(0)} x2={xScale(8)} y2={yScale(0)} stroke="#e5e7eb" stroke-dasharray="3 3" />
		<line x1={xScale(0)} y1={yScale(-5)} x2={xScale(0)} y2={yScale(3.5)} stroke="#e5e7eb" stroke-dasharray="3 3" />

		<!-- Vektor Indikator -->
		<g class="vectors" opacity="0.4">
			{#each data.loadings as vec}
				{@const vx = xScale(vec.loading_pc1 * vectorMultiplier)}
				{@const vy = yScale(vec.loading_pc2 * vectorMultiplier)}
				<line x1={xScale(0)} y1={yScale(0)} x2={vx} y2={vy} stroke="#6b7280" stroke-width="1" marker-end="url(#arrow-biplot)" />
				<text x={vx} y={vy - 3} font-size="8" fill="#4b5563" font-family="sans-serif">{vec.name}</text>
			{/each}
		</g>

		<!-- Titik Provinsi -->
		<g class="dots">
			{#each data.provinces as p}
				{@const active = isHighlighted(p)}
				{@const cx = xScale(p.pc1)}
				{@const cy = yScale(p.pc2)}
				{@const col = getColor(p)}
				<circle
					role="button"
					tabindex="0"
					{cx}
					{cy}
					r={hoveredId === p.id ? rScale(p.pmi_count) * 1.3 : rScale(p.pmi_count)}
					fill={col}
					stroke={active ? "#111827" : "#d1d5db"}
					stroke-width={hoveredId === p.id ? 2 : active ? 1.2 : 0.5}
					opacity={active ? 0.95 : 0.15}
					class="biplot-dot"
					onmouseenter={() => (hoveredId = p.id)}
					onmouseleave={() => (hoveredId = null)}
					onclick={() => toggleSelect(p.id)}
					onkeydown={(e) => { if (e.key === 'Enter') toggleSelect(p.id); }}
				/>
				{#if hoveredId === p.id || selectedIds.includes(p.id)}
					<text x={cx} y={cy - rScale(p.pmi_count) - 3} text-anchor="middle" font-size="9" font-weight="700" fill="#111827">
						{p.provinsi}
					</text>
				{/if}
			{/each}
		</g>
	</svg>
</div>

<style>
	.biplot-panel {
		background: transparent;
		border: none;
		padding: 0;
		box-shadow: none;
        width: 100%;
	}
	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.4rem;
	}
	.panel-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.75rem;
		font-weight: 700;
		color: #1f2937;
	}
	.panel-hint {
		font-family: var(--mono, monospace);
		font-size: 0.65rem;
		color: #9ca3af;
	}
	svg {
		width: 100%;
		height: auto;
		display: block;
	}
	.biplot-dot {
		cursor: pointer;
		transition: transform 0.2s, opacity 0.3s;
	}
</style>