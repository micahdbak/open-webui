<script lang="ts">
	import { page } from '$app/stores';

	import Call from '$lib/components/call/Call.svelte';
</script>

<Call id={$page.params.id} />
