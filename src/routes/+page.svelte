<script>
    import { base } from '$app/paths';
    import Select from 'svelte-select';
    import { MapLibre, GeoJSON, LineLayer, CircleLayer, Popup } from 'svelte-maplibre';
    import * as Plot from "@observablehq/plot";
    import { tableFromIPC } from "apache-arrow";
    import geojsonbounds from "geojson-bounds";
    import {format} from "isoformat";
    import CarbonTropicalStorm from '~icons/carbon/tropical-storm';
    import PhInfoDuotone from '~icons/ph/info-duotone';
    import PhCaretDownDuotone from '~icons/ph/caret-down-duotone';
    import PhCaretUpDuotone from '~icons/ph/caret-up-duotone';
    import { tooltip } from "@svelte-plugins/tooltips";
    import pnnlLogo from "$lib/img/pnnl-logo.svg";
    import msdLogo from "$lib/img/MSD.LIVE.horiz.svg";
    import icomLogo from "$lib/img/icom-logo.svg";

    import storms from "$lib/storms.js";

    let scrollElement;
    let mapPopupMode = 'hover';

    const chartWidth = 640;
    const chartHeight = 500;

    const smallChartWidth = 320;
    const smallChartHeight = 250;

    const scenarios = [
        {
            name: 'IBTrACS',          id: 'ibtracs',          color: '0      0%     100%  ', symbol: 'line',
            description: 'IBTrACS'
        },
        {
            name: 'RAFT baseline',    id: 'baseline',         color: '280    30.51% 76.86%', symbol: 'square',
            description: 'RAFT baseline',
        },
        {
            name: 'ssp245 Near Cold', id: 'ssp245_near_cold', color: '200.66 52.14% 77.06%', symbol: 'circle',
            description: 'RAFT ssp245 Near Cold',
        },
        {
            name: 'ssp245 Far Cold',  id: 'ssp245_far_cold',  color: '91.76  57.05% 70.78%', symbol: 'circle',
            description: 'RAFT ssp245 Far Cold',
        },
        {
            name: 'ssp245 Near Hot',  id: 'ssp245_near_hot',  color: '204.16 70.62% 41.37%', symbol: 'circle',
            description: 'RAFT ssp245 Near Hot',
        },
        {
            name: 'ssp245 Far Hot',   id: 'ssp245_far_hot',   color: '116.38 56.86% 40%   ', symbol: 'circle',
            description: 'RAFT ssp245 Far Hot',
        },
        {
            name: 'ssp585 Near Cold', id: 'ssp585_near_cold', color: '0.61   92.45% 79.22%', symbol: 'circle',
            description: 'RAFT ssp585 Near Cold',
        },
        {
            name: 'ssp585 Far Cold',  id: 'ssp585_far_cold',  color: '33.8   97.26% 71.37%', symbol: 'circle',
            description: 'RAFT ssp585 Far Cold',
        },
        {
            name: 'ssp585 Near Hot',  id: 'ssp585_near_hot',  color: '359.4  79.45% 49.61%', symbol: 'circle',
            description: 'RAFT ssp585 Near Hot',
        },
        {
            name: 'ssp585 Far Hot',   id: 'ssp585_far_hot',   color: '29.88  100%   50%   ', symbol: 'circle',
            description: 'RAFT ssp585 Far Hot',
        },
    ];
    let selectedScenarios = scenarios.slice(0, 2).map(s => s.id);

    const colorScale = [
        { min:   0, max:  25,      color: '#211459', },
        { min:  25, max:  35,      color: '#334080', },
        { min:  35, max:  45,      color: '#4f7d9e', },
        { min:  45, max:  55,      color: '#5ca9a3', },
        { min:  55, max:  65,      color: '#6bc28c', },
        { min:  65, max:  75,      color: '#94d68c', },
        { min:  75, max:  85,      color: '#c7eb8c', },
        { min:  85, max:  95,      color: '#fff273', },
        { min:  95, max: 105,      color: '#f2d67d', },
        { min: 105, max: 115,      color: '#e6ab6e', },
        { min: 115, max: 125,      color: '#d98254', },
        { min: 125, max: 135,      color: '#cc5940', },
        { min: 135, max: 145,      color: '#bd3d3d', },
        { min: 145, max: 155,      color: '#ab2b40', },
        { min: 155, max: Infinity, color: '#8a083d', },
    ];
    function getColor(v) {
        return colorScale.find(s => (v >= s.min) && (v < s.max))?.color;
    }

    let vmaxPlot;
    let vmpiPlot;
    let u200Plot;
    let shrdPlot;
    let delvPlot;
    let lp500Plot;
    let pslvPlot;
    let rhloPlot;
    let latPlot;
    let lonPlot;

    let storm; // = storms.find(d => d.id == 2011);
    let isFetching = false;
    let stormData;
    let stormTrack;
    let stormPoints;
    let stormBounds;

    let showMore = false;

    function toggleShowMore() {
        showMore = !showMore;
        setTimeout(() => {
            requestAnimationFrame(() => {
                scrollElement?.scroll({ top: scrollElement?.scrollHeight, behavior: 'smooth' });
            });
        }, 200);
    }

    function fetchData(storm) {
        isFetching = true;
        const path = `${base}/data/${storm.value}.arrow`;
        tableFromIPC(fetch(path)).then((table) => table.toArray().map((item) => {
            const d = item.toJSON();
            return d;
        })).then(d => {
            stormData = d;
            stormTrack = {
                type: 'Feature',
                geometry: {
                    type: 'LineString',
                    coordinates: stormData.map(d => ([d.lon, d.lat])),
                },
            };
            stormBounds = geojsonbounds.extent(stormTrack);
            stormBounds = [stormBounds[0] - 2, stormBounds[1] - 2, stormBounds[2] + 2, stormBounds[3] + 2];
            stormPoints = {
                "type": "FeatureCollection",
                "features": stormData.filter(d => d.vmax_kts_baseline).map(d => ({
                    "type": "Feature",
                    "geometry": {
                        "type": "Point",
                        "coordinates": [d.lon, d.lat]
                    },
                    "properties": {
                        "time": d.time,
                        "color_undefined": "white",
                        ...scenarios.reduce((o, s) => {
                            o[`vmax_kts_${s.id}`] = d[`vmax_kts_${s.id}`];
                            o[`color_${s.id}`] = getColor(d[`vmax_kts_${s.id}`]);
                            return o;
                        }, {}),
                    }
                })),
            };
        }).catch(e => {
            console.log(e);
        }).finally(() => {
            isFetching = false;
        });
    }

    function changeStorm(scroll=true) {
        stormData   = null;
        stormTrack  = null;
        stormPoints = null;
        showMore = false;
        if (scroll) {
            scrollElement?.scroll({ top: scrollElement?.scrollHeight, behavior: 'smooth' });
        }
    }

    function clearStorm() {
        storm = undefined;
        changeStorm(false);
        vmaxPlot?.firstChild.remove();
    }

    function selectAllScenarios() {
        selectedScenarios = scenarios.map(s => s.id);
    }

    function resetScenarios() {
        selectedScenarios = scenarios.slice(0, 2).map(s => s.id);
    }

    $: if (storm && !stormData && !isFetching) {
        fetchData(storm);
    }

    $: activeScenarios = scenarios.filter(s => selectedScenarios.includes(s.id));

    $: if (storm && stormData && activeScenarios) {
        const useColorScale = activeScenarios.filter(s => s.id !== 'ibtracs').length <= 1;

        // intensity plot
        vmaxPlot?.firstChild?.remove();
        vmaxPlot?.append(Plot.plot({
            width: chartWidth/1.25,
            height: chartHeight/1.25,
            x: {type: 'utc', label: 'Time (UTC)'},
            y: {grid: true, label: 'Intensity vmax (knots)', domain: [0, 185]},
            marks: [
                Plot.rect(stormData.reduce((o, d) => {
                    if (o.length===0) {
                        if (d.over_land) {
                            o = [{
                                x1: d.time,
                            }];
                            return o;
                        }
                        return o;
                    }
                    else if (!o[o.length-1].x2) {
                        if (!d.over_land) {
                            o[o.length-1].x2 = d.time;
                            return o;
                        }
                        else if (d === stormData[stormData.length-1]) {
                            o[o.length-1].x2 = d.time;
                            return o;
                        }
                        return o;
                    }
                    else {
                        if (d.over_land) {
                            o.push({
                                x1: d.time,
                            });
                            return o;
                        }
                        return o;
                    }
                }, []), {
                    x1: 'x1',
                    x2: 'x2',
                    y1: 0,
                    y2: 185,
                    fill: 'white',
                    fillOpacity: 0.1,
                }),
                activeScenarios.find(s => s.id == 'ibtracs') && Plot.line(stormData.filter(d => (d.vmax_kts_ibtracs != null)), {
                    x: "time", y: "vmax_kts_ibtracs",
                    strokeWidth: 4,
                    curve: 'catmull-rom',
                    stroke: `hsl(${scenarios[0].color})`,
                }),
                Plot.ruleX(stormData.filter(d => (d['vmax_kts_baseline'] != null)), Plot.pointerX({
                    x: "time",
                    stroke: "red",
                    channels: {
                        'Latitude': 'lat',
                        'Longitude': 'lon',
                        ...activeScenarios.reduce((o, s) => ({
                            ...o,
                            [s.name]: `vmax_kts_${s.id}`
                        }), {})
                    },
                    tip: {
                        fill: 'black',
                        pointerSize: 0,
                        frameAnchor: "top",
                        fontSize: 12,
                        format: {
                            x: true,
                            y: false,
                            stroke: false,
                            fill: false,
                            Latitude: true,
                            Longitude: true,
                            ...activeScenarios.reduce((o, s) =>({
                                ...o,
                                [s.name]: d => d.toFixed(0),
                            }), {})
                        },
                    },
                })),
                ...activeScenarios.filter(s => s.id !== 'ibtracs').map(s => (
                    Plot.dot(stormData.filter(d => d[`vmax_kts_${s.id}`]), {
                        x: "time", y: `vmax_kts_${s.id}`,
                        symbol: s.symbol,
                        r: 7, strokeWidth: 2, fillOpacity: 0.8,
                        stroke: useColorScale ? (d => getColor(d.vmax_kts_baseline)) : `hsl(${s.color})`,
                        fill: useColorScale ? (d => getColor(d.vmax_kts_baseline)) : `hsl(${s.color})`,
                    }))
                ),
                ...[
                    {color: '#bd3d3d',  knots: 137},
                    {color: '#d98254',  knots: 113},
                    {color: '#f2d67d',  knots:  96},
                    {color: '#c7eb8c',  knots:  83},
                    {color: '#6bc28c',  knots:  64},
                    {color: '#4f7d9e',  knots:  34},
                    {color: '#ffffff',  knots:   0},
                ].map(cat => [
                    Plot.ruleY(
                        [cat.knots],
                        {
                            stroke: cat.color,
                            strokeOpacity: cat.knots === 0 ? 1 : 0.667,
                            strokeWidth: cat.knots === 0 ? 1 : 1.5,
                        }),
                ]),
                Plot.axisY({grid: true, label: 'Intensity vmax (knots)', domain: [0, 185]}),
                Plot.axisY({
                    label: '',
                    anchor: 'right',
                    labelAnchor: 'center',
                    labelArrow: false,
                    labelOffset: 0,
                    marginTop: 0,
                    marginRight: 0,
                    marginBottom: 0,
                    marginLeft: 0,
                    ticks: [0, 34, 64, 83, 96, 113, 137],
                    tickSize: 0,
                    tickPadding: 0,
                    dx: -28,
                    dy: -6,
                    lineAnchor: 'bottom',
                    tickFormat: (d, i, _) => ['    TD', '    TS', 'Cat 1', 'Cat 2', 'Cat 3', 'Cat 4', 'Cat 5'][i],
                }),
            ],
        }));

        // bonus plots
        const bonusPlots = [
            { element: vmpiPlot,  variable: 'VMPI_t0',  title: "VMPI",   domain: [0, 200],     },
            { element: u200Plot,  variable: 'U200_t0',  title: "U200",   domain: [-500, 1100], },
            { element: shrdPlot,  variable: 'SHRD_t0',  title: "SHRD",   domain: [0, 1000],    },
            { element: delvPlot,  variable: 'DELV_6',   title: "DELV",   domain: [-50, 20],    },
            { element: rhloPlot,  variable: 'RHLO_t0',  title: "RHLO",   domain: [20, 100],    },
        ];
        const bonusPlotsStatic = [
            { element: lp500Plot, variable: 'LP500_t0_baseline', title: "LP500",  domain: [0, 1],       },
            { element: pslvPlot,  variable: 'PSLV_v3_baseline',  title: "PSLV",   domain: [0, 450],     },
            { element: latPlot,   variable: 'lat',  title: "latitude",   domain: [0, 55],      },
            { element: lonPlot,   variable: 'lon',  title: "longitude",  domain: [-115, 0],    },
        ];
        bonusPlots.forEach(bp => {
            bp.element?.firstChild?.remove();
            bp.element?.append(Plot.plot({
                width: smallChartWidth,
                height: smallChartHeight,
                x: {type: 'utc', label: '', labelArrow: 'none',},
                y: {grid: true, label: '', labelArrow: 'none', domain: bp.domain},
                marks: [
                    ...activeScenarios.filter(s => s.id !== 'ibtracs').flatMap(s => ([
                        Plot.line(stormData.filter(d => d[`${bp.variable}_${s.id}`]), {
                            x: "time", y: `${bp.variable}_${s.id}`,
                            strokeWidth: 1,
                            curve: 'catmull-rom',
                            stroke: `hsl(${s.color})`,
                        }),
                        Plot.dot(stormData.filter(d => (d[`${bp.variable}_${s.id}`] != null)), {
                            x: "time", y: `${bp.variable}_${s.id}`,
                            symbol: s.symbol,
                            r: 2,
                            fill: `hsl(${s.color})`,
                        }),
                    ])),
                    Plot.ruleY([0]),
                ],
            }));
        });
        bonusPlotsStatic.forEach(bp => {
            bp.element?.firstChild?.remove();
            bp.element?.append(Plot.plot({
                width: smallChartWidth,
                height: smallChartHeight,
                x: {type: 'utc', label: '', labelArrow: 'none'},
                y: {grid: true, label: '', labelArrow: 'none', domain: bp.domain},
                marks: [
                    Plot.line(stormData.filter(d => (d[`${bp.variable}`]) != null), {
                        x: "time", y: `${bp.variable}`,
                        strokeWidth: 2,
                        curve: 'catmull-rom',
                        stroke: 'white',
                    }),
                    Plot.dot(stormData.filter(d => (d.vmax_kts_baseline != null) && (d[`${bp.variable}`] != null)), {
                        x: "time", y: `${bp.variable}`,
                        r: 2,
                        stroke: 'white',
                    }),
                    Plot.ruleY([0]),
                ],
            }));
        });
    }

    function onKeyUp(e) {
        if (e.key === "`")
            mapPopupMode = mapPopupMode === 'click' ? 'hover' : 'click';
    }

</script>

<svelte:window
    on:keyup={onKeyUp}
/>

<div class="absolute top-0 left-0 w-full h-full bg-fixed bg-cover bg-no-repeat -z-10 bg-[url('$lib/img/patricia_nasa_scott_kelly.jpg')]" />

{#if storm}
<div bind:this={scrollElement} class="w-screen h-screen overflow-y-auto relative">
    
    <div class="z-10 sticky top-0 flex flex-row text-white h-20 px-4 bg-black items-center justify-between shrink-0">
        <button
            class="flex flex-row items-center justify-start select-none text-start"
            on:click={clearStorm}
        >
            <img class="w-12 h-12 mr-4 rounded-full overflow-hidden" alt="" src="{base}/cyclone_bw.png" />
            <div class="flex flex-col">
                <h1 class="text-xl font-semibold leading-tight">
                    ICoM RAFT
                </h1>
                <h2 class="text-lg leading-tight">
                    Hurricane <span class="hidden sm:inline">Projections</span> Dashboard
                </h2>
            </div>
        </button>
        <div class="flex-1 max-w-96 flex flex-row items-center justify-center">
            <label class="mr-2 text-base font-light" for="storm">
                STORM:
            </label>
            <Select
                id="storm"
                searchable
                items={storms}
                bind:value={storm}
                on:change={changeStorm}
                on:clear={clearStorm}
                placeholder="search storms..."
                --border-radius="0"
                --border-radius-focused="0"
                --item-first-border-radius="0"
                --list-border-radius="0"
                --item-color="black"
                --selected-item-color="black"
                --chevron-color="black"
                --clear-icon-color="red"
                --item-hover-color="black"
            />
        </div>
    </div>

    <div class="flex flex-col w-full bg-black/90 backdrop-blur backdrop-grayscale">

        <div class="flex flex-row flex-wrap w-full min-h-[calc(100vh-7.5rem)]">

            <div class="w-[var(--width)] shrink-0 h-full p-4 flex flex-col gap-y-8" style="--width:{chartWidth}px">
                <div class="flex flex-row">
                    <div class="flex flex-col gap-y-2">
                        <h3 class="text-xl font-semibold flex flex-row items-center gap-x-2">
                            Scenarios
                            <button
                                class="relative tooltip-container cursor-help"
                                style="
                                    --tooltip-background-color: #fff;
                                    --tooltip-color: #000;
                                "
                                use:tooltip={{
                                    content: `<div class="cursor-default">
                                        Eight future climate scenarios are derived by analyzing monthly data from two emission scenarios,
                                        SSP585 and SSP245, using global climate models categorized as 'hot' or 'cold' based on their
                                        sensitivity to global warming. The 'near future' period spans from 2020 to 2059 while the
                                        'far future' covers 2060 to 2099. The future climate signals are directly applied to the following
                                        model inputs: VMPI, U200, SHRD, and RHLO. Changes to these inputs can be viewed in the 'MORE'
                                        section when future scenarios are selected. Future climate signals are derived following the method
                                        in the Jones et. al.'s
                                        <a
                                            class="text-blue-400 cursor-pointer hover:underline underline-offset-4"
                                            href="https://tgw-data.msdlive.org" target="_blank" rel="noreferrer"
                                        >
                                        <i>Continental United States climate projections based on thermodynamic modification of
                                        historical weather</i></a>.</div>
                                    `,
                                    theme:'offset-tooltip',
                                    autoPosition: true,
                                    hideOnClickOutside: true,
                                    action: 'click',
                                    position: 'right',
                                    maxWidth: 600,
                                }}
                            >
                                <PhInfoDuotone />
                            </button>
                        </h3>
                        <div class="flex flex-row items-center gap-x-4 h-8 pb-2 ">
                            <button
                                type="button"
                                class="text-xs font-light border border-white px-2 shadow-md shadow-white/25 hover:shadow-white/50 active:shadow-none active:mt-1"
                                on:click={selectAllScenarios}
                            >
                                SELECT ALL
                            </button>
                            <button
                                type="button"
                                class="text-xs font-light border border-white px-2 shadow-md shadow-white/25 hover:shadow-white/50 active:shadow-none active:mt-1"
                                on:click={resetScenarios}
                            >
                                RESET
                            </button>
                        </div>
                        <div class="flex flex-row flex-wrap gap-x-8 gap-y-4">
                        {#each scenarios as s}
                            <label class="w-56 flex flex-row items-center gap-x-2">
                                <input class="" type="checkbox" disabled={s.id==='ibtracs'} bind:group={selectedScenarios} value={s.id}/>
                                {#if (s.symbol === 'circle')}
                                <div class="w-4 h-4 rounded-full bg-custom-color/50 border border-custom-color" style="--custom-color:{s.color}"/>
                                {:else if (s.symbol === 'square')}
                                <div class="w-4 h-4 bg-custom-color/50 border border-custom-color" style="--custom-color:{s.color}"/>
                                {:else if (s.symbol === 'line')}
                                <div class="w-4 h-1 bg-custom-color" style="--custom-color:{s.color}"/>
                                {/if}
                                {s.name}
                            </label>
                        {/each}
                        </div>
                    </div>
                    <div class="flex flex-col items-end">
                        <div
                            class="transition-all
                            {(activeScenarios.filter(s => s.id !== 'ibtracs').length <= 1) ? 'grayscale-0 opacity-100' : 'grayscale opacity-25'}
                            "
                        >
                            <span class="">
                                Knots
                            </span>
                            <div class="flex flex-row items-start">
                                <div class="flex flex-col-reverse pt-2">
                                    {#each colorScale as c}
                                        <span class="w-8 h-4 text-xs text-end">
                                            {c.min} -
                                        </span>
                                    {/each}
                                    <span class="w-8 h-4 text-xs text-end">
                                        {colorScale[colorScale.length-1].min + 10} -
                                    </span>
                                </div>
                                <div class="flex flex-col-reverse">
                                {#each colorScale as c}
                                    <div class="w-4 h-4 bg-[var(--color)]" style="--color: {c.color}" />
                                {/each}
                                <div style="
                                    width: 0;
                                    height: 0;
                                    border: 0.5rem solid transparent;
                                    border-top: 0;
                                    border-bottom: 1rem solid {colorScale[colorScale.length-1].color};
                                "/>
                                </div>
                            </div>
                        </div>
                        <div class="flex flex-row items-end justify-center gap-x-1 mt-2">
                            <span class="text-xs">Overland:</span>
                            <div class="w-3 h-5 bg-white/25"/>
                        </div>
                    </div>
                </div>
                <div bind:this={vmaxPlot} class="svg-container w-full flex-1 shrink-0" role="img" />
            </div>

            <div class="flex-1 w-full min-w-96 min-h-96 h-auto p-4">
                {#if stormBounds}
                <MapLibre
                    bounds={stormBounds}
                    class="w-full h-full"
                    standardControls
                    cooperativeGestures
                    style="https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json"
                >
                    {#if stormTrack}
                    <GeoJSON id="track" data={stormTrack}>
                        <LineLayer
                            layout={{ 'line-cap': 'round', 'line-join': 'round' }}
                            paint={{
                                'line-width': 4,
                                'line-color': '#ffffff',
                                'line-opacity': 0.8,
                            }}
                        />
                    </GeoJSON>
                    <GeoJSON id="points" data={stormPoints}>
                        <CircleLayer
                            paint={{
                                'circle-radius': [
                                    'interpolate',
                                    ['exponential', 0.9],
                                    ['zoom'],
                                    4, 7,
                                    23, 100
                                ],
                                'circle-color': scenarios.filter(s => selectedScenarios.includes(s.id) && (s.id !== 'ibtracs')).length <= 1
                                    ? [
                                        'get',
                                        `color_${scenarios.find(s => s.id === selectedScenarios[selectedScenarios.length-1])?.id}`
                                      ]
                                    : 'black',
                                'circle-opacity': 0.9,
                                'circle-stroke-width': 3,
                                'circle-stroke-color': scenarios.filter(s => selectedScenarios.includes(s.id) && (s.id !== 'ibtracs')).length <= 1
                                    ? [
                                        'get',
                                        `color_${scenarios.find(s => s.id === selectedScenarios[selectedScenarios.length-1])?.id}`
                                      ]
                                    : 'white',
                                'circle-stroke-opacity': 1,
                            }}
                        >
                            <Popup openOn={mapPopupMode} let:features>
                                <div class="text-sm text-white">
                                    <p><span class="font-semibold">Time (UTC):</span> {format(features[0].properties.time)}</p>
                                    <p><span class="font-semibold">Latitude:</span> {features[0].geometry.coordinates[1].toFixed(2)}</p>
                                    <p><span class="font-semibold">Longitude:</span> {features[0].geometry.coordinates[0].toFixed(2)}</p>
                                    {#each scenarios.filter(s => selectedScenarios.includes(s.id)) as s}
                                        <p>
                                            <span class="font-semibold">{s.name} intensity vmax (knots):</span>
                                            {features[0].properties[`vmax_kts_${s.id}`].toFixed(0)}
                                        </p>
                                    {/each}
                                </div>
                            </Popup>
                        </CircleLayer>
                    </GeoJSON>
                    {/if}
                </MapLibre>
                {/if}
            </div>
        </div>

        <div
            class="
                flex flex-row w-full transition-all px-4
                {showMore
                    ? 'min-h-[calc(100vh-7.5rem)]'
                    : 'h-0'
                }
            "
        >
            <div
                class="
                    {showMore ? 'flex' : 'hidden'}
                    gap-x-4 gap-y-6
                    flex-row flex-wrap items-center justify-around
                "
            >
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">VMPI</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'Maximum potential intensity from Kerry Emanuel equation (kt)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={vmpiPlot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">U200</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: '200 hPa zonal wind (kt*10) vs time (r=200-800 km)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={u200Plot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">SHRD</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: '850-200 hPa shear magnitude (kt*10) vs time (200-800 km)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={shrdPlot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">DELV</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'Last 6 hourly intensity change (kt)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={delvPlot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">RHLO</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: '850-700 hPa relative humidity (%) vs time (200-800 km)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={rhloPlot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">LP500</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'Land percentage within 500 kilometers at current position',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={lp500Plot} role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">PSLV</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'The observed zonal storm motion component (m/s*10) at position in 6 hours',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={pslvPlot}  role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">Latitude</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'Storm center latitude in degrees North',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={latPlot}   role="img" />
                </div>
                <div class="w-96 h-64 flex flex-col items-center">
                    <div class="flex flex-row items-center gap-x-2">
                        <span class="">Longitude</span>
                        <button
                            class="relative tooltip-container-bottom cursor-help"
                            style="
                                --tooltip-background-color: #fff;
                                --tooltip-color: #000;
                                --tooltip-offset-x: 10px;
                            "
                            on:click|preventDefault|stopPropagation
                            use:tooltip={{
                                content: 'Storm center longitude in degrees East (negative values for West of Prime Meridian)',
                                autoPosition: true,
                                hideOnClickOutside: true,
                                action: 'click',
                                position: 'bottom',
                            }}
                        >
                            <PhInfoDuotone />
                        </button>
                    </div>
                    <div class="flex-1 svg-container" bind:this={lonPlot}   role="img" />
                </div>
            </div>
        </div>

        <button
            class="
                flex flex-row w-full h-10 items-center justify-center gap-x-2 cursor-pointer
                hover:bg-white/20 active:bg-white/10 transition-all select-none
            "
            on:click={toggleShowMore}
        >
            {#if showMore}
            <PhCaretUpDuotone class="text-xl pb-px" />
            <p class="ml-px text-xs font-semibold tracking-widest">LESS</p>
            <PhCaretUpDuotone class="text-xl pb-px" />
            {:else}
            <PhCaretDownDuotone class="text-xl pb-px" />
            <p class="ml-px text-xs font-semibold tracking-widest">MORE</p>
            <PhCaretDownDuotone class="text-xl pb-px" />
            {/if}

        </button>

    </div>

</div>
{:else}
<div class="w-screen h-screen overflow-y-auto relative flex flex-col bg-black/60 backdrop-blur-sm">
    
    <div class="z-10 sticky top-0 flex flex-row text-white h-20 px-4 bg-black items-center justify-between shrink-0">
        <div class="flex-1 flex flex-row items-center justify-start">
            <img class="w-12 h-12 mr-4 rounded-full overflow-hidden" alt="" src="{base}/cyclone_bw.png" />
            <div class="flex flex-col">
                <h1 class="text-xl font-semibold leading-tight">
                    ICoM RAFT
                </h1>
                <h2 class="text-lg leading-tight">
                    Hurricane <span class="hidden sm:inline">Projections</span> Dashboard
                </h2>
            </div>
        </div>
    </div>

    <div class="w-full h-full flex flex-col items-center justify-start sm:justify-center gap-y-6 p-4">
        <h3
            class="text-3xl text-white font-semibold max-w-4xl leading-relaxed"
            style="text-shadow: 1px 1px 3px rgba(0,0,0,0.5);"
        >
            Type in the name, year, or ID for the storm you're interested in.
        </h3>
        <div class="w-full max-w-4xl">
            <Select
                id="storm"
                required
                searchable
                items={storms}
                bind:value={storm}
                placeholder="Search storms by name, year, or ID"
                --border-radius="0"
                --border-radius-focused="0"
                --item-first-border-radius="0"
                --list-border-radius="0"
                --height="64px"
                --item-color="black"
                --selected-item-color="black"
                --chevron-color="black"
                --clear-icon-color="red"
                --item-hover-color="black"
            >
                <div slot="prepend" class="mr-2 text-gray-600 text-3xl">
                    <CarbonTropicalStorm />
                </div>
                <div
                    slot="required"
                    class="hidden md:flex flex-row items-center justify-end text-gray-600 gap-x-2 pr-2"
                >
                    <span class="">
                        Try:
                    </span>
                    <button
                        class="rounded-sm px-2 py-1 bg-sky-700 hover:bg-sky-600 text-white shadow-sm shadow-black/25"
                        on:click={() => {
                            storm = storms.find(d => d.id == 2115);
                        }}
                    >
                        Irene (2011)
                    </button>
                    <button
                        class="rounded-sm px-2 py-1 bg-fuchsia-800 hover:bg-fuchsia-700 text-white shadow-sm shadow-black/25"
                        on:click={() => {
                            storm = storms.find(d => d.id == 2011);
                        }}
                    >
                        Katrina (2005)
                    </button>
                </div>
            </Select>
        </div>
        <div
            class="text-base text-white max-w-4xl text-justify"
            style="text-shadow: 1px 1px 3px rgba(0,0,0,0.25);"
        >
            <p class="mt-2">
                Welcome to the <span class="font-semibold">ICoM RAFT Hurricane Projections Dashboard</span>!
            </p>
            <p class="mt-2">
                This dashboard visualizes any of the <span class="font-semibold">620 historic tropical cyclones</span>
                from 1979 to 2018 replayed under eight different future climate scenarios, as described in the poster presentation:
                <a
                    class="italic text-blue-400 font-semibold hover:underline underline-offset-4"
                    target="_blank"
                    rel="noreferrer"
                    href="{base}/HydroML_2024_RAFT_hurricane_projections_poster.pdf"
                >
                    North Atlantic tropical cyclone intensity projections based on thermodynamic modifications of historic environments</a>.
                A paper and data download link will be available soon!
            </p>
            <p class="mt-2">
                When a storm is selected, the <span class="font-semibold">intensity plot</span> and <span class="font-semibold">map</span>
                display <span class="font-semibold">RAFT's deep learning baseline simulation</span> of the historic event. The intensity plot
                also includes a white line representing the observed intensities from the IBTrACS dataset. To assess the impact of future climate
                scenarios, select options from the menu in the top left corner. For a deeper dive into the changes for each scenario, click
                the "<span class="font-semibold">MORE</span>" section at the bottom which reveals how the model inputs are modified for the
                selected scenarios.
            </p>
        </div>
        <div class="w-full max-w-4xl p-2 flex flex-row flex-wrap mt-12 items-center justify-evenly gap-x-12 gap-y-8">
            <a class="h-12" href="https://icom.pnnl.gov/" target="_blank" rel="noreferrer">
                <img src={icomLogo} alt="ICOM" class="h-full object-contain" />
            </a>
            <a class="h-12" href="https://msdlive.org/" target="_blank" rel="noreferrer">
                <img src={msdLogo} alt="MSD-LIVE" class="h-full object-contain" />
            </a>
            <a class="h-24" href="https://pnnl.gov/" target="_blank" rel="noreferrer">
                <img src={pnnlLogo} alt="PNNL" class="h-full object-contain" />
            </a>
        </div>
    </div>

</div>
{/if}
