<script>
import { onMount } from "svelte";
import { Aioli } from "@biowasm/aioli";

// User input
let seq1 = "ACGTGCCCCACAGAT";
let seq2 = "AGGTGGACGAGAT";
let result = "";

// Aioli setup
let ready = false;
let smith_waterman = new Aioli("seq-align/smith_waterman/2017.10.18");
let needleman_wunsch = new Aioli("seq-align/needleman_wunsch/2017.10.18");
onMount(async () => {
	Promise.all([
		smith_waterman.init(),
		needleman_wunsch.init()
	]).then(() => ready = true);
});

// Run alignment when user input changes
$: {
	if(ready)
	{
		smith_waterman.exec(`${seq1} ${seq2}`).then(d =>
		{
			result = d.stdout;
			if(d.stderr != "")
				console.warn(d.stderr);
		});
	}
}
</script>

{#if !ready}
Loading...<br /><br />
{/if}

<input type="text" bind:value={seq1} disabled={!ready}><br />
<input type="text" bind:value={seq2} disabled={!ready}>

<pre>
{result}
</pre>
