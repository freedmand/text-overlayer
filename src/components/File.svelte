<script lang="ts">
	import { onMount } from 'svelte';
	import Page from '../routes/+page.svelte';

	export let files: File[];

	interface Block {
		layout: Layout;
	}

	interface Point {
		x: number;
		y: number;
	}

	interface Segment {
		startIndex: string;
		endIndex: string;
	}

	interface Layout {
		boundingPoly: {
			normalizedVertices: Point[];
		};
		confidence: number;
		orientation: string;
		textAnchor: {
			textSegments: Segment[];
		};
	}

	interface Page {
		dimension: { width: number; height: number; unit: string };
		pageNumber: number;
		layout: Layout;
		blocks: Block[];
		lines: Block[];
		paragraphs: Block[];
		tokens: Block[];
		text: string;
		image: {
			content: string;
			width: number;
			height: number;
			mimeType: string;
		};
	}

	function getB64Image(page: Page): string {
		const { image } = page;
		return `data:${image.mimeType};base64,${image.content}`;
	}

	interface Doc {
		pages: Page[];
		shardInfo: { shardCount: string; shardIndex: string; textOffset: string };
		text: string;
		uri: string;
	}

	let jsonData: Doc[] = [];

	$: orderedPages = extractOrderedPages(jsonData);

	function getLeft(layout: Layout): number {
		let minX: number | null = null;
		for (const vertex of layout.boundingPoly.normalizedVertices) {
			if (minX == null || vertex.x < minX) {
				minX = vertex.x;
			}
		}
		return minX!;
	}

	function getTop(layout: Layout): number {
		let minY: number | null = null;
		for (const vertex of layout.boundingPoly.normalizedVertices) {
			if (minY == null || vertex.y < minY) {
				minY = vertex.y;
			}
		}
		return minY!;
	}

	function getRight(layout: Layout): number {
		let maxX: number | null = null;
		for (const vertex of layout.boundingPoly.normalizedVertices) {
			if (maxX == null || vertex.x > maxX) {
				maxX = vertex.x;
			}
		}
		return maxX!;
	}

	function getBottom(layout: Layout): number {
		let maxY: number | null = null;
		for (const vertex of layout.boundingPoly.normalizedVertices) {
			if (maxY == null || vertex.y > maxY) {
				maxY = vertex.y;
			}
		}
		return maxY!;
	}

	function extractOrderedPages(docs: Doc[]): Page[] {
		const pages: Page[] = [];
		for (const doc of docs) {
			for (const page of doc.pages) {
				page.text = doc.text;
				pages[page.pageNumber - 1] = page;
			}
		}
		console.log({ pages });
		return pages;
	}

	function getText(page: Page, layout: Layout) {
		const { textAnchor } = layout;
		const { textSegments } = textAnchor;
		const { startIndex, endIndex } = textSegments[0];
		return page.text.substring(Number(startIndex), Number(endIndex));
	}

	async function readFilesAsJSON(files: File[]) {
		const fileReadPromises = files.map((file) => {
			return new Promise((resolve, reject) => {
				const reader = new FileReader();

				reader.onload = () => {
					try {
						const json = JSON.parse(reader.result as string);
						resolve(json);
					} catch (error) {
						reject(error);
					}
				};

				reader.onerror = () => {
					reject(reader.error);
				};

				reader.readAsText(file);
			});
		});

		try {
			jsonData = (await Promise.all(fileReadPromises)) as any;
			console.log(jsonData);
		} catch (error) {
			console.error('Error reading files:', error);
		}
	}

	onMount(async () => {
		console.log('files', await readFilesAsJSON(files));
	});

	let showBoxes = true;
	let showText = true;
	let fadeImage = true;
	let fontSize = 16;
	let pageWidth = 650;
	const BASE_PAGE_WIDTH = 650;
</script>

<div class="sticky top-0 bg-gray-600 text-white p-4 z-10">
	<label class="mr-4"
		><input type="checkbox" class="mr-2" bind:checked={showBoxes} />Show boxes</label
	>
	<label class="mr-4"><input type="checkbox" class="mr-2" bind:checked={showText} />Show text</label
	>
	<label class="mr-4"
		><input type="checkbox" class="mr-2" bind:checked={fadeImage} />Fade image</label
	>
	<label class="mr-4"
		><input type="range" class="mr-2" bind:value={fontSize} min="4" max="40" />Overlay text size</label
	>
	<label class="mr-4"
		><input type="range" class="mr-2" bind:value={pageWidth} min="100" max="1500" />Page width</label
	>
</div>

{#each orderedPages as page}
	<div class="mt-4 mx-4">Page {page.pageNumber}</div>
	<div class="flex">
		<div
			class="mx-4 relative flex-shrink-0"
			style="width: {pageWidth}px; height: {page.dimension.height /
				(page.dimension.width / pageWidth)}px;"
		>
			<img
				class="absolute inset-0"
				alt="Page {page.pageNumber}"
				src={getB64Image(page)}
				style={fadeImage ? 'filter: brightness(0.2);' : ''}
			/>
			{#each page.tokens as token}
				{#if showBoxes}
					<div
						class="absolute text-white"
						style="left: {getLeft(token.layout) * 100}%; top: {getTop(token.layout) *
							100}%; width: {getRight(token.layout) * 100 -
							getLeft(token.layout) * 100}%; height: {getBottom(token.layout) * 100 -
							getTop(token.layout) *
								100}%; border: 1px solid rgba(255, 0, 0, 0.4); border-radius: 4px; background-color: rgba(255, 0, 0, 0.2);"
					/>
				{/if}
				{#if showText}
					<div
						class="absolute text-black font-bold"
						style="left: {getLeft(token.layout) * 100}%; top: {getTop(token.layout) *
							100}%; width: {getRight(token.layout) * 100 -
							getLeft(token.layout) * 100}%; height: {getBottom(token.layout) * 100 -
							getTop(token.layout) * 100}%; font-size: {(fontSize * pageWidth) /
							BASE_PAGE_WIDTH}px; text-shadow: 0 0 2px white, 0 0 4px white;"
					>
						{getText(page, token.layout)}
					</div>
				{/if}
			{/each}
		</div>
		<div class="max-w-md ml-8 flex-0 whitespace-break-spaces break-words">
			{getText(page, page.layout)}
		</div>
	</div>
{/each}
