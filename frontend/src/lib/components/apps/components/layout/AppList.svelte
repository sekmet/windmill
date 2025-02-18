<script lang="ts">
	import { getContext } from 'svelte'
	import { initConfig, initOutput } from '../../editor/appUtils'
	import SubGridEditor from '../../editor/SubGridEditor.svelte'
	import type { AppViewerContext, ComponentCustomCSS, RichConfigurations } from '../../types'
	import { concatCustomCss } from '../../utils'
	import InitializeComponent from '../helpers/InitializeComponent.svelte'
	import ListWrapper from './ListWrapper.svelte'
	import type { AppInput } from '../../inputType'
	import ResolveConfig from '../helpers/ResolveConfig.svelte'
	import { components } from '../../editor/component'
	import RunnableWrapper from '../helpers/RunnableWrapper.svelte'
	import { Button } from '$lib/components/common'
	import { Loader2, ChevronLeft, ChevronRight } from 'lucide-svelte'

	export let id: string
	export let componentInput: AppInput | undefined
	export let configuration: RichConfigurations
	export let customCss: ComponentCustomCSS<'containercomponent'> | undefined = undefined
	export let render: boolean
	export let initializing: boolean | undefined

	const { app, focusedGrid, selectedComponent, worldStore, connectingInput } =
		getContext<AppViewerContext>('AppViewerContext')
	let page = 0

	const outputs = initOutput($worldStore, id, {
		result: undefined,
		loading: false,
		inputs: {},
		page: 0
	})

	let resolvedConfig = initConfig(
		components['listcomponent'].initialData.configuration,
		configuration
	)

	function onFocus() {
		$focusedGrid = {
			parentComponentId: id,
			subGridIndex: 0
		}
	}

	$: css = concatCustomCss($app.css?.containercomponent, customCss)
	let result: any[] | undefined = undefined

	$: isCard = resolvedConfig.width?.selected == 'card'

	let inputs = {}
	let loading: boolean = false
	let isPreviousLoading = false
	let isNextLoading = false

	$: if (!loading) {
		isPreviousLoading = false
		isNextLoading = false
	}

	function getPagination(
		configuration: {
			auto: { pageSize: number | undefined }
			manual: { pageCount: number | undefined }
		},
		mode: 'auto' | 'manual' = 'auto',
		initialData: Array<any> | undefined = [],
		page: number = 0
	) {
		if (mode === 'auto') {
			const pageSize: number = configuration.auto.pageSize ?? 0
			const data = initialData?.slice(0 + page * pageSize, pageSize + page * pageSize) ?? []
			const shouldDisplayPagination = pageSize < initialData?.length ?? false
			const total = Math.ceil(initialData?.length / pageSize ?? 0)

			return {
				data,
				shouldDisplayPagination,
				indexOffset: page * pageSize,
				disableNext: pageSize > 0 && (page + 1) * pageSize >= initialData?.length,
				total: total
			}
		} else {
			const pageCount = configuration.manual.pageCount ?? 0
			const total = pageCount

			return {
				shouldDisplayPagination: true,
				data: initialData ?? [],
				indexOffset: 0,
				disableNext: page + 1 >= pageCount,
				total: total
			}
		}
	}

	$: pagination = getPagination(
		resolvedConfig.pagination?.configuration,
		resolvedConfig.pagination?.selected,
		result,
		page
	)
</script>

{#each Object.keys(components['listcomponent'].initialData.configuration) as key (key)}
	<ResolveConfig
		{id}
		{key}
		bind:resolvedConfig={resolvedConfig[key]}
		configuration={configuration[key]}
	/>
{/each}

<InitializeComponent {id} />

<RunnableWrapper
	render={true}
	{outputs}
	autoRefresh
	{componentInput}
	{id}
	bind:initializing
	bind:result
	bind:loading
>
	<div class="flex flex-col divide-y h-full">
		<div
			class="w-full flex flex-wrap overflow-auto {isCard ? 'h-full gap-2' : 'divide-y max-h-full'}"
		>
			{#if $app.subgrids?.[`${id}-0`]}
				{#if Array.isArray(pagination.data) && pagination.data.length > 0}
					{#each pagination?.data ?? [] as value, index}
						<div
							style={`${
								isCard
									? `min-width: ${resolvedConfig.width?.configuration?.card?.minWidthPx}px; `
									: ''
							} max-height: ${resolvedConfig.heightPx}px;`}
							class="overflow-auto {!isCard ? 'w-full' : 'border'}"
						>
							<ListWrapper
								on:inputsChange={() => {
									outputs?.inputs.set(inputs, true)
								}}
								bind:inputs
								{value}
								index={index + pagination.indexOffset}
							>
								<SubGridEditor
									visible={render}
									{id}
									class={css?.container?.class}
									style={css?.container?.style}
									subGridId={`${id}-0`}
									containerHeight={resolvedConfig.heightPx}
									on:focus={() => {
										if (!$connectingInput.opened) {
											$selectedComponent = [id]
										}
										onFocus()
									}}
								/>
							</ListWrapper>
						</div>
					{/each}
				{:else}
					<ListWrapper disabled value={undefined} index={0}>
						<SubGridEditor visible={false} {id} subGridId={`${id}-0`} />
					</ListWrapper>
					{#if !Array.isArray(result)}
						<div class="text-center text-tertiary">Input data is not an array</div>
					{/if}
				{/if}
			{/if}
		</div>
		{#if pagination.shouldDisplayPagination}
			<div class="bg-surface-secondary h-8 flex flex-row gap-1 p-1 items-center">
				<Button
					size="xs2"
					variant="border"
					color="light"
					on:click={() => {
						isPreviousLoading = true
						page = page - 1
						outputs?.page.set(page, true)
					}}
					disabled={page === 0}
				>
					<div class="flex flex-row gap-1 items-center">
						{#if isPreviousLoading && loading}
							<Loader2 size={14} class="animate-spin" />
						{:else}
							<ChevronLeft size={14} />
						{/if}
						Previous
					</div>
				</Button>
				<Button
					size="xs2"
					variant="border"
					color="light"
					on:click={() => {
						isNextLoading = true
						page = page + 1
						outputs?.page.set(page, true)
					}}
					disabled={pagination.disableNext}
				>
					<div class="flex flex-row gap-1 items-center">
						Next

						{#if isNextLoading && loading}
							<Loader2 size={14} class="animate-spin" />
						{:else}
							<ChevronRight size={14} />
						{/if}
					</div>
				</Button>
				<div class="text-xs">{page + 1} of {pagination.total}</div>
			</div>
		{/if}
	</div>
</RunnableWrapper>
