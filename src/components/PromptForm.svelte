<script lang="ts">
	import autosize from 'svelte-autosize'
	import { page } from '$app/stores'
	import type { Models } from '@kittycad/lib/types'
	import ArrowRight from './Icons/ArrowRight.svelte'
	import { endpoints } from '$lib/endpoints'
	import { localGenerations, unreadGenerations } from '$lib/stores'
	import { paths } from '$lib/paths'
	import { goto } from '$app/navigation'

	export let input: HTMLTextAreaElement | null
	export let token: string
	export let disabled: boolean = false

	let inputValue = $page.url.searchParams.get('prompt') ?? ''
	let button = null as HTMLButtonElement | null
	let error: string | null = null
	let isSubmitting = false
	let showSuccessMessage: boolean = false
	let isCoolingDown = false

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter') {
			button?.click()
		}
	}

	const submitPrompt = async (prompt: string) => {
		const OUTPUT_FORMAT: Models['FileExportFormat_type'] = 'gltf'
		const response = await fetch(endpoints.prompt(OUTPUT_FORMAT), {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: 'Bearer ' + token
			},
			body: JSON.stringify({ prompt })
		})

		if (!response.ok) {
			error = 'Failed to submit prompt'
			return
		}

		const responseData = (await response
			.json()
			.catch((e) => console.error('failed to parse response data', e))) as Models['TextToCad_type']

		if (responseData) {
			$localGenerations = [responseData, ...$localGenerations]
			$unreadGenerations = [responseData.id, ...$unreadGenerations]
		}

		goto(paths.VIEW(responseData.id))
	}

	const submitForm = async (e: Event) => {
		e.preventDefault()
		isSubmitting = true
		const prompt = (e.target as HTMLFormElement).prompt.value
		await submitPrompt(prompt)
		const form = e.target as HTMLFormElement
		form.reset()

		showSuccessMessage = true

		isCoolingDown = true
		setTimeout(() => {
			isCoolingDown = false
		}, 3000)

		isSubmitting = false
	}
</script>

<form
	on:submit={submitForm}
	class="flex border border-chalkboard-30 dark:border-chalkboard-90 focus-within:border-currentColor items-stretch text-lg"
>
	<label class="flex-1 grid place-items-center">
		<span class="sr-only">Enter a text-to-CAD prompt:</span>
		<!-- svelte-ignore a11y-autofocus -->
		<textarea
			autocapitalize="false"
			name="prompt"
			autofocus
			placeholder="e.g. Create a plate with 4 holes and rounded corners"
			required
			disabled={isSubmitting}
			class="w-full tracking-wide px-4 py-1 focus:outline-none focus:bg-green/20 focus:placeholder-shown:bg-green/10 max-h-64"
			bind:this={input}
			bind:value={inputValue}
			on:keydown={handleKeydown}
			on:input={() => {
				showSuccessMessage = false
				error = null
			}}
			use:autosize
		/>
	</label>
	<button
		type="submit"
		class="submit"
		disabled={disabled || isCoolingDown || isSubmitting}
		bind:this={button}
	>
		<span class="sr-only md:not-sr-only md:pt-0.5">
			{#if isSubmitting}
				Submitting
			{:else}
				Submit
			{/if}
		</span>
		<ArrowRight class="w-8 h-8 md:w-5 md:h-5" />
	</button>
</form>
{#if error}
	<p class="text-red mt-2">{error}</p>
{:else if showSuccessMessage}
	<p class="border border-t-0 border-chalkboard-70 dark:border-chalkboard-40 p-2 bg-green/40">
		Prompt submitted!
	</p>
{/if}

<style lang="postcss">
	.submit {
		@apply m-1 md:px-4 lg:px-6 md:pt-1 border;
		@apply self-end flex items-center justify-center gap-2;
		@apply font-mono uppercase tracking-[1px] text-sm;
		@apply border-chalkboard-100 dark:border-chalkboard-20;
		@apply text-chalkboard-120 bg-green hover:hue-rotate-15;
		@apply disabled:bg-chalkboard-40 dark:disabled:bg-chalkboard-70;
	}

	textarea {
		@apply bg-transparent;
	}
</style>
