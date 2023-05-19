<script lang="ts">
	import { onMount, tick } from 'svelte';
	import FileComponent from '../components/File.svelte';

	let filesByBaseName: Record<string, File[]> = {};
	$: files = Object.entries(filesByBaseName).map(([baseName, docs]) => {
		return {
			baseName,
			docs
		};
	});

	async function selectDirectory() {
		const dir = await showDirectoryPicker({ mode: 'read' });

		for await (const file of dir.keys()) {
			const fileHandle = await dir.getFileHandle(file);
			const fileContents = await fileHandle.getFile();
			const baseName = file.split('-')[0];
			if (!filesByBaseName[baseName]) {
				filesByBaseName[baseName] = [];
			}
			filesByBaseName[baseName].push(fileContents);
		}
		filesByBaseName = filesByBaseName;
	}

	let selectedBaseName: string | null = null;
	let urlParam: string | null = null;
	$: selectedFile =
		urlParam != null
			? urlParam
			: selectedBaseName == null
			? null
			: filesByBaseName[selectedBaseName];

	let hideInterface = false;
	onMount(async () => {
		// Get URL query param
		hideInterface = true;
		await tick();
		const url = new URL(window.location.href);
		const urlValue = url.searchParams.get('url');
		if (urlValue != null) {
			urlParam = urlValue;
		}
	});
</script>

{#if !hideInterface}
	<button class="border border-black p-2 m-4" on:click={selectDirectory}
		>Click and select directory with Google Cloud Vision API output JSON files</button
	>
	(or append <code>?url=&lt;/url&gt;</code> to the URL)

	{#if files.length}
		<div class="p-2 m-4 font-bold">Files loaded: {files.length}</div>
	{/if}
	{#each files as { baseName }}
		<button on:click={() => (selectedBaseName = baseName)} class="border border-black p-2 m-4"
			>{baseName}</button
		>
	{/each}
{/if}

{#if selectedFile != null}
	<FileComponent files={selectedFile} />
{/if}
