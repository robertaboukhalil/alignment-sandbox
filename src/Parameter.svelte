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

$: id = `param-${type}-${label}`;
</script>


<div class="form-group row mb-0">
    <label class="col-sm-6 col-form-label" for={id}>{label}</label>

    <div class="col-sm-4 p-0">
        {#if type == "text"}
            <div class="input-group input-group-sm">
                {#if prepend != ""}
                <div class="input-group-prepend">
                    <div class="input-group-text">{prepend}</div>
                </div>
                {/if}

                <input id={id} type="text" class="form-control form-control-sm" bind:value={value} placeholder={placeholder} disabled={disabled}>

                {#if append != ""}
                <div class="input-group-append">
                    <div class="input-group-text">{append}</div>
                </div>
                {/if}
            </div>

        {:else if type == "checkbox"}
            <div class="custom-control custom-control custom-checkbox mt-2">
                <input id={id} type="checkbox" bind:checked={value} class="custom-control-input" disabled={disabled}>
                <label class="custom-control-label" for={id}></label>
            </div>

        {:else if type == "dropdown"}
            <select class="custom-select custom-select-sm" bind:value={value}>
                {#each options as option}
                    <option value="{option}">{option == null ? "None" : option}</option>
                {/each}
            </select>
        {/if}
    </div>

    <div class="col-sm-1 p-0 pl-3">
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
