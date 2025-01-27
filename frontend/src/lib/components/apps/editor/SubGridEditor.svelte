<script lang="ts">
	import { classNames } from '$lib/utils'
	import { createEventDispatcher, getContext } from 'svelte'
	import { twMerge } from 'tailwind-merge'
	import { columnConfiguration, isFixed, toggleFixed } from '../gridUtils'
	import type { AppEditorContext, AppViewerContext, GridItem } from '../types'
	import Component from './component/Component.svelte'
	import { expandGriditem, findGridItem } from './appUtils'
	import { push } from '$lib/history'
	import Grid from '../svelte-grid/Grid.svelte'
	import GridViewer from './GridViewer.svelte'

	export let containerHeight: number | undefined = undefined
	export let containerWidth: number | undefined = undefined
	let classes = ''
	export { classes as class }
	export let style = ''
	export let noPadding = false
	export let noYPadding = false
	export let subGridId: string
	export let visible: boolean = true
	export let id: string
	export let shouldHighlight: boolean = true

	const dispatch = createEventDispatcher()

	const { app, connectingInput, selectedComponent, focusedGrid, mode, parentWidth, breakpoint } =
		getContext<AppViewerContext>('AppViewerContext')

	const editorContext = getContext<AppEditorContext>('AppEditorContext')

	$: highlight = id === $focusedGrid?.parentComponentId && shouldHighlight

	const onpointerdown = (e) => {
		dispatch('focus')
	}

	function selectComponent(id: string) {
		if (!$connectingInput.opened) {
			dispatch('focus')
			$selectedComponent = id
		}
	}

	function lock(dataItem: GridItem) {
		let fComponent = findGridItem($app, dataItem.id)
		if (fComponent) {
			fComponent = toggleFixed(fComponent)
		}
		$app = $app
	}

	let container: HTMLElement | undefined = undefined
</script>

<div
	class="translate-x-0 translate-y-0 relative w-full subgrid {visible
		? 'visible'
		: 'invisible h-0 overflow-hidden'} 	"
	bind:this={container}
	on:pointerdown={onpointerdown}
>
	<div
		class={twMerge(
			'overflow-auto',
			noYPadding ? '' : 'py-2',
			classes ?? '',
			noPadding ? 'px-0' : 'px-2'
		)}
		style="height: {containerHeight}px; {style ?? ''}"
	>
		{#if $mode !== 'preview'}
			<div class={highlight ? 'border-gray-400  border border-dashed min-h-full' : ''}>
				<Grid
					items={$app.subgrids?.[subGridId] ?? []}
					on:redraw={(e) => {
						push(editorContext?.history, $app)
						if ($app.subgrids) {
							$app.subgrids[subGridId] = e.detail
						}
					}}
					onTopId={$selectedComponent}
					let:dataItem
					rowHeight={36}
					cols={columnConfiguration}
					fastStart={true}
					gap={[4, 2]}
					scroller={container}
					parentWidth={$parentWidth - 17}
					{containerWidth}
				>
					<!-- svelte-ignore a11y-click-events-have-key-events -->
					{#if $connectingInput.opened}
						<div
							on:pointerenter={() => ($connectingInput.hoveredComponent = dataItem.id)}
							on:pointerleave={() => ($connectingInput.hoveredComponent = undefined)}
							class="absolute w-full h-full bg-black border-2 bg-opacity-25 z-20 flex justify-center items-center"
						/>
						<div
							style="transform: translate(-50%, -50%);"
							class="absolute w-fit justify-center bg-indigo-500/90 left-[50%] top-[50%] z-50 px-6 rounded border text-white py-2 text-5xl center-center"
						>
							{dataItem.id}
						</div>
					{/if}

					<!-- svelte-ignore a11y-click-events-have-key-events -->
					<div
						on:pointerdown={() => selectComponent(dataItem.id)}
						class={classNames(
							'h-full w-full center-center',
							$selectedComponent === dataItem.id ? 'active-grid-item' : '',
							'top-0'
						)}
					>
						<Component
							render={visible}
							component={dataItem.data}
							selected={$selectedComponent === dataItem.id}
							locked={isFixed(dataItem)}
							on:lock={() => lock(dataItem)}
							on:expand={() => {
								const parentGridItem = findGridItem($app, id)

								if (!parentGridItem) {
									return
								}
								$selectedComponent = dataItem.id
								push(editorContext?.history, $app)

								expandGriditem(
									$app.subgrids?.[subGridId] ?? [],
									dataItem.id,
									$breakpoint,
									parentGridItem
								)
								$app = $app
							}}
						/>
					</div>
				</Grid>
			</div>
		{:else}
			<GridViewer
				onTopId={$selectedComponent}
				items={$app.subgrids?.[subGridId] ?? []}
				let:dataItem
				rowHeight={36}
				cols={columnConfiguration}
				gap={[4, 2]}
				parentWidth={$parentWidth - 17}
				{containerWidth}
			>
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				<div
					on:pointerdown={() => selectComponent(dataItem.id)}
					class={classNames('h-full w-full center-center', 'top-0')}
				>
					<Component
						render={visible}
						component={dataItem.data}
						selected={$selectedComponent === dataItem.id}
						locked={isFixed(dataItem)}
					/>
				</div>
			</GridViewer>
		{/if}
	</div>
</div>
