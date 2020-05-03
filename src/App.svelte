<script>
import { onMount } from "svelte";
import { Aioli } from "@biowasm/aioli";
import Parameter from "./Parameter.svelte";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

// User input
let Seq1 = "ACGTGCCCCACAGAT";
let Seq2 = "AGGTGGACGAGAT";
let Params = [];
let Options = {
	// Scoring
	match: 2,
	mismatch: -2,
	gapopen: -1,
	gapextend: -1,
	// General
	case_sensitive: false
};
// Aioli/WebAssembly setup
let CLI = {
	ready: false,
	sw: new Aioli("seq-align/smith_waterman/2017.10.18"),
	nw: new Aioli("seq-align/needleman_wunsch/2017.10.18")
}
// Output
let Result = {
	sw: "Loading...",
	nw: "Loading..."
};


// -----------------------------------------------------------------------------
// Reactive Statements
// -----------------------------------------------------------------------------

// Update CLI parameters when user updates settings
$: Params = Object.keys(Options).map(arg => {
	let value = Options[arg];
	if(value === null)
		return "";
	if(typeof value === "boolean")
		return value === true ? `--${arg}` : "";
	return `--${arg} ${value}`;
}).filter(d => d != "").join(" ").trim();

// Debugging
$: console.log(`Parameters: ${Params}`);


// -----------------------------------------------------------------------------
// On page load
// -----------------------------------------------------------------------------

onMount(async () => {
	// Initialize wasm tools
	Promise.all([ CLI.sw.init(), CLI.nw.init() ]).then(launch);

	// Enable jQuery tooltips
	jQuery("[data-toggle='popover']").popover();
});


// -----------------------------------------------------------------------------
// Launch alignments
// -----------------------------------------------------------------------------

function launch()
{
	CLI.ready = false;

	// Run Smith-Waterman algorithm
	let sw = CLI.sw
		.exec(`--printmatrices ${Params} ${Seq1} ${Seq2}`)
		.then(d => parseOutput(d, "sw"));

	// Run Needleman-Wunsch algorithm
	let nw = CLI.nw
		.exec(`--printmatrices --printscores ${Params} ${Seq1} ${Seq2}`)
		.then(d => parseOutput(d, "nw"));

	// Re-enable UI parameters once everything is done loading
	Promise.all([ sw, nw ]).then(() => CLI.ready = true);
}


// -----------------------------------------------------------------------------
// Utility functions
// -----------------------------------------------------------------------------

// Convert sequence to list of subscripted bases
// e.g. "ACATC" --> "A1, C1, A2, T1, C2"
function seqToArray(seq)
{
	let n = 1;
	let w = seq.split("").map(d => `${d}<sub>${n++}</sub>`);
	return [""].concat(w);
}

// Parse results and dynamic programming matrices from output
function parseOutput(out, algorithm)
{
	// Stop if errors
	if(out.stderr != "") {
		Result[algorithm] = d.stderr;
		return;		
	}

	// Extract DP matrices from stdout
	let result = "";    // seq-align output displayed to user (excluding matrices)
	let which = null;   // which matrix we're extracting
	let matrices = { all: [] };
	let stdout = out.stdout.split("\n");
	for(let line of stdout)
	{
		// Matrices in the output are prepended with their description
		if(line == "match_scores:") {
			which = "match";
		} else if(line == "gap_a_scores:") {
			which = "gap_a";
		} else if(line == "gap_b_scores:") {
			which = "gap_b";
		} else if(line.startsWith("match: ")) {
			which = null;
		// If we're at a line from a matrix, split the output into an array
		} else if(which != null) {
			if(!(which in matrices))
				matrices[which] = [];
			let values = line.split(":").pop().trim().split(/\s+/).map(d => +d);
			matrices[which].push(values);
		// Keep track of the non-DP output
		} else if(line.trim() != "" && !line.startsWith("==") && !line.startsWith("seq_a:") && !line.startsWith("seq_b:")) {
			result += `${line}\n`;
		}
	}
	Result[algorithm] = result;

	// Generate Plotly annotations for matrix
	let annotations = [];
	for(let rowNb in matrices.match)
	{
		matrices.all[rowNb] = [];
		for(let colNb in matrices.match[rowNb])
		{
			matrices.all[rowNb][colNb] = Math.max(
				matrices.match[rowNb][colNb],
				matrices.gap_a[rowNb][colNb],
				matrices.gap_b[rowNb][colNb],
			);
			annotations.push({
				x: colNb,
				y: rowNb,
				text: matrices.all[rowNb][colNb],
				showarrow: false
			})
		}
	}

	// Create or update plot
	Plotly.react(`matrix-${algorithm}`, [{
		x: seqToArray(Seq1),
		y: seqToArray(Seq2),
		z: matrices.all,
		type: "heatmap",
		showscale: false,
		hoverinfo: "skip",
		hovertemplate: '%{x}, %{y}<extra> %{z}</extra>',
	}], {
		margin: { t: 30, l: 30, r: 20, b: 5 },
		annotations: annotations,
		xaxis: { side: "top" },
		yaxis: { autorange: "reversed" },
	}, {
		displayModeBar: false
	});
}


// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

<nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
	<a class="navbar-brand" href="/">Alignment Sandbox</a>
	<div class="collapse navbar-collapse" id="navbarsExampleDefault">
		<ul class="navbar-nav mr-auto"></ul>
		<ul class="navbar-nav">
			<li class="nav-item active">
				<a class="nav-link" href="/">Code</a>
			</li>
		</ul>
	</div>
</nav>

<main role="main">
	<div class="jumbotron mt-2 mb-3 pb-3">
		<div class="container">
			<!-- Input Parameters -->
			<div class="row">
				<div class="col-6 border-right">
					<p><strong>Sequences to align:</strong></p>
				</div>
				<div class="col-2 border-right">
					<p><strong>Match Scores:</strong></p>
				</div>
				<div class="col-2 border-right">
					<p><strong>Gap Penalties:</strong></p>
				</div>
			</div>
			<div class="row">
				<div class="col-6 border-right">
					<Parameter col="0" disabled={!CLI.ready} on:launch={launch} bind:value={Seq1} />
					<Parameter col="0" disabled={!CLI.ready} on:launch={launch} bind:value={Seq2} />
				</div>
				<div class="col-2 border-right">
					<Parameter label="Match" disabled={!CLI.ready} on:launch={launch}  help="Score given to matching bases" bind:value={Options.match} />
					<Parameter label="Mismatch" disabled={!CLI.ready} on:launch={launch}  help="Penalty for mismatches" bind:value={Options.mismatch} />
				</div>
				<div class="col-2 border-right">
					<Parameter label="Open" disabled={!CLI.ready} on:launch={launch}  help="Penalty for starting a gap. Set to 0 to disable affine gap penalties." bind:value={Options.gapopen} />
					<Parameter label="Extend" disabled={!CLI.ready} on:launch={launch}  help="Penalty for each base that extends an open gap." bind:value={Options.gapextend} />
				</div>
				<div class="col-2">
					<button on:click={launch} style="width:100%" disabled={!CLI.ready} on:launch={launch}  class="btn btn-primary btn-lg pt-4 pb-4">Align</button>
				</div>
			</div>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- Smith-Waterman alignment output -->
			<div class="col-6">
				<h4 class="mb-3">Smith-Waterman</h4>
				<pre style="height: 10vh; border: 1px solid #ccc">
					{Result.sw}
				</pre>
				<div id="matrix-sw"></div>
			</div>

			<!-- Needleman-Wunsch alignment output -->
			<div class="col-6">
				<h4 class="mb-3">Needleman-Wunsch</h4>
				<pre style="height: 10vh; border: 1px solid #ccc">
					{Result.nw}
				</pre>
				<div id="matrix-nw"></div>
			</div>
		</div>
	</div>
</main>
