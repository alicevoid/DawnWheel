<script lang="ts">
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" type="image/x-icon" href="/favicon.ico" />
</svelte:head>

{@render children()}
