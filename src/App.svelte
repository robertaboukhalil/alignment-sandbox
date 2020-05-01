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
	case_sensitive: false,
	scoring: null
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

// Run alignment when user input changes
$: {
	if(CLI.ready)
	{
		// --printmatrices
		CLI.sw.exec(`${Params} ${Seq1} ${Seq2}`).then(d =>
		{
			Result.sw = cleanOutput(d.stdout);
			if(d.stderr != "")
				Result.sw = d.stderr;
		});

		CLI.nw.exec(`--printscores ${Params} ${Seq1} ${Seq2}`).then(d => {
			Result.nw = cleanOutput(d.stdout);
			if(d.stderr != "")
				Result.nw = d.stderr;
		});
	}
}

// -----------------------------------------------------------------------------
// Utility functions
// -----------------------------------------------------------------------------

function cleanOutput(output) {
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
		CLI.sw.init(),
		CLI.nw.init()
	]).then(() => CLI.ready = true);

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
					<input type="text" bind:value={Seq1} disabled={!CLI.ready} class="sequences form-control mb-1" style="font-family: monospace">
					<input type="text" bind:value={Seq2} disabled={!CLI.ready} class="sequences form-control" style="font-family: monospace">
				</div>
			</div>
		</div>
	</div>

	<div class="container">
		<div class="row">
			<!-- CLI Parameters -->
			<div class="col-md-4">
				<h4 class="mb-3">Parameters</h4>

				<h6>General</h6>
				<Parameter label="Case Sensitive" type="checkbox" help="Enable case-sensitive alignment" bind:value={Options.case_sensitive} />
				<br />

				<h6>Scores <small><a href="https://en.wikipedia.org/wiki/Gap_penalty" target="_blank">(learn more)</a></small></h6>
				<Parameter label="Match" type="text" help="Score given to matching bases" bind:value={Options.match} />
				<Parameter label="Mismatch" type="text" help="Penalty for mismatches" bind:value={Options.mismatch} />
				<Parameter label="Gap Open" type="text" help="Penalty for starting a gap" bind:value={Options.gapopen} />
				<Parameter label="Gap Extend" type="text" help="Penalty for each base that extends an open gap. If Gap Extend == Gap Open, this is referred to as a <strong>Linear Gap Penalty</strong>, otherwise it's called an <strong>Affine Gap Penalty</strong>." bind:value={Options.gapextend} />
				<Parameter label="Scoring Matrix" type="dropdown" options={[null, "PAM30", "PAM70", "BLOSUM80", "BLOSUM62"]} help="Commonly used substitution matrices for protein sequences" bind:value={Options.scoring} />
			</div>

			<!-- Smith-Waterman alignment output -->
			<div class="col-md-4">
				<h4 class="mb-3">Smith-Waterman</h4>
				<pre>
				{Result.sw}
				</pre>
			</div>

			<!-- Needleman-Wunsch alignment output -->
			<div class="col-md-4">
				<h4 class="mb-3">Needleman-Wunsch</h4>
				<pre>
				{Result.nw}
				</pre>
			</div>
		</div>
	</div>
</main>
