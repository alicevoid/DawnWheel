<script lang="ts">
	import { base } from '$app/paths';
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" type="image/x-icon" href="{base}/favicon.ico" />
</svelte:head>

{@render children()}
