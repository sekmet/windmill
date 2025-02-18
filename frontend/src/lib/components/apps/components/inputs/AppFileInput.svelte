<script lang="ts">
	import { getContext } from 'svelte'
	import { twMerge } from 'tailwind-merge'
	import { FileInput } from '../../../common'
	import { initOutput } from '../../editor/appUtils'
	import type { AppViewerContext, ComponentCustomCSS, RichConfigurations } from '../../types'
	import { concatCustomCss } from '../../utils'
	import InputValue from '../helpers/InputValue.svelte'
	import InitializeComponent from '../helpers/InitializeComponent.svelte'

	export let id: string
	export let configuration: RichConfigurations
	export let customCss: ComponentCustomCSS<'containercomponent'> | undefined = undefined
	export let render: boolean

	const { app, worldStore } = getContext<AppViewerContext>('AppViewerContext')

	let acceptedFileTypes: string[] | undefined = undefined
	let allowMultiple: boolean | undefined = undefined
	let text: string | undefined = undefined
	let includeMimeType: boolean | undefined = undefined

	let outputs = initOutput($worldStore, id, {
		result: [] as { name: string; data: string }[] | undefined
	})

	// Receives Base64 encoded strings from the input component
	async function handleChange(files: { name: string; data: string }[] | undefined) {
		if (includeMimeType === false) {
			files = files?.map((file) => {
				const [_, data] = file.data.split('base64,')
				return { name: file.name, data }
			})
		}

		outputs?.result.set(files)
	}

	$: css = concatCustomCss($app.css?.fileinputcomponent, customCss)
</script>

<InputValue {id} input={configuration.acceptedFileTypes} bind:value={acceptedFileTypes} />
<InputValue {id} input={configuration.allowMultiple} bind:value={allowMultiple} />
<InputValue {id} input={configuration.text} bind:value={text} />
<InputValue {id} input={configuration.includeMimeType} bind:value={includeMimeType} />

<InitializeComponent {id} />

{#if render}
	<div class="w-full h-full p-1">
		<FileInput
			accept={acceptedFileTypes?.length ? acceptedFileTypes?.join(', ') : undefined}
			multiple={allowMultiple}
			convertTo="base64"
			returnFileNames
			on:change={({ detail }) => {
				handleChange(detail)
			}}
			class={twMerge('w-full h-full', css?.container?.class)}
			style={css?.container?.style}
		>
			{text}
		</FileInput>
	</div>
{/if}
