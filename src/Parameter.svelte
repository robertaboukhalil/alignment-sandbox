<script>
export let type = "text";
export let label = "";
export let prepend = "";
export let append = "";
export let value = "";
export let options = [];
export let disabled = false;
export let placeholder = "";
export let help = "";
export let col = 6;

$: id = `param-${type}-${label}`;
</script>


<div class="form-group row mb-2">
    <label class="col-{col} col-form-label" for={id}>{label}</label>

    <div class="col-3 p-0">
        {#if type == "text"}
            <div class="input-group input-group-sm">
                <input id={id} type="text" class="form-control form-control-sm" bind:value={value} placeholder={placeholder} disabled={disabled}>
            </div>

        {:else if type == "checkbox"}
            <div class="custom-control custom-control custom-checkbox mt-2">
                <input id={id} type="checkbox" bind:checked={value} class="custom-control-input" disabled={disabled}>
                <label class="custom-control-label" for={id}></label>
            </div>

        {:else if type == "dropdown"}
            <select class="custom-select custom-select-sm" bind:value={value}>
                {#each options as option}
                    <option value="{option}">{option == null ? "—" : option}</option>
                {/each}
            </select>
        {/if}
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
</div>
