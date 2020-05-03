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
// HTML
// -----------------------------------------------------------------------------
</script>

<style>
.sequences {
	font-family: monospace !important;
}
</style>

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
					<input type="text" bind:value={Seq1} disabled={!CLI.ready} class="sequences form-control mb-1" style="font-family: monospace">
					<input type="text" bind:value={Seq2} disabled={!CLI.ready} class="sequences form-control" style="font-family: monospace">
				</div>
				<div class="col-2 border-right">
					<Parameter label="Match" type="text" disabled={!CLI.ready} help="Score given to matching bases" bind:value={Options.match} />
					<Parameter label="Mismatch" type="text" disabled={!CLI.ready} help="Penalty for mismatches" bind:value={Options.mismatch} />
				</div>
				<div class="col-2 border-right">
					<Parameter col="6" label="Open" type="text" disabled={!CLI.ready} help="Penalty for starting a gap. Set to 0 to disable affine gap penalties." bind:value={Options.gapopen} />
					<Parameter col="6" label="Extend" type="text" disabled={!CLI.ready} help="Penalty for each base that extends an open gap." bind:value={Options.gapextend} />
				</div>
				<div class="col-2">
					<button on:click={launch} style="width:100%" disabled={!CLI.ready} class="btn btn-primary btn-lg pt-4 pb-4">Align</button>
				</div>
			</div>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- Smith-Waterman alignment output -->
			<div class="col-6">
				<h4 class="mb-3">Smith-Waterman</h4>
				<pre style="height: 30vh; border: 1px solid #ccc">
				{Result.sw}
				</pre>
				<div id="matrix-sw"></div>
			</div>

			<!-- Needleman-Wunsch alignment output -->
			<div class="col-6">
				<h4 class="mb-3">Needleman-Wunsch</h4>
				<pre style="height: 30vh; border: 1px solid #ccc">
				{Result.nw}
				</pre>
				<div id="matrix-nw"></div>
			</div>
		</div>
	</div>
</main>
