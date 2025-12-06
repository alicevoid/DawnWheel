<script lang="ts">
	import faviconImg from '$lib/SMM_logo.ico';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" type="image/x-icon" href={faviconImg} />
</svelte:head>

{@render children()}
