<script lang="ts">
	import { Clipboard } from 'lucide-svelte'
	import { copyToClipboard } from '../../../../utils'
	import Button from '../../../common/button/Button.svelte'
	import Popover from '../../../Popover.svelte'
	import type { AppInput, EvalAppInput } from '../../inputType'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import RunnableWrapper from '../helpers/RunnableWrapper.svelte'
	import { twMerge } from 'tailwind-merge'
	import type {
		AppEditorContext,
		AppViewerContext,
		ComponentCustomCSS,
		RichConfigurations
	} from '../../types'
	import { getContext } from 'svelte'
	import { initConfig, initOutput } from '../../editor/appUtils'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import { get } from 'svelte/store'
	import ResolveConfig from '../helpers/ResolveConfig.svelte'
	import { components } from '../../editor/component'
	import { isCodeInjection } from '$lib/components/flows/utils'

	export let id: string
	export let componentInput: AppInput | undefined
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = 'left'
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export let configuration: RichConfigurations
	export let initializing: boolean | undefined = undefined
	export let customCss: ComponentCustomCSS<'textcomponent'> | undefined = undefined
	export let render: boolean

	let resolvedConfig = initConfig(
		components['textcomponent'].initialData.configuration,
		configuration
	)

	const { app, worldStore, mode } = getContext<AppViewerContext>('AppViewerContext')

	const editorcontext = getContext<AppEditorContext>('AppEditorContext')

	let result: string | undefined = undefined

	if (componentInput?.type == 'template' && !isCodeInjection(componentInput.eval)) {
		result = componentInput.eval
		initializing = false
	}

	const outputs = initOutput($worldStore, id, {
		result,
		loading: initializing
	})

	function getComponent() {
		switch (resolvedConfig.style) {
			case 'Title':
				return 'h1'
			case 'Subtitle':
				return 'h3'
			case 'Body':
				return 'p'
			case 'Caption':
				return 'p'
			case 'Label':
				return 'label'
			default:
				return 'p'
		}
	}

	function getClasses() {
		switch (resolvedConfig.style) {
			case 'Caption':
				return 'text-sm italic text-gray-500'
			case 'Label':
				return 'font-semibold text-sm'
			default:
				return ''
		}
	}

	let component = 'p'
	let classes = ''
	$: resolvedConfig.style && (component = getComponent())
	$: resolvedConfig.style && (classes = getClasses())
</script>

{#each Object.keys(components['textcomponent'].initialData.configuration) as key (key)}
	<ResolveConfig
		{id}
		{key}
		bind:resolvedConfig={resolvedConfig[key]}
		configuration={configuration[key]}
	/>
{/each}

<RunnableWrapper {outputs} {render} {componentInput} {id} bind:initializing bind:result>
	<div class="h-full w-full overflow-hidden">
		<AlignWrapper {horizontalAlignment} {verticalAlignment}>
			{#if !result || result === ''}
				<div class="text-gray-400 bg-gray-100 flex justify-center items-center h-full w-full">
					No text
				</div>
			{:else}
				<!-- svelte-ignore a11y-click-events-have-key-events -->
				<div
					class="flex flex-wrap gap-2 pb-0.5  {$mode === 'dnd' && componentInput?.type == 'template'
						? 'cursor-text'
						: ''}"
					on:click={() => {
						if ($mode === 'dnd' && componentInput?.type == 'template') {
							let ontextfocus = editorcontext?.ontextfocus
							if (ontextfocus) {
								get(ontextfocus)?.()
							}
						}
					}}
				>
					<svelte:element
						this={component}
						class={twMerge(
							'whitespace-pre-wrap',
							$app.css?.['textcomponent']?.['text']?.class,
							customCss?.text?.class,
							classes
						)}
						style={[$app.css?.['textcomponent']?.['text']?.style, customCss?.text?.style].join(';')}
					>
						{String(result)}
						{#if resolvedConfig.tooltip != ''}
							<Tooltip>{resolvedConfig.tooltip}</Tooltip>
						{/if}
						{#if resolvedConfig.copyButton && result}
							<Popover notClickable>
								<Button
									variant="border"
									size="xs"
									color="dark"
									btnClasses="!p-1"
									on:click={() => copyToClipboard(result)}
								>
									<Clipboard size={14} strokeWidth={2} />
								</Button>
								<svelte:fragment slot="text">Copy to clipboard</svelte:fragment>
							</Popover>
						{/if}
					</svelte:element>
				</div>
			{/if}
		</AlignWrapper>
	</div>
</RunnableWrapper>
