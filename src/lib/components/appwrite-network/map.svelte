<script lang="ts">
    import { slugify } from '$lib/utils/slugify';
    import MapNav from './map-nav.svelte';
    import { useMousePosition } from '$lib/actions/mouse-position.svelte';
    import { useAnimateInView } from '$lib/actions/animate-in-view';
    import { pins, type PinSegment } from './data/pins';
    import MapTooltip, {
        handleSetActiveTooltip,
        handleResetActiveTooltip
    } from './map-tooltip.svelte';
    import { createMap } from 'svg-dotted-map';
    import { browser } from '$app/environment';

    let activeSegment = $state<string>('pop-locations');
    let activeMarkers = $derived(pins[activeSegment as PinSegment]);
    let activeRegion = $state<string | null>(null);
    let activeMarker: SVGGElement | null = null;

    const { action: mousePosition, position } = useMousePosition();
    const { action: inView } = useAnimateInView();

    const scrollMarkerIntoView = (marker: SVGGElement) => {
        return new Promise<void>((resolve) => {
            marker.scrollIntoView({
                behavior: 'smooth',
                block: 'nearest',
                inline: 'center'
            });

            const observer = new IntersectionObserver(
                (entries) => {
                    if (entries[0].intersectionRatio > 0.5) {
                        observer.disconnect();
                        resolve();
                    }
                },
                { threshold: [0, 0.25, 0.5, 0.75, 1] }
            );

            observer.observe(marker);
        });
    };

    const handleSetActiveMarker = async (city: string) => {
        const citySlug = slugify(city);

        if (activeRegion === citySlug) {
            activeMarker = null;
            activeRegion = null;
            return;
        }

        activeMarker = document.querySelector(`[data-region="${citySlug}"]`);

        if (activeMarker) {
            await scrollMarkerIntoView(activeMarker);
            activeRegion = citySlug;
        }
    };

    const radius = 0.4;
    const height = 75;
    const { points, addMarkers } = createMap({
        width: height * 2,
        height,
        mapSamples: 5000,
        radius
    });

    const markers = $derived(
        addMarkers<{ city: string; code: string; available?: boolean; date?: string }>(
            activeMarkers
        )
    );

    type Props = {
        theme: 'light' | 'dark';
    };

    const { theme = 'dark' }: Props = $props();
</script>

<div class="relative w-full overflow-x-scroll [scrollbar-width:none]">
    <div class="relative mx-auto h-full [scrollbar-width:none] md:w-full" use:inView>
        <div
            class="relative mx-auto my-10 h-fit w-full max-w-5xl origin-bottom transform-[perspective(25px)_rotateX(1deg)_scale3d(1.2,_1.2,_1)] transition-all [scrollbar-width:none] md:my-0 md:-translate-x-20"
            use:mousePosition
        >
            <svg viewBox={`0 0 ${height * 2} ${height}`}>
                {#each points as point}
                    <ellipse
                        cx={point.x}
                        cy={point.y}
                        rx={radius}
                        ry={radius * 1.25}
                        fill={theme === 'dark' ? 'rgba(255,255,255,.1)' : '#dadadd'}
                    />
                {/each}
                {#each markers as marker}
                    <g
                        role="tooltip"
                        class="animate-fade-in outline-none"
                        aria-label={`${marker.city} (${marker.code})`}
                        onmouseover={() =>
                            handleSetActiveTooltip(
                                marker.city,
                                marker.code,
                                marker.available,
                                marker.date
                            )}
                        onfocus={() =>
                            handleSetActiveTooltip(
                                marker.city,
                                marker.code,
                                marker.available,
                                marker.date
                            )}
                        onblur={() => handleResetActiveTooltip()}
                        onmouseout={() => handleResetActiveTooltip()}
                        data-region={slugify(marker.city)}
                    >
                        <circle cx={marker.x} cy={marker.y} r={radius * 1.25} class="fill-accent" />
                        <circle cx={marker.x} cy={marker.y} r={radius * 0.5} class="fill-white" />
                        <circle
                            cx={marker.x}
                            cy={marker.y}
                            r={radius * 4}
                            class="fill-transparent"
                        />
                    </g>
                {/each}
            </svg>
        </div>
    </div>
</div>
<MapTooltip {theme} {...position()} />

<MapNav {theme} onValueChange={(value) => (activeSegment = value)} />
