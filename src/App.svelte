<script>
import { onMount } from "svelte";
import Aioli from "@biowasm/aioli";
import Parameter from "./Parameter.svelte";
import Heatmap from "./Heatmap.svelte";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

// User input
let Seq1 = "GGTTGACTA";
let Seq2 = "TGTTCGG";
let Params = [];
let Options = {
	match: 3,
	mismatch: -3,
	gapopen: 0,
	gapextend: -2,
};
// Aioli/WebAssembly setup
let ready = false;
let CLI;
// Output
let Result = {
	sw: "Loading...",
	nw: "Loading..."
};
let Matrix = {
	sw: [],
	nw: []
};
let Pointers = {
	sw: [],
	nw: []
}


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
	CLI = await new Aioli([
		"seq-align/smith_waterman/2017.10.18",
		"seq-align/needleman_wunsch/2017.10.18"
	], { printInterleaved: false })
	launch();

	// Enable jQuery tooltips
	jQuery("[data-toggle='popover']").popover();
});


// -----------------------------------------------------------------------------
// Launch alignments
// -----------------------------------------------------------------------------

async function launch()
{
	ready = false;

	// Run Smith-Waterman algorithm
	await CLI
		.exec(`smith_waterman --printmatrices ${Params} ${Seq1} ${Seq2}`)
		.then(d => parseOutput(d, "sw"));

	// Run Needleman-Wunsch algorithm
	await CLI
		.exec(`needleman_wunsch --printmatrices --printscores ${Params} ${Seq1} ${Seq2}`)
		.then(d => parseOutput(d, "nw"));

	// Re-enable UI parameters once everything is done loading
	ready = true;
}


// -----------------------------------------------------------------------------
// Utility functions
// -----------------------------------------------------------------------------

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
	let matrices = {};  // all three matrices
	let matrix = [];    // the matrix we get by combining info from all three matrices
	let pointers = [];  // matrix to help us backtrack
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

	// Generate a single DP matrix from the three matrices
	for(let rowNb in matrices.match) {
		matrix[rowNb] = [];
		pointers[rowNb] = [];
		for(let colNb in matrices.match[rowNb])
		{
			// Track max value
			matrix[rowNb][colNb] = Math.max(
				matrices.match[rowNb][colNb],
				matrices.gap_a[rowNb][colNb],
				matrices.gap_b[rowNb][colNb],
			);
			// But also whether it was a match, insertion or deletion to help us backtrack later
			pointers[rowNb][colNb] = [ matrices.match[rowNb][colNb],
				matrices.gap_a[rowNb][colNb],
				matrices.gap_b[rowNb][colNb]
			].indexOf(matrix[rowNb][colNb]);
		}
	}

	Matrix[algorithm] = matrix;
	Pointers[algorithm] = pointers;
}


// -----------------------------------------------------------------------------
// HTML
// -----------------------------------------------------------------------------
</script>

<style>
pre {
	height: 10vh;
	border: 1px solid #ccc;
}

@media screen and (max-width: 800px) {
	pre {
		height: 100px;
	}
}
</style>

<nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
	<a class="navbar-brand" href="/">Alignment Sandbox</a>
	<div class="collapse navbar-collapse">
		<ul class="navbar-nav mr-auto"></ul>
		<ul class="navbar-nav">
			<li class="nav-item active">
				<a class="nav-link" href="https://github.com/robertaboukhalil/alignment-sandbox">Code</a>
			</li>
		</ul>
	</div>
</nav>

<main>
	<div class="jumbotron mb-3 pb-2">
		<div class="container">
			<!-- Input Parameters -->
			<div class="row mt-4 mt-sm-0">
				<div class="col-12 col-sm-6 mt-2 mt-sm-0 border-right">
					<p><strong>Sequences to align:</strong></p>
					<Parameter col="0" disabled={!ready} on:launch={launch} bind:value={Seq1} />
					<Parameter col="0" disabled={!ready} on:launch={launch} bind:value={Seq2} />
				</div>
				<div class="col-12 col-sm-6 col-md-2 mt-2 mt-sm-0 border-right">
					<p><strong>Match Scores:</strong></p>
					<Parameter label="Match" disabled={!ready} on:launch={launch}  help="Score given to matching bases" bind:value={Options.match} />
					<Parameter label="Mismatch" disabled={!ready} on:launch={launch}  help="Penalty for mismatches" bind:value={Options.mismatch} />
				</div>
				<div class="col-12 col-sm-6 col-md-2 mt-2 mt-sm-0 border-right">
					<p><strong>Gap Penalties:</strong></p>
					<Parameter label="Open" disabled={!ready} on:launch={launch}  help="Penalty for starting a gap. Set to 0 to disable affine gap penalties." bind:value={Options.gapopen} />
					<Parameter label="Extend" disabled={!ready} on:launch={launch}  help="Penalty for each base that extends an open gap." bind:value={Options.gapextend} />
				</div>
				<div class="col-12 col-sm-6 col-md-2 mt-2 mt-sm-0 mb-2 mb-sm-0">
					<p class="d-none d-sm-block">&nbsp;</p>
					<button on:click={launch} style="width:100%" disabled={!ready} on:launch={launch}  class="btn btn-primary btn-lg pt-4 pb-4">Align</button>
				</div>
			</div>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- Smith-Waterman alignment output -->
			<div class="col-md-6 mb-2">
				<h4 class="mb-3">Smith-Waterman</h4>
				<pre>{Result.sw}</pre>
				<Heatmap algorithm="sw" seq1={Seq1} seq2={Seq2} matrix={Matrix.sw} pointers={Pointers.sw} />
			</div>

			<!-- Needleman-Wunsch alignment output -->
			<div class="col-md-6 mb-2">
				<h4 class="mb-3">Needleman-Wunsch</h4>
				<pre>{Result.nw}</pre>
				<Heatmap algorithm="nw" seq1={Seq1} seq2={Seq2} matrix={Matrix.nw} pointers={Pointers.nw} />
			</div>
		</div>
	</div>
</main>
