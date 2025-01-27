<script lang="ts">
	import AgGridSvelte from 'ag-grid-svelte/AgGridSvelte.svelte'
	import 'ag-grid-community/styles/ag-grid.css'
	import 'ag-grid-community/styles/ag-theme-alpine.css'

	import { getContext, onMount } from 'svelte'
	import type { Output } from '../../../rx'
	import type { AppViewerContext, RichConfiguration, RichConfigurations } from '../../../types'
	import InputValue from '../../helpers/InputValue.svelte'
	import type { AppInput } from '../../../inputType'
	import RunnableWrapper from '../../helpers/RunnableWrapper.svelte'
	import { isObject } from '$lib/utils'

	import Alert from '$lib/components/common/alert/Alert.svelte'
	import { initOutput } from '$lib/components/apps/editor/appUtils'

	export let id: string
	export let componentInput: AppInput | undefined
	export let configuration: RichConfigurations
	export let initializing: boolean | undefined = undefined
	export let render: boolean

	let result: Record<number, any>[] | undefined = undefined

	const { worldStore, selectedComponent } = getContext<AppViewerContext>('AppViewerContext')

	let outputs = initOutput($worldStore, id, {
		selectedRowIndex: 0,
		selectedRow: {},
		result: [] as Record<number, any>[],
		loading: false
	})

	let selectedRowIndex = -1

	function toggleRow(row: Record<string, any>, rowIndex: number) {
		if (selectedRowIndex !== rowIndex) {
			selectedRowIndex = rowIndex
			outputs?.selectedRow.set(row.original)
			outputs?.selectedRowIndex.set(rowIndex)
		}
	}

	let mounted = false
	onMount(() => {
		mounted = true
	})

	$: selectedRowIndex === -1 &&
		Array.isArray(result) &&
		result.length > 0 &&
		// We need to wait until the component is mounted so the world is created
		mounted &&
		outputs &&
		toggleRow({ original: result[0] }, 0)

	$: outputs?.result?.set(result ?? [])

	let clientHeight
	let clientWidth

	let columnDefs: any = undefined

	let allEditable: boolean | undefined = undefined
	let pagination: boolean | undefined = undefined
	let pageSize: number | undefined = 10

	function onCellValueChanged(event) {
		if (result) {
			let dataCell = event.newValue
			try {
				dataCell = JSON.parse(dataCell)
			} catch (e) {}
			result[event.node.rowIndex][event.column.colId] = dataCell
		}
	}
</script>

<InputValue {id} input={configuration.columnDefs} bind:value={columnDefs} />
<InputValue {id} input={configuration.allEditable} bind:value={allEditable} />
<InputValue {id} input={configuration.pagination} bind:value={pagination} />
<InputValue {id} input={configuration.pageSize} bind:value={pageSize} />

<RunnableWrapper {outputs} {render} {componentInput} {id} bind:initializing bind:result>
	{#if Array.isArray(result) && result.every(isObject)}
		<div
			class="border border-gray-300 shadow-sm divide-y divide-gray-300  flex flex-col h-full"
			bind:clientHeight
			bind:clientWidth
		>
			<div
				on:pointerdown|stopPropagation={() => {
					$selectedComponent = id
				}}
				style:height="{clientHeight}px"
				style:width="{clientWidth}px"
				class="ag-theme-alpine"
			>
				{#key pagination}
					<AgGridSvelte
						bind:rowData={result}
						{columnDefs}
						{pagination}
						paginationPageSize={pageSize}
						defaultColDef={{ flex: 1, editable: allEditable, onCellValueChanged }}
					/>
				{/key}
			</div>
		</div>
	{:else if result != undefined}
		<Alert title="Parsing issues" type="error" size="xs">
			The result should be an array of objects, received:
			<pre class="overflow-auto">
				{JSON.stringify(result)}
			</pre>
		</Alert>
	{/if}
</RunnableWrapper>
