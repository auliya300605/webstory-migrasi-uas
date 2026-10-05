<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";

	let {
		data,
		selectedIds = $bindable([]),
		hoveredId = $bindable(null),
		width = 640,
		height = 240
	} = $props();

	const margin = { top: 35, right: 25, bottom: 25, left: 25 };

	const indicatorKeys = [
		{ key: "pdrb", label: "PDRB" },
		{ key: "persen_miskin", label: "Miskin %" },
		{ key: "ipm", label: "IPM" },
		{ key: "tpt", label: "TPT %" },
		{ key: "persen_pertanian", label: "Tani %" },
		{ key: "lama_sekolah", label: "RLS" },
		{ key: "infornal", label: "Informal %" },
		{ key: "ump", label: "UMP" },
		{ key: "dependecy", label: "Beban Rasio" }
	];

	// Skala dibuat reaktif via $derived agar tidak memicu warning state_referenced_locally
	let xScale = $derived(
		d3.scalePoint()
			.domain(indicatorKeys.map((d) => d.key))
			.range([margin.left, width - margin.right])
	);

	let yScales = $derived.by(() => {
		const scales = {};
		indicatorKeys.forEach(({ key }) => {
			const extent = d3.extent(data.provinces, (p) => p.raw_indicators?.[key] ?? 0);
			scales[key] = d3.scaleLinear()
				.domain(extent)
				.range([height - margin.bottom, margin.top]);
		});
		return scales;
	});

	function pathGenerator(p) {
		if (!p.raw_indicators) return "";
		const line = d3.line();
		const points = indicatorKeys.map(({ key }) => [
			xScale(key),
			yScales[key](p.raw_indicators[key] ?? 0)
		]);
		return line(points);
	}

	function getColor(p) {
		if (p.group === "makmur") return "#0d9488";
		if (p.group === "sentra") return "#e11d48";
		if (p.group === "tertinggal") return "#64748b";
		return "#9ca3af";
	}

	function isActive(p) {
		if (hoveredId !== null) return p.id === hoveredId;
		if (selectedIds.length > 0) return selectedIds.includes(p.id);
		return true;
	}

	let brushRanges = {};

	function onBrush(key, event) {
		if (!event.selection) {
			delete brushRanges[key];
		} else {
			const [y1, y0] = event.selection;
			brushRanges[key] = [yScales[key].invert(y0), yScales[key].invert(y1)];
		}

		const activeKeys = Object.keys(brushRanges);
		if (activeKeys.length === 0) {
			selectedIds = [];
		} else {
			selectedIds = data.provinces
				.filter((p) => {
					if (!p.raw_indicators) return false;
					return activeKeys.every((k) => {
						const val = p.raw_indicators[k];
						const [min, max] = brushRanges[k];
						return val >= min && val <= max;
					});
				})
				.map((p) => p.id);
		}
	}

	let gElements = {};
	onMount(() => {
		indicatorKeys.forEach(({ key }) => {
			const brush = d3.brushY()
				.extent([[-10, margin.top], [10, height - margin.bottom]])
				.on("brush end", (e) => onBrush(key, e));

			d3.select(gElements[key]).call(brush);
		});
	});
</script>

<div class="pcp-panel">
	<div class="panel-header">
		<span class="panel-title">PARALLEL COORDINATES PLOT (9 INDIKATOR DAERAH)</span>
		<span class="panel-hint">Tarik kuas vertikal pada sumbu untuk menyaring data</span>
	</div>

	<svg viewBox="0 0 {width} {height}" preserveAspectRatio="xMidYMid meet">
		<g class="lines-layer">
			{#each data.provinces as p}
				{@const active = isActive(p)}
				<!-- svelte-ignore a11y_no_static_element_interactions -->
				<path
					d={pathGenerator(p)}
					fill="none"
					stroke={getColor(p)}
					stroke-width={hoveredId === p.id ? 2.5 : active ? 1.4 : 0.6}
					opacity={hoveredId === p.id ? 1 : active ? 0.75 : 0.12}
					class="pcp-line"
					onmouseenter={() => (hoveredId = p.id)}
					onmouseleave={() => (hoveredId = null)}
				/>
			{/each}
		</g>

		{#each indicatorKeys as { key, label }}
			<g transform="translate({xScale(key)}, 0)">
				<line y1={margin.top} y2={height - margin.bottom} stroke="#9ca3af" stroke-width="1.2" />
				<text y={margin.top - 12} text-anchor="middle" font-size="8.5" font-weight="600" fill="#1f2937">
					{label}
				</text>
				<g bind:this={gElements[key]} class="axis-brush"></g>
			</g>
		{/each}
	</svg>
</div>

<style>
	.pcp-panel {
        background: transparent;
        border: none;
        box-shadow: none;
        padding: 0;
        width: 100%;
    }
	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 0.35rem;
	}
	.panel-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.72rem;
		font-weight: 700;
		color: #1f2937;
	}
	.panel-hint {
		font-family: var(--mono, monospace);
		font-size: 0.62rem;
		color: #6b7280;
	}
	svg {
		width: 100%;
		height: auto;
		display: block;
	}
	.pcp-line {
		transition: opacity 0.2s, stroke-width 0.2s;
		cursor: pointer;
	}
	:global(.axis-brush .selection) {
		fill: #d95f43;
		fill-opacity: 0.35;
		stroke: #d95f43;
	}
</style>