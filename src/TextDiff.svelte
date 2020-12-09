<svelte:options tag="text-diff" immutable={true} />
<script>
	import { createEventDispatcher } from 'svelte';
	import { diffChars } from "diff"
	import { findBestMatch } from "string-similarity"

	export let basetext = "test"
	export let time = 60

	const dispatch = createEventDispatcher();

	const splitter = /\s(?! )/
	$: timeLeft = time
	let timeout = false

  $: trackText = basetext
	let writtenText = ""
	let mistakes = 0

	let prevBestIndex = 0

	function handleInput(e) {
		if (e.inputType.startsWith("delete") || e.data.length !== 1) {
			return
		}

		writtenText = writtenText + e.data

		const splitBText = basetext.split(splitter)
		const splitTText = trackText.split(splitter)
		const splitWText = writtenText.split(splitter)
		const lastFullWord = splitWText[splitWText.length - 1] || ""
		const bestMatchingWord = findBestMatch(lastFullWord, splitBText)

		if (bestMatchingWord.bestMatch.target.length < 3) {
			return
		}

		const indexOfBest = splitBText.indexOf(bestMatchingWord.bestMatch.target)

		const written = splitBText.slice(0, indexOfBest).join(" ")
		const notWritten = splitBText.slice(indexOfBest).join(" ")
		if ((indexOfBest < prevBestIndex || Math.abs(indexOfBest - prevBestIndex) > 3)) {
			return
		}

		prevBestIndex = indexOfBest

		const	fragment = document.createElement('span')
		const	writtenSpan = document.createElement('span')
		const	notWrittenSpan = document.createElement('span')
		writtenSpan.classList.add("written")
		writtenSpan.appendChild(document.createTextNode(`${written} `))
		fragment.appendChild(writtenSpan)
		notWrittenSpan.appendChild(document.createTextNode(notWritten))
		fragment.appendChild(notWrittenSpan)

		trackText = fragment.innerHTML
	}

	function makeDiff() {
		const splitBText = basetext.split(splitter)
		const splitWText = writtenText.trim().split(splitter)
		const lastFullWord = splitWText[splitWText.length - 1] || ""
		const bestMatchingWord = findBestMatch(lastFullWord, splitBText)
		const indexOfBest = splitBText.indexOf(bestMatchingWord.bestMatch.target)
		const written = splitBText.slice(0, indexOfBest + 1).join(" ")
		const notWritten = splitBText.slice(indexOfBest + 1).join(" ")

		const diff = diffChars(written, writtenText)
		const	fragment = document.createElement('span')

		diff.forEach((part) => {
			if (part.added || part.removed) {
				mistakes += part.count
			}
			// green for additions, red for deletions
			// grey for common parts
			const className = part.added ? 'added' :
				part.removed ? 'removed' : 'correct'
			const span = document.createElement('span')
			span.classList.add(className)
			span.appendChild(document
				.createTextNode(part.value))
			fragment.appendChild(span)
		})
		fragment.appendChild(document.createTextNode(` ${notWritten}`))

		trackText = fragment.innerHTML
	}

	function dispatchSavedDateEvent() {
		dispatch('saved', {
			mistakes,
			written: writtenText.length,
		});
  }

	function setTimer() {
		const timer = setInterval(() => {
			timeLeft = timeLeft - 1
		}, 1000)
		setTimeout(() => {
			timeout = true
			makeDiff()
			dispatchSavedDateEvent()
			clearInterval(timer)
		}, time * 1000)

	}
</script>

<main>
	{#if timeout}
		<div class="info"><strong>Legenda: </strong><span class="correct">Správně</span> <span class="removed">Chybějící</span> <span class="added">Navíc</span> <span>Nenapsáno</span></div>
	{:else}
		<div class="info"><strong>Legenda: </strong><span class="written">Napsáno</span> <span>Nenapsáno</span></div>
	{/if}
	<h1>{@html trackText}</h1>
	<textarea value={writtenText} on:beforeinput|once={setTimer} on:beforeinput|preventDefault={handleInput} disabled={timeout}></textarea>
	<div class="info">
		<span><strong>Napsáno:</strong> {writtenText.length}</span>
		{#if !timeout}
		<span><strong>Zbývá:</strong> {timeLeft}s</span>
		{:else}
		<span><strong>Chyb:</strong> {mistakes}</span>
		{/if}
	</div>
</main>

<style>
	.info {
		display: flex;
		justify-content: space-between;
		margin-top: 1rem;
	}

	.added, .written {
		color: lightgrey;
	}

	.added {
		text-decoration: line-through;
	}

	.removed {
		color: red;
		text-decoration: underline;
	}

	.correct {
		color: green
	}

	main {
		text-align: center;
		padding: 1em;
		max-width: 100%;
		margin: 0 auto;
	}

	h1 {
		font-size: 1.5em;
		font-weight: 300;
		user-select: none;
		white-space: pre-wrap;
	}

	textarea {
		width: 100%;
		height: 200px;
		border: none;
		border-bottom: 1px solid lightgrey;
		resize: none;
	}

	textarea:focus {
		outline: none;
		border: none;
		border-bottom: 2px solid darkblue;
	}

	textarea::selection {
		background-color: transparent;
	}
</style>
