<script lang="ts">
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
	$: selectedFile = selectedBaseName == null ? null : filesByBaseName[selectedBaseName];
</script>

<button class="border border-black p-2 m-4" on:click={selectDirectory}
	>Click and select directory with Google Cloud Vision API output JSON files</button
>

{#if files.length}
	<div class="p-2 m-4 font-bold">Files loaded: {files.length}</div>
{/if}
{#each files as { baseName }}
	<button on:click={() => (selectedBaseName = baseName)} class="border border-black p-2 m-4"
		>{baseName}</button
	>
{/each}

{#if selectedFile != null}
	<FileComponent files={selectedFile} />
{/if}
