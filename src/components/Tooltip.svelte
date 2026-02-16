<script>
    export let data;
    import { fade } from "svelte/transition";

    $: props = data?.properties;
</script>

{#if props}
    <div transition:fade>
        <h3>{props.nombre}</h3>
        <p>{props.nota}</p>
    </div>
{/if}

<style>
    div {
        position: absolute;
        bottom: 30px;
        right: 0px;
        text-align: right;
    }

    h3 {
        font-size: 1.3rem;
        font-weight: 700;
        margin-bottom: 0.25rem;
    }
</style>
