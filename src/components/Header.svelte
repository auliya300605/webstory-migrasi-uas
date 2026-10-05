<script>
	// shadow, plain, line, bubble
	import wordmark from "$svg/wordmark-script-stacked-plain.svg";
</script>

<header>
	<div class="wordmark">
		<a href="https://pudding.cool" aria-label="The Pudding" target="_self"
			>{@html wordmark}</a
		>
	</div>
</header>

<style>
	.wordmark {
		max-width: 10em;
		margin: 0 auto;
		padding: 32px 0;
		/* transform: rotate(-4deg); */
	}

	.wordmark a {
		border: none;
		display: block;
		color: var(--color-fg);
	}

	.wordmark a:hover {
		background-color: transparent;
	}

	header {
    margin-bottom: 0.5rem !important;
    padding-bottom: 0 !important;
}
</style>
