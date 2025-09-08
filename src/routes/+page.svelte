<script lang="ts">
	import type { MapOptions } from 'leaflet';
	import { onMount } from 'svelte';
	import type {
		Feature as GJFeature,
		FeatureCollection as GJFeatureCollection,
		Point as GJPoint
	} from 'geojson';
	import type * as LNS from 'leaflet';

	type Properties = { name?: string; rating?: number; tags?: string[]; notes?: string };
	type Feature = GJFeature<GJPoint, Properties>;

	type FeatureCollection = GJFeatureCollection<GJPoint, Properties>;

	let mapEl: HTMLDivElement | null = null;
	let L: typeof import('leaflet') | null = null;
	let map: LNS.Map | null = null;
	let markersLayer: LNS.LayerGroup | null = null;
	let customIcon: LNS.DivIcon | null = null;
	let popup: LNS.Popup | null = null;

	function esc(x: string): string {
		return x.replace(
			/[&<>\"']/g,
			(m) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[m]!
		);
	}

	function mapsHref(lat: number, lng: number, name?: string): string {
		const ua = navigator.userAgent || '';
		const label = encodeURIComponent(name ?? `${lat},${lng}`);
		if (/Android/i.test(ua)) return `geo:${lat},${lng}?q=${lat},${lng}(${label})`;
		if (/iPhone|iPad|iPod|Macintosh/i.test(ua) && /AppleWebKit/i.test(ua))
			return `https://maps.apple.com/?ll=${lat},${lng}&q=${label}`;
		return `https://www.google.com/maps/search/?api=1&query=${lat},${lng}`;
	}

	async function loadSpots(): Promise<FeatureCollection> {
		const res = await fetch('/spots.json', { cache: 'no-cache' });
		if (!res.ok) throw new Error(`spots.json load failed: ${res.status}`);
		return (await res.json()) as FeatureCollection;
	}

	function mkPopupHtml(f: Feature, lat: number, lng: number): string {
		const name = f.properties?.name ?? 'Ohne Titel';
		const rating = f.properties?.rating ? `${Math.max(1, Math.min(5, f.properties.rating))}/5` : '';
		const tags = (f.properties?.tags ?? [])
			.map(esc)
			.map((t) => `#${t}`)
			.join(' ');
		const notes = f.properties?.notes ? esc(f.properties.notes) : '';
		const href = mapsHref(lat, lng, name);

		return `
		<div>
			<div class="flex items-baseline justify-between gap-2">
			<h3 class="text-lg font-medium">${esc(name)}</h3>
			<span class="text-yellow-600 flex items-center gap-0.5">
				${rating}
				<svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24"> <path fill="currentColor" d="m8.243 7.34l-6.38.925l-.113.023a1 1 0 0 0-.44 1.684l4.622 4.499l-1.09 6.355l-.013.11a1 1 0 0 0 1.464.944l5.706-3l5.693 3l.1.046a1 1 0 0 0 1.352-1.1l-1.091-6.355l4.624-4.5l.078-.085a1 1 0 0 0-.633-1.62l-6.38-.926l-2.852-5.78a1 1 0 0 0-1.794 0z" /> </svg>
			</span>
			</div>
			<hr class="border-t border-muted mt-2" />
			${notes ? `<p class="!mt-2 !mb-0">${notes}</p>` : ''}
			<div class="mt-2 text-muted-foreground">${tags}</div>
			<div class="mt-4">
				<a href="${href}" target="_blank" rel="noopener" class="inline-flex items-center justify-center gap-1 w-full bg-primary !text-primary-foreground font-medium rounded-xs p-2 hover:bg-primary/90 transition">
					<svg aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24">
						<g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2">
							<path d="M9 11a3 3 0 1 0 6 0a3 3 0 0 0-6 0" />
							<path d="M17.657 16.657L13.414 20.9a2 2 0 0 1-2.827 0l-4.244-4.243a8 8 0 1 1 11.314 0" />
						</g>
					</svg>
					<span>Öffnen in Karten</span>
				</a>
			</div>
		</div>`;
	}

	function renderFeatures(fc: FeatureCollection): void {
		if (!L || !map) return;

		if (!markersLayer) markersLayer = L.layerGroup().addTo(map);
		else markersLayer.clearLayers();

		if (!customIcon) {
			customIcon = L.divIcon({
				className:
					'!size-8 bg-primary text-primary-foreground rounded-full !flex !items-center !justify-center',
				html: `<svg aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"> <g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2"> <path d="M3 14c.83.642 2.077 1.017 3.5 1c1.423.017 2.67-.358 3.5-1s2.077-1.017 3.5-1c1.423-.017 2.67.358 3.5 1M8 3a2.4 2.4 0 0 0-1 2a2.4 2.4 0 0 0 1 2m4-4a2.4 2.4 0 0 0-1 2a2.4 2.4 0 0 0 1 2" /> <path d="M3 10h14v5a6 6 0 0 1-6 6H9a6 6 0 0 1-6-6z" /> <path d="M16.746 16.726a3 3 0 1 0 .252-5.555" /> </g> </svg>`,
				iconAnchor: [16, 16],
				popupAnchor: [0, -16]
			});
		}

		if (!popup) popup = L.popup({ maxWidth: 320, autoPan: true });

		for (const f of fc.features) {
			if (!f || f.type !== 'Feature' || f.geometry?.type !== 'Point') continue;
			const [lng, lat] = f.geometry.coordinates;
			const marker = L.marker([lat, lng], { icon: customIcon! });
			marker.on('click', () => {
				popup!
					.setLatLng([lat, lng])
					.setContent(mkPopupHtml(f, lat, lng))
					.openOn(map!);
			});
			marker.addTo(markersLayer);
		}
	}

	const options: MapOptions = {
		center: [48.7784, 9.1806],
		zoom: 13,
		preferCanvas: false,
		zoomAnimation: true
	} satisfies MapOptions;

	onMount(async () => {
		L = (await import('leaflet')) as typeof import('leaflet');
		await import('leaflet/dist/leaflet.css');

		if (!mapEl) return;
		map = L.map(mapEl, options);

		L.tileLayer('https://tile.openstreetmap.de/{z}/{x}/{y}.png', {
			attribution:
				'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
			updateWhenIdle: true,
			updateWhenZooming: false,
			keepBuffer: 1,
			minZoom: 1
		}).addTo(map);

		const fc = await loadSpots();
		renderFeatures(fc);
	});
</script>

<svelte:head>
	<title>Favoriten von Tim Wehrle</title>
</svelte:head>

<div class="h-screen w-full" bind:this={mapEl}></div>

<style>
	:global(.leaflet-popup-content-wrapper) {
		background: var(--background) !important;
		border: 1px solid var(--border);
		padding: 0 !important;
		border-radius: var(--radius) !important;
		min-width: 10rem;
		width: 100%;
	}

	:global(.leaflet-popup-content) {
		font-family: system-ui, sans-serif;
		font-size: 0.9rem;
		padding: 1rem;
		margin: 0 !important;
	}

	:global(.leaflet-popup-close-button) {
		transition: color 0.2s ease;
	}
</style>
