<script>
import { createEventDispatcher } from "svelte";

export let label = "";
export let value = "";
export let disabled = false;
export let placeholder = "";
export let help = "";
export let col = 6;

let id = `param-${label}`;
let dispatch = createEventDispatcher();

// Execute command when press Enter
function handleKeydown(event)
{
	if(event.key == "Enter")
		dispatch("launch");
}
</script>


<div class="form-group row mb-2">
    {#if col != "0"}
        <label class="col-{col} col-form-label" for={id}>{label}</label>
        <div class="col-3 p-0">
            <div class="input-group input-group-sm">
                <input id={id} type="text" on:keydown={handleKeydown} class="form-control form-control-sm" bind:value={value} placeholder={placeholder} disabled={disabled}>
            </div>
        </div>

        <div class="col-1 p-0 pl-1">
            {#if help != ""}
            <button
                type="button"
                class="btn btn-sm btn-outline-primary"
                data-toggle="popover"
                data-trigger="hover"
                data-content="{help}"
                data-html="true"
            >?</button>
            {/if}
        </div>
    {:else}
        <div class="col-12">
            <input id={id} type="text" on:keydown={handleKeydown} class="form-control" bind:value={value} placeholder={placeholder} disabled={disabled}  style="font-family: monospace">
        </div>
    {/if}
</div>
