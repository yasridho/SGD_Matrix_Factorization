<script lang="ts">
	import katex from 'katex';
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';

	export let expression: string;

	// Store for rendered HTML to use with Svelte's `@html` directive
	const renderedExpression = writable('');

	onMount(() => {
		try {
			// Render the LaTeX expression to HTML using KaTeX
			renderedExpression.set(
				katex.renderToString(expression, {
					throwOnError: false,
					displayMode: true // Set to true for display mode (block math)
				})
			);
		} catch (error) {
			console.error('KaTeX rendering error:', error);
			renderedExpression.set(`<span style="color: red;">Error rendering LaTeX</span>`);
		}
	});
</script>

<!-- Render the processed HTML output with Svelte's `@html` directive -->
<div class="katex">
	{@html $renderedExpression}
</div>

<style>
	.katex {
		display: inline-block;
		vertical-align: middle;
	}
</style>
