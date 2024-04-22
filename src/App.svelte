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
	const datasets = ["region", "district", "board", "word"];
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
	let data = { district: {}, region: {}, board: {}, word: {} };
	let metadata = { district: {}, region: {}, board: {}, word: {} };
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
	let xKey = "area"; // xKey for scatter chart
	let yKey = null; // yKey for scatter chart
	let zKey = null; // zKey (color) for scatter chart
	let rKey = null; // rKey (radius) for scatter chart
	let mapKey = "density"; // Key for data to be displayed on map
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
				mapKey = "density";
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
				yKey = null;
				zKey = null;
				rKey = null;
				explore = false;
			},
			chart02: () => {
				xKey = "area";
				yKey = null;
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart03: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = false;
			},
			chart04: () => {
				xKey = "area";
				yKey = "density";
				zKey = "parent_name";
				rKey = "pop";
				explore = false;
			},
			chart05: () => {
				xKey = "area";
				yKey = "density";
				zKey = null;
				rKey = "pop";
				explore = true;
			},
		},
	};

	let jitterFactor = 0.01;

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
				code: d.code,
				name: d.name,
				parent: d.parent ? d.parent : null,
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
				pop: d.pop,
				density: d.density,
				age_med: d.age_med,
				bname: d.bname,
				Noise: d.Noise,
				Heat: d.Heat,
				Parking: d.Parking,
				term: d.term,
				frequency: d.frequency,
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
						code: d.code,
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

	function transformDataForBarChart(board) {
		return [
			{ category: "Noise", value: board.Noise },
			{ category: "Heating", value: board.Heat },
			{ category: "Parking", value: board.Parking },
		];
	}

	function getColorForCategory(category) {
		const colorMap = {
			Noise: "#334ECD", // Example red color for Noise
			Heat: "#9F795F", // Example green color for Heating
			Parking: "#729F5F", // Example blue color for Parking
		};
		return colorMap[category] || "#dddddd"; // Fallback color
	}
</script>

<Header 
    style="background-image: url('/img/file.jpeg'); background-size: cover; background-position: center; background-attachment: fixed;" 
    theme="dark" 
    center={false} 
    short={true}>
    <h1>How much do you know about your community board?</h1>
    <p class="text-big" style="margin-top: 5px">
        Like many other grassroots organizations, the roles played by community
        boards are always ignored. With OpenAI, I examined 1000+ board meeting
        files. And here's what they are talking about.
    </p>
    <p style="margin-top: 20px">By <a href="https://www.linkedin.com/in/thomas-tao-li">Thomas Li</a></p>
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
	<p>
		The concept of Community Boards in New York City originated in the early
		1950s as part of an effort to involve citizens more directly in local
		governance. Officially, the current system was established under the
		1963 City Charter, which set the framework for local participatory
		democracy. Each of the city's 59 community boards consists of up to 50
		volunteer members appointed by the borough president, half of them on
		the recommendation of City Council members representing the district.
		These members serve staggered two-year terms and are responsible for
		advising on the use and development of land and facilities in their
		district.
	</p>
</Section>

<Divider />

<Section>
	<img src="img/cb5.jpeg" alt="Description" />
	<p class="caption">
		A monthly committee meeting of Queens CB5, which covers the
		neighborhoods of Glendale, Maspeth, Middle Village, and Ridgewood. Photo
		by Queens CB5.
	</p>
</Section>

<Section>
	<h2>What does community board do</h2>
	<p>
		The primary function of community boards is to act as a liaison between
		the citizens and the city government, ensuring that the community's
		voice is heard on important decisions. Their duties include reviewing
		and recommending action on issues concerning land use (such as rezoning
		and planning), assessing the needs of their neighborhoods annually, and
		working on other community concerns. They also have a say in budget
		priorities, pushing for funding on projects that address local needs.
	</p>
	<p>
		Community boards hold public hearings and meet monthly to address
		community concerns in a formal setting. The boards are pivotal in
		shaping policy at a local level but do not have the authority to enforce
		laws. This advisory role, however, does afford them significant
		influence, especially in areas related to public land and infrastructure
		projects.
	</p>
	<blockquote class="text-indent">
		"Each community board shall have the duties and responsibilities
		prescribed by this chapter, which shall include but not be limited to:
		considering the needs of the district which it serves."&mdash;Chapter
		70, Section 2800, New York City Charter
	</blockquote>
	<p>
		While the duty of community boards is undeniably important in shaping
		New York City's neighborhood policies and development, there are
		significant challenges concerning transparency and record-keeping. Many
		of their records are incomplete or missing, similar to other grassroots
		organizations, which can hinder accountability and public trust. To
		better understand what boards around the city is doing, I used the
		OpenAI API to analyze over 1,000 meeting records that I scraped from
		Manhattan's CB1.
	</p>
</Section>

<Divider />

{#if data.word.indicators}
	<Media col="medium" caption="Source: Manhattan CB1 Meeting Minutes">
		<div class="chart-sml">
			<BarChart
				data={[...data.word.indicators].sort(
					(a, b) => a.frequency - b.frequency,
				)}
				xKey="frequency"
				yKey="term"
				snapTicks={false}
				xTicks={[5, 10, 15, 20, 25, 30]}
				height={500}
				padding={{ top: 0, bottom: 15, left: 140, right: 0 }}
				area={false}
				title="Most discussed terms in Manhattan CB1 meeting minutes, 2003-2023"
			/>
		</div>
	</Media>
{/if}

<Divider />
<Section>
	<p>
		Yes, one of community boards' most important jobs is the regulation of
		liquor licenses - which is a direct response to the impacts that
		establishments selling alcohol can have on community safety and quality
		of life. These establishments must apply for a liquor license through
		the New York State Liquor Authority (SLA), but as part of the process,
		they must also present their applications at community board meetings.
	</p>
	<p>
		The boards' recommendations can significantly influence the SLA's
		decisions to grant, modify, or deny licenses, based on potential
		neighborhood impacts like noise and traffic. This process, as mandated
		by the City Charter, ensures active community participation in such
		regulatory matters, safeguarding neighborhood character and residential
		life.
	</p>
</Section>

<Section>
	<h2>Oversights, also oddities</h2>
	<p>
		Like other forms of local governance, community boards in New York City
		occasionally encounter ethical challenges and allegations of misconduct.
		These instances often involve conflicts of interest, where board members
		may have personal or financial stakes in the projects they are
		reviewing.
	</p>
	<p>
		Last month, two members of Manhattan CB5 suddenly resigned after
		tensions between the board because of “special interest” influence.
		Layla Law-Gisiko, one of the resigning members, said that "the presence
		of board members who work or are affiliated with an organization that
		lobbies and represents special interest is very disturbing."
	</p>
	<p>
		Apart from routine discussions on land use and zoning - which oftentimes
		bring out the most heated debates - Community Boards sometimes sometimes
		engage in debates over less typical issues that reflect the unique
		character of their neighborhoods. The CB1 files I examined - for
		example, routinely propose all sorts of parades and street fairs,
		including a petting zoo at Fulton Stall Market and a Ticker Tape Parade.
		Besides, the board also sometimes discusses and endorses local and
		international political agendas, such as supporting New York's Same-Sex
		Marriage Act and Chinese dissidents in the community who practice Falun
		Gong.
	</p>
</Section>

<Divider />

<Section>
	<img src="img/Laughing-Man.jpg" alt="Description" />
	<p class="caption">
		Hugh Jackman's Laughing Man Cafe - which is a popular spot in the
		Tribeca neighborhood - is also an evergreen topic in the CB1 meeting
		minutes. The board often discusses the cafe's outdoor seats. Photo by
		Tribeca Citizen.
	</p>
</Section>

<Divider />

<Section>
	<h2>A reflection of NYC's wealth gaps</h2>
	<p>
		Community Boards in New York City serve as a mirror to the larger
		socio-economic disparities prevalent across different neighborhoods.
		While these boards are designed to provide equal representation and
		influence, the reality often showcases a stark contrast in resources and
		outcomes, closely tied to the economic profiles of the areas they
		represent.
	</p>
</Section>

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
							}))}
							colors={explore ? ["lightgrey"] : colors.cat}
							{xKey}
							{yKey}
							{zKey}
							{rKey}
							idKey="code"
							labelKey="name"
							r={[3, 10]}
							xScale="linear"
							xTicks={[30000, 60000, 90000, 120000, 150000]}
							xFormatTick={(d) => d.toLocaleString()}
							xSuffix=" $"
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
						>medium household incomes</strong
					> of each community board in New York City. While Manhattan CB
					1&2 - which located in the downtown financial district - have
					the highest incomes, the Bronx CB 7 for Kingsbridge and Riverdale
					has the lowest.
				</p>
			</div>
		</section>
		<section data-id="chart02">
			<div class="col-medium">
				<p>
					The radius of each circle shows the <strong
						>total population</strong
					> of the board. Queens CB 12, which includes Jamaica and Hollis,
					has the highest population.
				</p>
			</div>
		</section>
		<section data-id="chart03">
			<div class="col-medium">
				<p>
					The vertical axis shows the <strong
						>per centage below poverty</strong
					> of the board. While many Bronx boards have high poverty rates,
					Manhattan shows a more mixed picture - with CB 3 in the Lower
					East Side and CB 10&11 in Harlem having the highest rate.
				</p>
			</div>
		</section>
		<section data-id="chart04">
			<div class="col-medium">
				<p>
					The colour of each circle shows the <strong>borough</strong>
					of each board. Generally, Queens has the lowest income gaps.
				</p>
			</div>
		</section>
	</div>
</Scroller>

<Divider />

<Section>
	<p>
		In wealthier districts, such as those in Manhattan, Community Boards
		tend to have access to a broader range of resources. This includes
		better funding, more active participation from residents who often have
		flexible schedules, and greater influence on city decisions. Their
		influence is further bolstered by higher rates of resident contributions
		and advocacy, ensuring their voices are heard loud and clear in the
		corridors of power.
	</p>
	<p>
		Contrast this with community boards in less affluent neighborhoods, such
		as parts of the Bronx or Brooklyn, where resources are markedly scarcer.
		Boards like those in the Bronx often grapple with more pressing
		socio-economic issues such as housing quality, safety, and access to
		basic services, which can overshadow their ability to lobby for or
		implement broader developmental agendas. Limited volunteer participation
		and the challenge of balancing jobs with board duties can also diminish
		the effectiveness of these boards, making it harder for them to advocate
		effectively for their communities.
	</p>
	<p>
		The disparities between wealthier and less affluent Community Boards in
		New York City can also be discerned through the volume and nature of 311
		complaints, the city's non-emergency helpline. Typically, wealthier
		areas tend to report fewer issues related to basic services such as
		sanitation, noise, and infrastructure, reflecting higher levels of
		maintenance and service delivery.
	</p>
</Section>

<Media
	col="full"
	height={600}
	caption="The chart below shows the number of 311 complaints in New York City by community board."
>
	<div class="chart-full">
		{#if data.district.timeseries}
			<LineChart
				data={data.district.timeseries}
				padding={{ left: 50, right: 150, top: 0, bottom: 0 }}
				height="500px"
				xKey="year"
				yKey="value"
				zKey="name"
				color="lightgrey"
				lineWidth={1}
				yTicks={[20000, 50000, 80000, 110000, 140000]}
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
				title="311 Complaints by Community Board, 2018-2023"
				labels
				labelKey="name"
			/>
		{/if}
	</div>
</Media>
<Divider />
<Section>
	<p>
		Conversely, in lower-income neighborhoods, 311 complaints often
		highlight persistent problems with trash collection, building
		violations, and utility failures, indicating a lag in self-governance
		and upkeep. This variation not only underscores the differences in
		living conditions across different districts but also suggests a
		disparity in how swiftly and effectively city services are mobilized and
		issues are addressed, often correlating with the economic and social
		capital of the communities involved.
	</p>
</Section>
<Divider />
<Section>
	<blockquote class="text-indent">
		Read More: <a
			href="https://docs.google.com/document/d/1HZXw-5Haue0BeBP7QcY48BYWLkx6azmRS4Q4jHSlqzU/edit?usp=sharing"
			>A tale of two cities within NYC: How wealth gaps determine 311
			response time</a
		> by Thomas Li and Charlene Lin
	</blockquote>
</Section>
<Divider />
<Section>
	<p>
		While CB1 in the financial district talks about the noise from
		helicopters and yachts, residents of Bronx are struggling to keep warm
		in New York's chilly winter. The number of 311 complaints on heating and
		hot water alone exceeded 225,000 from 2018-2023, which is five times the
		number of leading complaints from the wealthiest residents.
	</p>
</Section>
<Divider />
{#if data.board.indicators}
    <Media
        col="wide"
        grid="narrow"
        gap={15}
        title="Community Board Complaints Analysis"
        caption="Percentage of top 3 most 311-complained categories by community board, 2018-2023"
    >
        {#each data.board.indicators as board (board.bname)}
            <div class="chart-sml">
                <BarChart
                    data={transformDataForBarChart(board)}
                    xKey="value"
                    yKey="category"
                    color={getColorForCategory}
                    height={200}
                    padding={{ top: 0, bottom: 20, left: 30, right: 15 }}
                    title={board.bname}
                />
            </div>
        {/each}
    </Media>
{/if}
<Divider />
<Section>
	<p>
		Moreover, the disparity is evident in the ability of different boards to
		leverage city services. In wealthier neighborhoods, community boards are
		often more successful in attracting city funding and projects, from park
		renovations to road improvements. In contrast, boards in poorer
		districts sometimes struggle to get timely responses to basic needs,
		further widening the gap between the city's haves and have-nots.
	</p>
</Section>
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
<Divider />
<Section>
	<h2>Acknolwedgements</h2>
	<p>
		I'd like to thank <a
		href="https://www.linkedin.com/in/maxcresnik/"
		>Max Resnik</a> from City Bureau for his inspiration on this project, and help from <a
		href="https://www.linkedin.com/in/laura-bejder-jensen-812536171/"
		>Laura Jensen</a>, <a
		href="https://www.linkedin.com/in/matt-zinman-480315153/"
		>Matt Zinman</a> and <a
		href="https://www.linkedin.com/in/darryl-laiu/"
		>Darryl Laiu</a>. 
		We started this experiment during the OpenAI Hackathon.
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
