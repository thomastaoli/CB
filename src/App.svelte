<script>
	// CORE IMPORTS
	import { setContext, onMount } from "svelte";
	import { getMotion } from "./utils.js";
	import { themes } from "./config.js";
	import ONSHeader from "./layout/ONSHeader.svelte";
	import ONSFooter from "./layout/ONSFooter.svelte";
	import Header from "./layout/Header.svelte";
	import Section from "./layout/Section.svelte";
	import Media from "./layout/Media.svelte";
	import Scroller from "./layout/Scroller.svelte";
	import Filler from "./layout/Filler.svelte";
	import Divider from "./layout/Divider.svelte";
	import Toggle from "./ui/Toggle.svelte";
	import Arrow from "./ui/Arrow.svelte";
	import Em from "./ui/Em.svelte";

	// DEMO-SPECIFIC IMPORTS
	import bbox from "@turf/bbox";
	import {
		getData,
		setColors,
		getTopo,
		getBreaks,
		getColor,
	} from "./utils.js";
	import { colors, units } from "./config.js";
	import {
		ScatterChart,
		LineChart,
		BarChart,
	} from "@onsvisual/svelte-charts";
	import {
		Map,
		MapSource,
		MapLayer,
		MapTooltip,
	} from "@onsvisual/svelte-maps";

	// CORE CONFIG (COLOUR THEMES)
	// Set theme globally (options are 'light', 'dark' or 'lightblue')
	let theme = "light";
	setContext("theme", theme);
	setColors(themes, theme);

	let jitterFactor = 0.01;

	// CONFIG FOR SCROLLER COMPONENTS
	// Config
	const threshold = 0.65;
	// State
	let animation = getMotion(); // Set animation preference depending on browser preference
	let id = {}; // Object to hold visible section IDs of Scroller components
	let idPrev = {}; // Object to keep track of previous IDs, to compare for changes
	onMount(() => {
		idPrev = { ...id };
	});

	// DEMO-SPECIFIC CONFIG
	// Constants
	const datasets = ["region", "district", "word"];
	const topojson = "./data/geo_lad2021.json";
	const mapstyle =
		"https://bothness.github.io/ons-basemaps/data/style-omt.json";
	const mapbounds = {
		uk: [
			[-9, 49],
			[2, 61],
		],
	};

	// Data
	let data = { district: {}, region: {}, word: {} };
	let metadata = { district: {}, region: {}, word: {} };
	let geojson;

	// Element bindings
	let map = null; // Bound to mapbox 'map' instance once initialised

	// State
	let hovered; // Hovered district (chart or map)
	let selected; // Selected district (chart or map)
	$: region =
		selected && metadata.district.lookup
			? metadata.district.lookup[selected].parent
			: null; // Gets region code for 'selected'
	$: chartHighlighted =
		metadata.district.array && region
			? metadata.district.array
					.filter((d) => d.parent == region)
					.map((d) => d.code)
			: []; // Array of district codes in 'region'
	let mapHighlighted = []; // Highlighted district (map only)
	let xKey = "board"; // xKey for scatter chart
	let yKey = null; // yKey for scatter chart
	let zKey = null; // zKey (color) for scatter chart
	let rKey = null; // rKey (radius) for scatter chart
	let mapKey = "population"; // Key for data to be displayed on map
	let explore = false; // Allows chart/map interactivity to be toggled on/off

	// FUNCTIONS (INCL. SCROLLER ACTIONS)

	// Functions for chart and map on:select and on:hover events
	function doSelect(e) {
		console.log(e);
		selected = e.detail.id;
		if (e.detail.feature) fitById(selected); // Fit map if select event comes from map
	}
	function doHover(e) {
		hovered = e.detail.id;
	}

	// Functions for map component
	function fitBounds(bounds) {
		if (map) {
			map.fitBounds(bounds, { animate: animation, padding: 30 });
		}
	}
	function fitById(id) {
		if (geojson && id) {
			let feature = geojson.features.find(
				(d) => d.properties.AREACD == id,
			);
			let bounds = bbox(feature.geometry);
			fitBounds(bounds);
		}
	}

	// Actions for Scroller components
	const actions = {
		map: {
			// Actions for <Scroller/> with id="map"
			map01: () => {
				// Action for <section/> with data-id="map01"
				fitBounds(mapbounds.uk);
				mapKey = "population";
				mapHighlighted = [];
				explore = false;
			},
			map02: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = false;
			},
			map03: () => {
				let hl = [...data.district.indicators].sort(
					(a, b) => b.age_med - a.age_med,
				)[0];
				fitById(hl.code);
				mapKey = "age_med";
				mapHighlighted = [hl.code];
				explore = false;
			},
			map04: () => {
				fitBounds(mapbounds.uk);
				mapKey = "age_med";
				mapHighlighted = [];
				explore = true;
			},
		},
		chart: {
			chart01: () => {
				xKey = "area";
				yKey = "board";
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart02: () => {
				xKey = "area";
				yKey = "board";
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart03: () => {
				xKey = "area";
				yKey = "population";
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart04: () => {
				xKey = "area";
				yKey = "board";
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart05: () => {
				xKey = "area";
				yKey = "board";
				zKey = null;
				rKey = null;
				explore = false;
			},
		},
	};

	// Code to run Scroller actions when new caption IDs come into view
	function runActions(codes = []) {
		codes.forEach((code) => {
			if (id[code] != idPrev[code]) {
				if (actions[code][id[code]]) {
					actions[code][id[code]]();
				}
				idPrev[code] = id[code];
			}
		});
	}
	$: id && runActions(Object.keys(actions)); // Run above code when 'id' object changes

	// INITIALISATION CODE
	datasets.forEach((geo) => {
		getData(`./data/data_${geo}.csv`).then((arr) => {
			let meta = arr.map((d) => ({
				name: d.name,
				area: d.area,
				board: d.board,
				population: d.population,
				density: d.density,
				age_med: d.age_med,
				times: d.times,
				word: d.word,
				noise: d.noise,
				parking: d.parking,
				heating: d.heating,
			}));
			let lookup = {};
			meta.forEach((d) => {
				lookup[d.code] = d;
			});
			metadata[geo].array = meta;
			metadata[geo].lookup = lookup;

			let indicators = arr.map((d, i) => ({
				...meta[i],
				area: d.area,
				board: d.board,
				population: d.population,
				density: d.density,
				age_med: d.age_med,
				times: d.times,
				word: d.word,
				noise: d.noise,
				parking: d.parking,
				heating: d.heating,
			}));

			if (geo == "district") {
				["density", "age_med"].forEach((key) => {
					let values = indicators
						.map((d) => d[key])
						.sort((a, b) => a - b);
					let breaks = getBreaks(values);
					indicators.forEach(
						(d, i) =>
							(indicators[i][key + "_color"] = getColor(
								d[key],
								breaks,
								colors.seq,
							)),
					);
				});
			}
			data[geo].indicators = indicators;

			let years = [2018, 2019, 2020, 2021, 2022, 2023];

			let timeseries = [];
			arr.forEach((d) => {
				years.forEach((year) => {
					timeseries.push({
						name: d.name,
						value: d[year],
						year,
					});
				});
			});
			data[geo].timeseries = timeseries;
		});
	});

	getTopo(topojson, "geog").then((geo) => {
		geo.features.sort((a, b) =>
			a.properties.AREANM.localeCompare(b.properties.AREANM),
		);
		geojson = geo;
	});

	function transformDataForBarChart(region) {
		return [
			{ category: "Noise", value: region.noise },
			{ category: "Heating", value: region.heating },
			{ category: "Parking", value: region.parking },
		];
	}

	function getColorForCategory(category) {
		const colorMap = {
			Noise: "#334ECD", // Example red color for Noise
			Heating: "#9F795F", // Example green color for Heating
			Parking: "#729F5F", // Example blue color for Parking
		};
		return colorMap[category] || "#dddddd"; // Fallback color
	}
</script>

<Header bgcolor="black" bgfixed={true} theme="dark" center={false} short={true}>
	<h1>How much do you know about your community board?</h1>
	<p class="text-big" style="margin-top: 5px">
		We examined 1000+ board meeting files. And here's what they are talking
		about.
	</p>
	<p style="margin-top: 20px">By Thomas Li</p>
	<p style="margin-top: 0px">Apr 20, 2024</p>
	<div style="margin-top: 90px;">
		<Arrow color="white" {animation}></Arrow>
	</div>
</Header>

<Section>
	<p>
		New York City, known for its vibrant diversity and sprawling boroughs,
		operates a unique system of local governance through its community
		boards. These boards play a crucial role in shaping the city's
		neighborhoods, yet many residents are unaware of their functions and the
		challenges they face.
	</p>
	<h2>This is a section title</h2>
	<p>
		This is a short paragraph of text to demonstrate the standard "medium"
		column width, font size and line spacing of the template.
	</p>
	<p>
		This is a second short paragraph of text to demonstrate the size of the
		paragraph spacing in the template.
	</p>
	<blockquote class="text-indent">
		"This is an example of a large embedded quotation."&mdash;A. Person
	</blockquote>
</Section>

<Divider />

<Section>
	<h2>Embedded charts or media</h2>
	<p>
		Below is an embedded chart. It is set to the same width as the column,
		"medium" (680px), but could also be "narrow" (540px), "wide" (980px) or
		"full" width. All options are responsive to fit the width of narrow
		screens.
	</p>
</Section>

{#if data.word.indicators}
	<Media col="medium">
		<div class="chart-sml">
			<BarChart
				data={[...data.word.indicators].sort(
					(b, a) => a.times - b.times,
				)}
				xKey="times"
				yKey="word"
				snapTicks={false}
				height={800}
				padding={{ top: 0, bottom: 15, left: 140, right: 0 }}
				area={false}
				title="The most mentioned topic of Manhattan CB1, from 2001 to 2023"
			/>
		</div>
	</Media>
{/if}

<Divider />

<Scroller {threshold} bind:id={id["chart"]} splitscreen={true}>
	<div slot="background">
		<figure>
			<div class="col-wide height-full">
				{#if data.district.indicators && metadata.region.lookup}
					<div class="chart">
						<ScatterChart
							height="calc(100vh - 150px)"
							data={data.district.indicators.map((d) => ({
								...d,
								parent_name:
									metadata.region.lookup[d.parent].name,
								[xKey]:
									d[xKey] *
									(1 + (Math.random() - 0.5) * jitterFactor), // Apply jitter here
							}))}
							colors={explore ? ["lightgrey"] : colors.cat}
							{xKey}
							{yKey}
							{zKey}
							{rKey}
							idKey="code"
							labelKey="name"
							r={[3, 10]}
							xScale="log"
							xTicks={[10, 100, 1000, 10000]}
							xFormatTick={(d) => d.toLocaleString()}
							xSuffix="null"
							yFormatTick={(d) => d.toLocaleString()}
							legend={zKey != null}
							labels
							select={explore}
							selected={explore ? selected : null}
							on:select={doSelect}
							hover
							{hovered}
							on:hover={doHover}
							highlighted={explore ? chartHighlighted : []}
							colorSelect="#206095"
							colorHighlight="#999"
							overlayFill
							{animation}
						/>
					</div>
				{/if}
			</div>
		</figure>
	</div>

	<div slot="foreground">
		<section data-id="chart01">
			<div class="col-medium">
				<p>
					This chart shows the <strong
						>area in square kilometres</strong
					> of each local authority district in the UK. Each circle represents
					one district. The scale is logarithmic.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the district.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong>density</strong> of the district
					in people per hectare.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong
						>part of the country</strong
					> that the district is within.
				</p>
			</div>
		</section>
		<section data-id="chart05">
			<div class="col-medium">
				<h3>Select a district</h3>
				<p>
					Use the selection box below or click on the chart to select
					a district. The chart will also highlight the other
					districts in the same part of the country.
				</p>
				{#if geojson}
					<p>
						<!-- svelte-ignore a11y-no-onchange -->
						<select bind:value={selected}>
							<option value={null}>Select one</option>
							{#each geojson.features as place}
								<option value={place.properties.AREACD}>
									{place.properties.AREANM}
								</option>
							{/each}
						</select>
					</p>
				{/if}
			</div>
		</section>
	</div>
</Scroller>

<Divider />

<Section>
	<h2>Do all the boards worry about the same issue though?</h2>
	<p>
		From 2018 to 2023, 311 received more than 180,000 complaints citywide.
		While the city has some issues - such as rats and illegal parking - in
		common, there's a significant difference when it comes to complaints
		from richer and poorer neighborhoods.
	</p>
</Section>

{#if data.region.indicators}
	<Media col="wide" grid="narrow" gap={20}>
		{#each data.region.indicators as region (region.code)}
			<div class="chart-sml">
				<BarChart
					data={transformDataForBarChart(region)}
					xKey="value"
					yKey="category"
					color={getColorForCategory}
					height={200}
					padding={{ top: 0, bottom: 20, left: 30, right: 15 }}
					title={region.name}
				/>
			</div>
		{/each}
	</Media>
{/if}

<Divider />

<Section>
	<h2>What do community boards discuss?</h2>
	<p>
		The chart below will respond to the captions as you scroll down. The
		"Scroller" component is set to "splitscreen" mode, which means the
		captions will be on the left side on larger screens.
	</p>
	<p>
		The interactions are via Javascript functions that are called when each
		caption scrolls into view.
	</p>
</Section>

<Section>
	<h2>This is a full-width chart demo</h2>
	<p>
		Below is an example of a media grid where the column with is set to
		"full". This allows for full width images and charts.
	</p>
	<p></p>
</Section>

<Media col="full" height={600}>
	<div class="chart-full">
		{#if data.district.timeseries}
			<LineChart
				data={data.district.timeseries}
				padding={{ left: 50, right: 150, top: 0, bottom: 0 }}
				height="500px"
				xKey="year"
				yKey="value"
				color="lightgrey"
				lineWidth={1}
				yFormatTick={(d) => (d / 1e6).toFixed(1)}
				ySuffix="m"
				select
				{selected}
				on:select={doSelect}
				hover
				{hovered}
				on:hover={doHover}
				highlighted={chartHighlighted}
				colorSelect="#206095"
				colorHighlight="#999"
				area={false}
				title="Mid-year population by district, 2001 to 2020"
				labels
				labelKey="name"
			/>
		{/if}
	</div>
</Media>

<Divider />

<Section>
	<h2>Methodology</h2>
	<p>
		Examining meeting records from all Community Boards in New York City
		presents a formidable challenge due to the significant inconsistency in
		data formats across the boards. This study specifically targeted
		Manhattan's Community Board 1 (CB1), renowned for its well-digitalized
		and comprehensive record-keeping. The initial step involved scraping the
		CB1 website to retrieve categorized files, which included meeting
		minutes, agendas, and voting records, all in PDF format.
	</p>
	<p>
		Subsequently, the files were converted into a text format using <strong
			>pdfminer</strong
		>, a tool adept at extracting text from PDF files. To structure and
		analyze the textual data effectively, the OpenAI API was employed with
		three distinct prompts designed to extract and synthesize key
		information:
	</p>
	<ul>
		<li>
			<strong>Key Terms</strong>: Prompted to "Return two key terms from
			the meeting minutes," aiming to highlight the primary topics of
			discussion.
		</li>
		<li>
			<strong>Compliance</strong>: Asked, "Did the meeting comply with the
			Open Meetings Law and NYC Charter?" This prompt required a simple
			"Yes" for compliance, or, if non-compliant, a brief explanation of
			the reasons.
		</li>
		<li>
			<strong>Interesting Aspects</strong>: Sought insights on "two key
			terms about anything noticed as interesting, unusual, or unexpected
			for a community board meeting," which helped identify unique or
			noteworthy elements within the meetings.
		</li>
	</ul>
	<p>
		This methodological approach transformed the collection of meeting
		documents into structured data, enabling a detailed analysis of the
		proceedings and adherence to legal standards by Manhattan CB1.
	</p>
</Section>

<ONSFooter />

<style>
	/* Styles specific to elements within the demo */
	:global(svelte-scroller-foreground) {
		pointer-events: none !important;
	}
	:global(svelte-scroller-foreground section div) {
		pointer-events: all !important;
	}
	select {
		max-width: 350px;
	}
	.chart {
		margin-top: 45px;
		width: calc(100% - 5px);
	}
	.chart-full {
		margin: 0 20px;
	}
	.chart-sml {
		font-size: 0.85em;
	}
	/* The properties below make the media DIVs grey, for visual purposes in demo */
	.media {
		background-color: #f0f0f0;
		display: -webkit-box;
		display: -ms-flexbox;
		display: flex;
		-webkit-box-orient: vertical;
		-webkit-box-direction: normal;
		-ms-flex-flow: column;
		flex-flow: column;
		-webkit-box-pack: center;
		-ms-flex-pack: center;
		justify-content: center;
		text-align: center;
		color: #aaa;
	}
</style>