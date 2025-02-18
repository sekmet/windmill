<script lang="ts">
	import { Button } from '../common'

	import { SUPPORTED_LANGUAGES, copilot } from './lib'
	import type { SupportedLanguage } from '$lib/common'
	import { sendUserToast } from '$lib/toast'
	import type Editor from '../Editor.svelte'
	import { faCheck, faClose, faMagicWandSparkles } from '@fortawesome/free-solid-svg-icons'
	import { dbSchemas, existsOpenaiResourcePath, type DBSchema } from '$lib/stores'
	import type DiffEditor from '../DiffEditor.svelte'
	import { scriptLangToEditorLang } from '$lib/scripts'
	import Popover from '../Popover.svelte'
	import Popup from '../common/popup/Popup.svelte'
	import { writable } from 'svelte/store'
	import { sleep } from '$lib/utils'
	import { WindmillIcon } from '../icons'
	import HighlightCode from '../HighlightCode.svelte'
	import LoadingIcon from '../apps/svelte-select/lib/LoadingIcon.svelte'
	import { autoPlacement } from '@floating-ui/core'

	// props
	export let lang: SupportedLanguage
	export let editor: Editor | undefined
	export let diffEditor: DiffEditor | undefined
	export let error: string

	// state
	let genLoading: boolean = false
	let generatedCode = writable<string>('')
	let generatedExplanation = writable<string>('')
	let dbSchema: DBSchema | undefined = undefined
	let abortController: AbortController | undefined = undefined

	async function onFix(closePopup: () => void) {
		if (!error) {
			return
		}
		try {
			genLoading = true
			abortController = new AbortController()
			await copilot(
				{
					language: lang,
					code: editor?.getCode() || '',
					error,
					dbSchema: dbSchema,
					type: 'fix'
				},
				generatedCode,
				abortController,
				generatedExplanation
			)
			setupDiff()
			diffEditor?.setModified($generatedCode)
			await sleep(500)
			closePopup()
			await sleep(300)
			showDiff()
		} catch (err) {
			if (err?.message) {
				sendUserToast('Failed to generate code: ' + err.message, true)
			} else {
				sendUserToast('Failed to generate code', true)
				console.error(err)
			}
			closePopup()
		} finally {
			genLoading = false
		}
	}

	function acceptDiff() {
		editor?.setCode(diffEditor?.getModified() || '')
		editor?.format()
		clear()
	}

	function rejectDiff() {
		clear()
	}

	function setupDiff() {
		diffEditor?.setupModel(scriptLangToEditorLang(lang))
		diffEditor?.setOriginal(editor?.getCode() || '')
	}

	function showDiff() {
		diffEditor?.show()
		editor?.hide()
	}

	function hideDiff() {
		editor?.show()
		diffEditor?.hide()
	}

	function clear() {
		$generatedCode = ''
		$generatedExplanation = ''
	}

	$: lang && clear()

	$: !$generatedCode && hideDiff()

	$: dbSchema = $dbSchemas[Object.keys($dbSchemas)[0]]
</script>

{#if SUPPORTED_LANGUAGES.has(lang)}
	<div class="mb-2">
		{#if !genLoading && $generatedCode.length > 0}
			<div class="flex gap-1">
				<Button
					title="Discard generated code"
					size="xs"
					color="red"
					spacingSize="xs2"
					on:click={rejectDiff}
					variant="contained"
					startIcon={{ icon: faClose }}
				>
					Discard
				</Button><Button
					title="Accept generated code"
					size="xs"
					color="green"
					spacingSize="xs2"
					on:click={acceptDiff}
					startIcon={{ icon: faCheck }}
				>
					Accept
				</Button>
				{#if $generatedExplanation.length > 0}
					<Popover>
						<svelte:fragment slot="text">{$generatedExplanation}</svelte:fragment>
						<Button size="xs" color="light" variant="contained" spacingSize="xs2">Explain</Button
						></Popover
					>
				{/if}
			</div>
		{:else}
			<Popup
				floatingConfig={{
					middleware: [
						autoPlacement({
							allowedPlacements: [
								'bottom-start',
								'bottom-end',
								'top-start',
								'top-end',
								'top',
								'bottom'
							]
						})
					]
				}}
				let:close
			>
				<svelte:fragment slot="button">
					<Button
						title="Fix code"
						size="xs"
						color={genLoading ? 'red' : 'blue'}
						spacingSize="xs2"
						startIcon={genLoading ? undefined : { icon: faMagicWandSparkles }}
						nonCaptureEvent={!genLoading}
						on:click={genLoading ? () => abortController?.abort() : undefined}
						>{#if genLoading}
							<WindmillIcon
								white
								class="mr-1 text-white"
								height="16px"
								width="20px"
								spin="veryfast"
							/>
							Cancel
						{:else}
							AI Fix
						{/if}
					</Button>
				</svelte:fragment>
				{@const fixAction = (_) => {
					onFix(() => close(null))
				}}
				<div use:fixAction>
					<div class="w-[42rem] min-h-[3rem] max-h-[34rem] overflow-y-scroll">
						{#if $generatedCode.length > 0}
							<div class="overflow-x-scroll">
								<HighlightCode language={lang} code={$generatedCode} />
							</div>
							{#if $generatedExplanation.length > 0}
								<p class="text-sm mt-2"
									><span class="font-bold">Explanation:</span> {$generatedExplanation}</p
								>
							{/if}
						{:else}
							<LoadingIcon />
						{/if}
					</div>
					{#if !$existsOpenaiResourcePath}
						<p class="text-sm"
							>Enable Windmill AI in the <a href="/workspace_settings?tab=openai"
								>workspace settings.</a
							></p
						>
					{/if}
				</div>
			</Popup>
		{/if}
	</div>
{/if}
