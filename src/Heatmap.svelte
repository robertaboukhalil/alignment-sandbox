<script>
import { onMount } from "svelte";

export let algorithm = null;  // Accepted: "sw" (Smith-Waterman) or "nw" (Needleman-Wunsch)
export let seq1 = "";
export let seq2 = "";
export let matrix = [];
export let pointers = [];


// =============================================================================
// Reactive statements
// =============================================================================

// Plot heatmap when matrix is defined
$: if(matrix.length > 0) {
    plot();
}


// =============================================================================
// Utility functions
// =============================================================================

// Convert sequence to list of subscripted bases
// e.g. "ACATC" --> "A1, C1, A2, T1, C2"
function seqToArray(seq)
{
	let n = 1;
	let w = seq.split("").map(d => `${d}<sub>${n++}</sub>`);
	return [""].concat(w);
}


// =============================================================================
// Calculate backtrack from the dynamic programming matrix
// =============================================================================

function getBacktrack()
{
    // -------------------------------------------------------------------------
    // Calculate start coordinates of backtrack
    // -------------------------------------------------------------------------
    let rowNb = -1;
    let colNb = -1;

    // Needleman-Wunsch is a global alignment algorithm so start at bottom right corner
    if(algorithm == "nw") {
        rowNb = matrix.length - 1;
        colNb = matrix[0].length - 1;
    }

    // Smith-Waterman is a local alignment algorithm so start at position with max score
    else if(algorithm == "sw")
    {
        // Find largest number in the matrix
        // Note: use "..." since Math.max() doesn't support array inputs
        let max = Math.max(...matrix.map(row => Math.max(...row)));
        // Find coordinates of that max
        while(colNb == -1)
            colNb = matrix[++rowNb].indexOf(max);
    }

    // -------------------------------------------------------------------------
    // Do the backtrack
    // -------------------------------------------------------------------------
    let backtrack = [];
    // For global alignment, we backtrack until we get to (0, 0)
    while(rowNb >= 0 && colNb >= 0)
    {
        // Save coordinates as a linear array to simplify comparisons later on
        backtrack.push(matrix[0].length * rowNb + colNb);

        // For local alignment, we stop backtracking when we reach a value of 0
        if(algorithm == "sw" && matrix[rowNb][colNb] == 0)
            break;

        // Where to go next?
        //       || j - 1 |  j
        // ======||=======|=====
        // i - 1 ||   0   |  1
        // ------||-------|-----
        //   i   ||   2   |  *
        let whichDirection = pointers[rowNb][colNb];
        if(whichDirection == 0) {
            rowNb -= 1; colNb -=1;
        } else if(whichDirection == 1) {
            rowNb -= 1;
        } else if(whichDirection == 2) {
            colNb -=1;
        }
    }

    return backtrack;
}


// =============================================================================
// Plot heatmap
// =============================================================================

function plot()
{
    // Generate Plotly annotations
    let n = 0;
    let annotations = [];
    let backtrack = getBacktrack();
    for(let rowNb in matrix)
    {
        for(let colNb in matrix[rowNb])
        {
            let bordercolor = false;
            if(backtrack.includes(n))
                bordercolor = "white";
            annotations.push({
                x: colNb,
                y: rowNb,
                text: matrix[rowNb][colNb],
                showarrow: false,
                bordercolor: bordercolor,
                borderwidth: 3,
                borderpad: 5
            })
            n += 1;
        }
    }

    // Create or update plot
    Plotly.react(`heatmap-${algorithm}`, [{
        x: seqToArray(seq1),
        y: seqToArray(seq2),
        z: matrix,
        type: "heatmap",
        showscale: false,
        hoverinfo: "skip",
        hovertemplate: "%{x}, %{y}<extra> %{z}</extra>",
    }], {
        margin: { t: 30, l: 30, r: 20, b: 5 },
        annotations: annotations,
        xaxis: { side: "top" },
        yaxis: { autorange: "reversed" },
    }, {
        displayModeBar: false
    });
}
</script>

<style>
.heatmap {
	max-height: 50vh;
}

@media screen and (max-height: 500px) {
    .heatmap {
        max-height: 400px;
    }
}
</style>

<div class="heatmap" id="heatmap-{algorithm}"></div>
