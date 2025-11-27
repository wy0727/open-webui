<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { createEventDispatcher, onMount, getContext } from 'svelte';
	import { config as backendConfig, user } from '$lib/stores';
	import { getBackendConfig } from '$lib/apis';
	import {
		getImageGenerationModels,
		getImageGenerationConfig,
		updateConfig,
		verifyConfigUrl
	} from '$lib/apis/images';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import SensitiveInput from '$lib/components/common/SensitiveInput.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Textarea from '$lib/components/common/Textarea.svelte';
	import CodeEditorModal from '$lib/components/common/CodeEditorModal.svelte';
	const dispatch = createEventDispatcher();
	const i18n = getContext('i18n');

	let loading = false;
	let models = null;
	let config = null;

	let showComfyUIWorkflowEditor = false;
	let showComfyUIEditWorkflowEditor = false;

	const REQUIRED_WORKFLOW_NODES = [
		{ type: 'prompt', key: 'text', node_ids: '' },
		{ type: 'model', key: 'ckpt_name', node_ids: '' },
		{ type: 'width', key: 'width', node_ids: '' },
		{ type: 'height', key: 'height', node_ids: '' }
	];
	const REQUIRED_EDIT_WORKFLOW_NODES = [
		{ type: 'image', key: 'image', node_ids: '' },
		{ type: 'prompt', key: 'prompt', node_ids: '' },
		{ type: 'model', key: 'unet_name', node_ids: '' },
		{ type: 'width', key: 'width', node_ids: '' },
		{ type: 'height', key: 'height', node_ids: '' }
	];

	const getModels = async () => {
		models = await getImageGenerationModels(localStorage.token).catch((e) => {
			toast.error(`${e}`);
			return null;
		});
	};

	const updateConfigHandler = async () => {
		if (
				config.IMAGE_GENERATION_ENGINE === 'automatic1111' &&
				!config.AUTOMATIC1111_BASE_URL
		) {
			toast.error($i18n.t('AUTOMATIC1111 Base URL is required.'));
			return;
		}
		if (config.IMAGE_GENERATION_ENGINE === 'comfyui' && !config.COMFYUI_BASE_URL) {
			toast.error($i18n.t('ComfyUI Base URL is required.'));
			return;
		}
		if (config.IMAGE_GENERATION_ENGINE === 'magic' && !config.IMAGES_MAGIC_API_BASE_URL) {
			toast.error($i18n.t('MAGIC Base URL is required.'));
			return;
		}
		const res = await updateConfig(localStorage.token, config).catch((e) => {
			toast.error(`${e}`);
			return null;
		});
		if (res) {
			config = res;
			if (config.ENABLE_IMAGE_GENERATION) {
				backendConfig.set(await getBackendConfig());
				getModels();
			}
		}
	};

	onMount(async () => {
		loading = true;
		const res = await getImageGenerationConfig(localStorage.token).catch((e) => {
			toast.error(`${e}`);
			return null;
		});
		if (res) config = res;
		if (config.ENABLE_IMAGE_GENERATION) getModels();
		loading = false;
	});
</script>

<form
		class="w-full max-w-2xl mx-auto"
		on:submit|preventDefault={() => updateConfigHandler()}
>
	<div class="fixed top-2 right-3 z-40">
		<button
				class="btn btn-primary px-3 py-2 text-sm"
				type="button"
				on:click={() => dispatch('close')}
		>
			{$i18n.t('Close')}
		</button>
	</div>

	<!-- === Enable image generation switch === -->
	<div class="mb-3">
		<div class="flex justify-between items-center">
			<div class="text-lg font-semibold">{$i18n.t('Images')}</div>
			<label class="flex items-center gap-2">
				<span class="text-xs">{$i18n.t('Enable Image Generation')}</span>
				<Switch bind:enabled={config.ENABLE_IMAGE_GENERATION} />
			</label>
		</div>
	</div>

	<!-- === Image Generation === -->
	<div class="space-y-3 opacity-100">
		<h3 class="text-base font-medium">{$i18n.t('Generate Image')}</h3>
		<hr class="border-gray-200 dark:border-gray-800" />

		<!-- Engine selector -->
		<div class="flex justify-between items-center">
			<span class="text-xs">{$i18n.t('Image Generation Engine')}</span>
			<select
					class="dark:bg-gray-900 pr-6 rounded-sm text-xs"
					bind:value={config.IMAGE_GENERATION_ENGINE}
			>
				<option value="openai">OpenAI</option>
				<option value="comfyui">ComfyUI</option>
				<option value="automatic1111">Automatic1111</option>
				<option value="magic">MAGIC (Remote OpenWebUI)</option>
				<option value="gemini">Gemini</option>
			</select>
		</div>

		<!-- MAGIC settings -->
		{#if config?.IMAGE_GENERATION_ENGINE === 'magic'}
			<div class="space-y-2">
				<div class="flex justify-between items-center">
					<span class="text-xs">{$i18n.t('MAGIC Base URL')}</span>
					<input
							class="text-right bg-transparent border-b border-gray-300 text-sm"
							bind:value={config.IMAGES_MAGIC_API_BASE_URL}
							placeholder="https://openwebui.example.com"
					/>
				</div>
				<div class="flex justify-between items-center">
					<span class="text-xs">{$i18n.t('MAGIC API Key')}</span>
					<SensitiveInput
							inputClassName="text-right w-52"
							bind:value={config.IMAGES_MAGIC_API_KEY}
							placeholder="(optional)"
					/>
				</div>
			</div>
		{/if}
	</div>

	<!-- === Image Edit === -->
	<div class="mt-6">
		<h3 class="text-base font-medium">{$i18n.t('Edit Image')}</h3>
		<hr class="border-gray-200 dark:border-gray-800" />

		<div class="flex justify-between items-center mb-2">
			<span class="text-xs">{$i18n.t('Enable Image Edit')}</span>
			<Switch bind:enabled={config.ENABLE_IMAGE_EDIT} />
		</div>

		<div class="flex justify-between items-center">
			<span class="text-xs">{$i18n.t('Image Edit Engine')}</span>
			<select
					class="dark:bg-gray-900 pr-6 rounded-sm text-xs"
					bind:value={config.IMAGE_EDIT_ENGINE}
			>
				<option value="openai">OpenAI</option>
				<option value="comfyui">ComfyUI</option>
				<option value="magic">MAGIC (Remote OpenWebUI)</option>
				<option value="gemini">Gemini</option>
			</select>
		</div>

		<!-- MAGIC Edit settings -->
		{#if config?.IMAGE_EDIT_ENGINE === 'magic'}
			<div class="space-y-2 mt-2">
				<div class="flex justify-between items-center">
					<span class="text-xs">{$i18n.t('MAGIC Edit Base URL')}</span>
					<input
							class="text-right bg-transparent border-b border-gray-300 text-sm"
							bind:value={config.IMAGES_EDIT_MAGIC_API_BASE_URL}
							placeholder="https://openwebui.example.com"
					/>
				</div>
				<div class="flex justify-between items-center">
					<span class="text-xs">{$i18n.t('MAGIC Edit API Key')}</span>
					<SensitiveInput
							inputClassName="text-right w-52"
							bind:value={config.IMAGES_EDIT_MAGIC_API_KEY}
							placeholder="(optional)"
					/>
				</div>
			</div>
		{/if}
	</div>

	<div class="flex justify-end mt-6">
		<button class="btn btn-primary px-4 py-2 text-sm" type="submit">
			{$i18n.t('Save')}
			{#if loading}<Spinner class="ml-2" />{/if}
		</button>
	</div>
</form>
