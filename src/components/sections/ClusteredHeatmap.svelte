<script>
	import * as d3 from "d3";

	let {
		data,
		selectedIds = $bindable([]),
		hoveredId = $bindable(null)
	} = $props();

	const indicatorKeys = [
		{ key: "pdrb", label: "PDRB (Jt Rp)" },
		{ key: "persen_miskin", label: "Miskin (%)" },
		{ key: "ipm", label: "IPM" },
		{ key: "tpt", label: "TPT (%)" },
		{ key: "persen_pertanian", label: "Tani (%)" },
		{ key: "lama_sekolah", label: "RLS (Thn)" },
		{ key: "infornal", label: "Informal (%)" },
		{ key: "ump", label: "UMP (Jt Rp)" },
		{ key: "dependecy", label: "Beban Rasio" }
	];

	// Urutkan provinsi berdasarkan klaster tipologi
	let sortedProvinces = $derived(
		[...data.provinces].sort((a, b) => {
			const order = { makmur: 0, sentra: 1, tertinggal: 2, other: 3 };
			if (order[a.group] !== order[b.group]) return order[a.group] - order[b.group];
			return b.pc1 - a.pc1;
		})
	);

	// Hitung min-max per indikator secara reaktif
	let minMax = $derived.by(() => {
		const res = {};
		indicatorKeys.forEach(({ key }) => {
			res[key] = d3.extent(data.provinces, (p) => p.raw_indicators?.[key] ?? 0);
		});
		return res;
	});

	function getCellColor(key, value) {
		const [min, max] = minMax[key] || [0, 1];
		const t = max > min ? (value - min) / (max - min) : 0.5;
		return d3.interpolateYlOrRd(t * 0.85);
	}

	function isRowActive(p) {
		if (hoveredId !== null) return p.id === hoveredId;
		if (selectedIds.length > 0) return selectedIds.includes(p.id);
		return true;
	}
</script>

<div class="heatmap-panel">
	<div class="panel-header">
		<div class="title-wrap">
			<span class="panel-title">CLUSTERED HEATMAP MATRIX (34 PROVINSI vs 9 INDIKATOR)</span>
			<span class="panel-subtitle">Warna sel menunjukkan intensitas nilai indikator ternormalisasi</span>
		</div>
		<span class="panel-hint">Arahkan kursor pada baris untuk menyorot Biplot & PCP</span>
	</div>

	<div class="table-scroll-wrapper">
		<table class="heatmap-table">
			<thead>
				<tr>
					<th class="th-prov">Provinsi</th>
					<th class="th-group">Klaster</th>
					{#each indicatorKeys as { label }}
						<th>{label}</th>
					{/each}
				</tr>
			</thead>
			<tbody>
				{#each sortedProvinces as p}
					{@const active = isRowActive(p)}
					<tr
						class:active-row={hoveredId === p.id}
						class:dimmed-row={!active}
						onmouseenter={() => (hoveredId = p.id)}
						onmouseleave={() => (hoveredId = null)}
					>
						<td class="td-prov">{p.provinsi}</td>
						<td class="td-group">
							<span class="badge {p.group}">{p.group}</span>
						</td>
						{#each indicatorKeys as { key }}
							{@const rawVal = p.raw_indicators?.[key] ?? 0}
							{@const cellMax = minMax[key]?.[1] ?? 1}
							{@const displayVal = key === "ump" ? (rawVal / 1000000) : rawVal}
							<td
								class="td-cell"
								style="background-color: {getCellColor(key, rawVal)}; color: {rawVal > (cellMax * 0.75) ? '#fff' : '#1f2937'};"
							>
								{Number(displayVal).toLocaleString("id-ID", { maximumFractionDigits: 1 })}
							</td>
						{/each}
					</tr>
				{/each}
			</tbody>
		</table>
	</div>
</div>

<style>
	.heatmap-panel {
        background: transparent;
        border: none;
        box-shadow: none;
        padding: 0;
        width: 100%;
        box-sizing: border-box;
    }
	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: flex-end;
		margin-bottom: 0.5rem;
	}
	.title-wrap {
		display: flex;
		flex-direction: column;
		gap: 2px;
	}
	.panel-title {
		font-family: var(--sans, sans-serif);
		font-size: 0.75rem;
		font-weight: 700;
		color: #1f2937;
		letter-spacing: 0.03em;
	}
	.panel-subtitle {
		font-family: var(--mono, monospace);
		font-size: 0.62rem;
		color: #6b7280;
	}
	.panel-hint {
		font-family: var(--mono, monospace);
		font-size: 0.62rem;
		color: #9ca3af;
	}
	.table-scroll-wrapper {
		max-height: 200px;
		overflow-y: auto;
		border: 1px solid #e5e7eb;
		border-radius: 4px;
	}
	.heatmap-table {
		width: 100%;
		border-collapse: collapse;
		font-family: var(--sans, sans-serif);
		font-size: 0.7rem;
		text-align: right;
	}
	th {
		position: sticky;
		top: 0;
		background: #f9fafb;
		padding: 5px 8px;
		font-weight: 600;
		color: #4b5563;
		border-bottom: 1px solid #e5e7eb;
		z-index: 2;
		white-space: nowrap;
	}
	.th-prov {
		text-align: left;
		min-width: 130px;
	}
	.th-group {
		text-align: center;
		min-width: 80px;
	}
	td {
		padding: 3.5px 8px;
		border-bottom: 1px solid #f3f4f6;
	}
	.td-prov {
		text-align: left;
		font-weight: 500;
		white-space: nowrap;
	}
	.td-group {
		text-align: center;
	}
	.badge {
		font-size: 0.58rem;
		padding: 1px 5px;
		border-radius: 3px;
		text-transform: uppercase;
		font-weight: 700;
	}
	.badge.makmur { background: #ccfbf1; color: #0f766e; }
	.badge.sentra { background: #ffe4e6; color: #be123c; }
	.badge.tertinggal { background: #f1f5f9; color: #475569; }
	.badge.other { background: #f3f4f6; color: #9ca3af; }
	.td-cell {
		font-family: var(--mono, monospace);
		font-size: 0.68rem;
	}
	tr {
		transition: opacity 0.2s, background 0.15s;
		cursor: pointer;
	}
	.active-row {
		background: #eef2ff !important;
		font-weight: bold;
	}
	.dimmed-row {
		opacity: 0.22;
	}
</style>