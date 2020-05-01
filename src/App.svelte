<script>
import { onMount } from "svelte";
import { Aioli } from "@biowasm/aioli";
import Parameter from "./Parameter.svelte";


// -----------------------------------------------------------------------------
// Globals
// -----------------------------------------------------------------------------

// User input
let seq1 = "ACGTGCCCCACAGAT";
let seq2 = "AGGTGGACGAGAT";
let result = {
	sw: "Loading...",
	nw: "Loading..."
};

// Aioli setup
let ready = false;
let smith_waterman = new Aioli("seq-align/smith_waterman/2017.10.18");
let needleman_wunsch = new Aioli("seq-align/needleman_wunsch/2017.10.18");

let Params = [];
let Options = {
	// Scoring
	match: 2,
	mismatch: -2,
	gapopen: -1,
	gapextend: -1,
	// General
	case_sensitive: false,
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
}).join(" ").trim();

// Run alignment when user input changes
$: {
	if(ready)
	{
		// --printmatrices
		smith_waterman.exec(`${Params} ${seq1} ${seq2}`).then(d =>
		{
			result.sw = cleanOutput(d.stdout);
			if(d.stderr != "")
				console.warn(d.stderr);
		});

		needleman_wunsch.exec(`--printscores ${Params} ${seq1} ${seq2}`).then(d => {
			result.nw = cleanOutput(d.stdout);
			if(d.stderr != "")
				console.warn(d.stderr);
		});
	}
}

// -----------------------------------------------------------------------------
// Utility functions
// -----------------------------------------------------------------------------

function cleanOutput(output)
{
	// Remove lines that start with `==` and trim extra whitespace
	return output
		.split("\n")
		.filter(d => !d.startsWith("=="))
		.join("\n")
		.trim();
}

// -----------------------------------------------------------------------------
// On page load
// -----------------------------------------------------------------------------

onMount(async () => {
	// Initialize wasm tools
	Promise.all([
		smith_waterman.init(),
		needleman_wunsch.init()
	]).then(() => ready = true);

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
	<div class="jumbotron mt-2 mb-3 pb-4">
		<div class="container">
			<!-- Input sequences -->
			<p class="mb-2">Sequences to align:</p>
			<div class="row">
				<div class="col-12">
					<input type="text" bind:value={seq1} disabled={!ready} class="sequences form-control mb-1" style="font-family: monospace">
					<input type="text" bind:value={seq2} disabled={!ready} class="sequences form-control" style="font-family: monospace">
				</div>
			</div>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- CLI Parameters -->
			<div class="col-md-4">
				<h4 class="mb-3">Parameters</h4>
				<h6>Scores</h6>
				<Parameter label="Match" type="text" help="?" bind:value={Options.match} />
				<Parameter label="Mismatch" type="text" help="?" bind:value={Options.mismatch} />
				<Parameter label="Gap Open" type="text" help="?" bind:value={Options.gapopen} />
				<Parameter label="Gap Extend" type="text" help="?" bind:value={Options.gapextend} />
				<Parameter label="Scoring Matrix" type="dropdown" help="?" bind:value={Options.gapextend} />
				<br />

				<h6>General</h6>
				<Parameter label="Case Sensitive" type="checkbox" help="Enable case-sensitive alignment" bind:value={Options.case_sensitive} />

				<br />
			</div>

			<!-- Smith-Waterman alignment output -->
			<div class="col-md-4">
				<h4 class="mb-3">Smith-Waterman</h4>
				<pre>
				{result.sw}
				</pre>
			</div>

			<!-- Needleman-Wunsch alignment output -->
			<div class="col-md-4">
				<h4 class="mb-3">Needleman-Wunsch</h4>
				<pre>
				{result.nw}
				</pre>
			</div>
		</div>
	</div>
</main>
