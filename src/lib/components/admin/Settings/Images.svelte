<script lang="ts">
	import { toast } from 'svelte-sonner';

	import { createEventDispatcher, onMount, getContext } from 'svelte';
	import { config as backendConfig, user } from '$lib/stores';

	import { getBackendConfig } from '$lib/apis';
	import {
		getImageGenerationModels,
		getImageGenerationConfig,
		updateImageGenerationConfig,
		getConfig,
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
	let REQUIRED_WORKFLOW_NODES = [
		{
			type: 'prompt',
			key: 'text',
			node_ids: ''
		},
		{
			type: 'model',
			key: 'ckpt_name',
			node_ids: ''
		},
		{
			type: 'width',
			key: 'width',
			node_ids: ''
		},
		{
			type: 'height',
			key: 'height',
			node_ids: ''
		}
	];

	let showComfyUIEditWorkflowEditor = false;
	let REQUIRED_EDIT_WORKFLOW_NODES = [
		{
			type: 'image',
			key: 'image',
			node_ids: ''
		},
		{
			type: 'prompt',
			key: 'prompt',
			node_ids: ''
		},
		{
			type: 'model',
			key: 'unet_name',
			node_ids: ''
		},
		{
			type: 'width',
			key: 'width',
			node_ids: ''
		},
		{
			type: 'height',
			key: 'height',
			node_ids: ''
		}
	];

	const getModels = async () => {
		models = await getImageGenerationModels(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
	};

	const updateConfigHandler = async () => {
		if (
				config.IMAGE_GENERATION_ENGINE === 'automatic1111' &&
				config.AUTOMATIC1111_BASE_URL === ''
		) {
			toast.error($i18n.t('AUTOMATIC1111 Base URL is required.'));
			config.ENABLE_IMAGE_GENERATION = false;

			return null;
		} else if (config.IMAGE_GENERATION_ENGINE === 'comfyui' && config.COMFYUI_BASE_URL === '') {
			toast.error($i18n.t('ComfyUI Base URL is required.'));
			config.ENABLE_IMAGE_GENERATION = false;

			return null;
		} else if (config.IMAGE_GENERATION_ENGINE === 'magic' && config.IMAGES_MAGIC_API_BASE_URL === '') {
			toast.error($i18n.t('MAGIC Base URL is required.'));
			config.ENABLE_IMAGE_GENERATION = false;

			return null;
		} else if (config.IMAGE_GENERATION_ENGINE === 'openai' && config.IMAGES_OPENAI_API_KEY === '') {
			toast.error($i18n.t('OpenAI API Key is required.'));
			config.ENABLE_IMAGE_GENERATION = false;

			return null;
		} else if (config.IMAGE_GENERATION_ENGINE === 'gemini' && config.IMAGES_GEMINI_API_KEY === '') {
			toast.error($i18n.t('Gemini API Key is required.'));
			config.ENABLE_IMAGE_GENERATION = false;

			return null;
		}

		const res = await updateConfig(localStorage.token, {
			...config,
			AUTOMATIC1111_PARAMS:
					typeof config.AUTOMATIC1111_PARAMS === 'string' && config.AUTOMATIC1111_PARAMS.trim() !== ''
							? JSON.parse(config.AUTOMATIC1111_PARAMS)
							: {},
			IMAGES_OPENAI_API_PARAMS:
					typeof config.IMAGES_OPENAI_API_PARAMS === 'string' &&
					config.IMAGES_OPENAI_API_PARAMS.trim() !== ''
							? JSON.parse(config.IMAGES_OPENAI_API_PARAMS)
							: {}
		}).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			if (res.ENABLE_IMAGE_GENERATION) {
				backendConfig.set(await getBackendConfig());
				getModels();
			}

			return res;
		}

		return null;
	};

	const validateJSON = (json) => {
		try {
			const obj = JSON.parse(json);

			if (obj && typeof obj === 'object') {
				return true;
			}
		} catch (e) {}
		return false;
	};

	const saveHandler = async () => {
		loading = true;

		if (config?.IMAGE_EDIT_ENGINE === 'magic' && !config.IMAGES_EDIT_MAGIC_API_BASE_URL) {
			toast.error($i18n.t('MAGIC Edit Base URL is required.'));
			loading = false;
			return;
		}

		if (config?.COMFYUI_WORKFLOW) {
			if (!validateJSON(config?.COMFYUI_WORKFLOW)) {
				toast.error($i18n.t('Invalid JSON format for ComfyUI Workflow.'));
				loading = false;
				return;
			}

			config.COMFYUI_WORKFLOW_NODES = REQUIRED_WORKFLOW_NODES.map((node) => {
				return {
					type: node.type,
					key: node.key,
					node_ids:
							node.node_ids.trim() === '' ? [] : node.node_ids.split(',').map((id) => id.trim())
				};
			});
		}

		if (config?.IMAGES_EDIT_COMFYUI_WORKFLOW) {
			if (!validateJSON(config?.IMAGES_EDIT_COMFYUI_WORKFLOW)) {
				toast.error($i18n.t('Invalid JSON format for ComfyUI Edit Workflow.'));
				loading = false;
				return;
			}

			config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES = REQUIRED_EDIT_WORKFLOW_NODES.map((node) => {
				return {
					type: node.type,
					key: node.key,
					node_ids:
							node.node_ids.trim() === '' ? [] : node.node_ids.split(',').map((id) => id.trim())
				};
			});
		}

		const res = await updateConfigHandler();

		if (res) {
			config = res;
		}

		loading = false;
	};

	onMount(async () => {
		loading = true;

		const res = await getImageGenerationConfig(localStorage.token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		if (res) {
			config = res;
		}

		if (config.ENABLE_IMAGE_GENERATION) {
			getModels();
		}

		if (config.COMFYUI_WORKFLOW) {
			try {
				config.COMFYUI_WORKFLOW_NODES =
						config.COMFYUI_WORKFLOW_NODES.length > 0
								? config.COMFYUI_WORKFLOW_NODES
								: REQUIRED_WORKFLOW_NODES.map((node) => {
									return {
										type: node.type,
										key: node.key,
										node_ids: []
									};
								});
			} catch {
				config.COMFYUI_WORKFLOW_NODES = REQUIRED_WORKFLOW_NODES.map((node) => {
					return {
						type: node.type,
						key: node.key,
						node_ids: []
					};
				});
			}
		}

		if (config.IMAGES_EDIT_COMFYUI_WORKFLOW) {
			try {
				config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES =
						config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES.length > 0
								? config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES
								: REQUIRED_EDIT_WORKFLOW_NODES.map((node) => {
									return {
										type: node.type,
										key: node.key,
										node_ids: []
									};
								});
			} catch {
				config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES = REQUIRED_EDIT_WORKFLOW_NODES.map((node) => {
					return {
						type: node.type,
						key: node.key,
						node_ids: []
					};
				});
			}
		}

		loading = false;
	});
</script>

<form
		class="w-full max-w-2xl mx-auto"
		on:submit|preventDefault={() => {
		saveHandler();
	}}
>
	<div class="fixed top-2 right-3 z-40">
		<button
				class="btn btn-primary px-3 py-2 text-sm"
				class:bg-gray-600={loading}
				class:hover:bg-gray-800={loading}
				class:cursor-not-allowed={loading}
				type="button"
				on:click={() => {
				dispatch('close');
			}}
		>
			{$i18n.t('Close')}
		</button>
	</div>

	<div class=" mb-3">
		<div class="flex justify-between items-center">
			<div class="text-lg font-semibold tracking-tight">{$i18n.t('Images')}</div>
			<div class="">
				<label
						class="relative inline-flex items-center gap-3 cursor-pointer text-xs rounded-full px-3.5 py-2 bg-gray-50 dark:bg-gray-975 border border-gray-100 dark:border-gray-850"
						for="advanced-mode"
				>
					<div
							class=" text-xs font-medium flex gap-1 items-center"
							class:text-gray-500={config !== null && !config.ENABLE_IMAGE_GENERATION}
							class:text-blue-600={config !== null && config.ENABLE_IMAGE_GENERATION}
					>
						<span>{$i18n.t('Enable Image Generation')}</span>
					</div>

					<Switch
							id="advanced-mode"
							disabled={$user.role === 'user'}
							bind:enabled={config && config.ENABLE_IMAGE_GENERATION}
							on:click={async () => {
							if (config) {
								config.ENABLE_IMAGE_GENERATION = !config.ENABLE_IMAGE_GENERATION;

								if (config.ENABLE_IMAGE_GENERATION === false) {
									config.ENABLE_IMAGE_PROMPT_GENERATION = false;
								}

								if ($user.role === 'admin') {
									const res = await updateConfigHandler();

									if (res) {
										config = res;
									}
								}
							}
						}}
					/>
				</label>
			</div>
		</div>
	</div>

	<div class={config.ENABLE_IMAGE_GENERATION ? ' opacity-100' : ' opacity-70'}>
		<div class=" space-y-4">
			<div class="">
				<div class="">
					<div class=" mb-3">
						<div class=" mt-0.5 mb-2.5 text-base font-medium">{$i18n.t('Generate Image')}</div>

						<hr class=" border-gray-100 dark:border-gray-850 my-2" />

						<div class="mb-2.5">
							<div class="flex w-full justify-between items-center">
								<div class="text-xs pr-2">
									<div class="">
										{$i18n.t('Image Generation Engine')}
									</div>
								</div>

								<select
										class=" dark:bg-gray-900 w-fit pr-8 cursor-pointer rounded-sm px-2 text-xs bg-transparent outline-hidden text-right"
										bind:value={config.IMAGE_GENERATION_ENGINE}
										placeholder={$i18n.t('Select Engine')}
								>
									<option value="openai">{$i18n.t('Default (Open AI)')}</option>
									<option value="comfyui">{$i18n.t('ComfyUI')}</option>
									<option value="automatic1111">{$i18n.t('Automatic1111')}</option>
									<option value="magic">{$i18n.t('MAGIC (Remote Open WebUI)')}</option>
									<option value="gemini">{$i18n.t('Gemini')}</option>
								</select>
							</div>
						</div>

						{#if config?.IMAGE_GENERATION_ENGINE === 'openai'}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Base URL')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={config.IMAGES_OPENAI_API_BASE_URL}
											/>
										</div>

										<div class=" flex items-center text-[11px] gap-2">
											<div class="">
												{$i18n.t('Defaults to {{url}}', {
													replace: {
														url: 'https://api.openai.com/v1'
													}
												})}
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class=" flex gap-1 items-center">
											<div class="">
												{$i18n.t('OpenAI API Key')}
											</div>
										</div>
									</div>

									<div class="flex w-full flex-col gap-y-1">
										<div class="flex w-full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w-full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_OPENAI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w-full justify-end text-[11px]">
											<div class="text-gray-500">
												{$i18n.t(
														"Read your assistant's responses, learn your preferences, and personalize your experience."
												)}
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">{config.IMAGES_OPENAI_API_VERSION === '' ? 'Model' : 'Deployment Name'}</div>
									</div>

									{#if config.IMAGES_OPENAI_API_VERSION === ''}
										<div class="flex w-full">
											<div class="flex-1">
												<select
														class=" w-full bg-transparent text-sm outline-hidden text-right"
														bind:value={config.IMAGE_GENERATION_MODEL}
												>
													{#each models as model (model.id)}
														<option value={model.id}>{model.name}</option>
													{/each}
												</select>
											</div>
										</div>
									{:else}
										<div class="ms-auto">
											<div class="flex w-full">
												<div class="flex-1">
													<input
															class="w-fit ml-auto max-w-full text-sm bg-transparent outline-hidden text-right"
															placeholder={$i18n.t('Enter Deployment Name')}
															bind:value={config.IMAGE_GENERATION_MODEL}
													/>
												</div>
											</div>
										</div>
									{/if}
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Version')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Version')}
													bind:value={config.IMAGES_OPENAI_API_VERSION}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Additional Parameters')}
										</div>
									</div>
								</div>
								<div class="mt-1.5 flex w-full">
									<div class="flex-1 mr-2">
									<Textarea
											className="rounded-lg w-full py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden"
											bind:value={config.IMAGES_OPENAI_API_PARAMS}
											placeholder={$i18n.t('Enter additional parameters in JSON format')}
											minSize={100}
									/>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'magic'}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC Base URL')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1 mr-2">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('Enter URL (e.g. https://openwebui.example.com)')}
													bind:value={config.IMAGES_MAGIC_API_BASE_URL}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC API Key')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1">
											<SensitiveInput
													inputClassName="text-right w-full"
													placeholder={$i18n.t('API Key (optional)')}
													bind:value={config.IMAGES_MAGIC_API_KEY}
													required={false}
											/>
										</div>
									</div>
								</div>
							</div>
						{:else if (config?.IMAGE_GENERATION_ENGINE ?? 'automatic1111') === 'automatic1111'}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('AUTOMATIC1111 Base URL')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1 mr-2">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('Enter URL (e.g. http://127.0.0.1:7860/)')}
													bind:value={config.AUTOMATIC1111_BASE_URL}
											/>
										</div>
										<button
												class="  transition"
												type="button"
												aria-label="verify connection"
												on:click={async () => {
											await updateConfigHandler();
											const res = await verifyConfigUrl(localStorage.token).catch((error) => {
												toast.error(`${error}`);
												return null;
											});

											if (res) {
												toast.success($i18n.t('Server connection verified'));
											}
										}}
										>
											<svg
													xmlns="http://www.w3.org/2000/svg"
													viewBox="0 0 20 20"
													fill="currentColor"
													class="w-4 h-4"
											>
												<path
														fill-rule="evenodd"
														d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.312-.31...6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
														clip-rule="evenodd"
												/>
											</svg>
										</button>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Steps')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter steps (e.g. 50)')} placement="top-start">
										<input
												type="number"
												min="1"
												class=" text-right text-sm bg-transparent outline-hidden max-w-full w-16"
												placeholder={$i18n.t('50')}
												bind:value={config.IMAGE_STEPS}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Image Size')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter image size (e.g. 512x512)')} placement="top-start">
										<input
												class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
												placeholder={$i18n.t('Enter Image Size (e.g. 512x512)')}
												bind:value={config.IMAGE_SIZE}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Additional Parameters')}
										</div>
									</div>
								</div>
								<div class="mt-1.5 flex w-full">
									<div class="flex-1 mr-2">
									<Textarea
											className="rounded-lg w-full py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden"
											bind:value={config.AUTOMATIC1111_PARAMS}
											placeholder={$i18n.t('Enter additional parameters in JSON format')}
											minSize={100}
									/>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'comfyui'}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('ComfyUI Base URL')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1 mr-2">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('Enter URL (e.g. http://127.0.0.1:7860/)')}
													bind:value={config.COMFYUI_BASE_URL}
											/>
										</div>
										<button
												class="  transition"
												type="button"
												aria-label="verify connection"
												on:click={async () => {
											await updateConfigHandler();
											const res = await verifyConfigUrl(localStorage.token).catch((error) => {
												toast.error(`${error}`);
												return null;
											});

											if (res) {
												toast.success($i18n.t('Server connection verified'));
											}
										}}
										>
											<svg
													xmlns="http://www.w3.org/2000/svg"
													viewBox="0 0 20 20"
													fill="currentColor"
													class="w-4 h-4"
											>
												<path
														fill-rule="evenodd"
														d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.312-.31...6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
														clip-rule="evenodd"
												/>
											</svg>
										</button>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('ComfyUI API Key')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1">
											<SensitiveInput
													inputClassName="text-right w-full"
													placeholder={$i18n.t('API Key')}
													bind:value={config.COMFYUI_API_KEY}
													required={false}
											/>
										</div>
									</div>
								</div>
							</div>

							{#if models}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2">
											<div class="shrink-0">
												{$i18n.t('Model')}
											</div>
										</div>

										<Tooltip content={$i18n.t('Enter Model ID')} placement="top-start">
											<input
													list="model-list"
													class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
													placeholder={$i18n.t('Enter Model ID')}
													bind:value={config.IMAGE_GENERATION_MODEL}
											/>
										</Tooltip>

										<datalist id="model-list">
											{#each models as model (model.id)}
												<option value={model.id}>{model.name}</option>
											{/each}
										</datalist>
									</div>
								</div>
							{/if}

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Image Size')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter image size (e.g. 512x512)')} placement="top-start">
										<input
												class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
												placeholder={$i18n.t('Enter Image Size (e.g. 512x512)')}
												bind:value={config.IMAGE_SIZE}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Steps')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter steps (e.g. 50)')} placement="top-start">
										<input
												type="number"
												min="1"
												class=" text-right text-sm bg-transparent outline-hidden max-w-full w-16"
												placeholder={$i18n.t('50')}
												bind:value={config.IMAGE_STEPS}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<input
										id="upload-comfyui-workflow-input"
										hidden
										type="file"
										accept=".json"
										on:change={(e) => {
									const file = e.target.files[0];
									const reader = new FileReader();

									reader.onload = (e) => {
										config.COMFYUI_WORKFLOW = e.target.result;
										e.target.value = null;
									};

									reader.readAsText(file);
								}}
								/>
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('ComfyUI Workflow')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1 flex flex-col gap-y-1">
											<div class="flex w-full">
												<div class="flex-1 flex justify-end items-center gap-2">
													<div class="text-[11px] text-gray-500 dark:text-gray-400">
														{$i18n.t(
																'Make sure to export a workflow.json file as API format from ComfyUI.'
														)}
													</div>
													<div
															class=" inline-flex items-center align-middle justify-center w-fit rounded-full px-4 h-7 text-xs border border-solid border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-850 cursor-pointer"
															on:click={() => {
														const upload = document.getElementById(
															'upload-comfyui-workflow-input'
														) as HTMLInputElement;

														upload?.click();
													}}
													>
														{$i18n.t('Upload Workflow')}
													</div>
												</div>
											</div>

											<div class="flex w-full justify-end">
												<div class="flex w-fit flex-wrap justify-end gap-1.5 text-[11px]">
													{#each REQUIRED_WORKFLOW_NODES as node}
														<div class=" inline-flex items-center gap-1">
															<div
																	class=" inline-flex items-center align-middle justify-center w-fit rounded-full px-2 h-6 text-[11px] border border-solid border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-850"
															>
																{#if node.type === 'prompt'}
																	{$i18n.t('Prompt Node')}
																{:else if node.type === 'model'}
																	{$i18n.t('Model Node')}
																{:else if node.type === 'width'}
																	{$i18n.t('Width Node')}
																{:else if node.type === 'height'}
																	{$i18n.t('Height Node')}
																{/if}
															</div>

															<Tooltip
																	content={$i18n.t('Enter node IDs (e.g. 1,2)')}
																	placement="top-start"
															>
																<input
																		class=" text-right text-xs bg-transparent outline-hidden max-w-32 w-full"
																		placeholder={$i18n.t('Enter Node IDs (e.g. 1,2)')}
																		bind:value={node.node_ids}
																/>
															</Tooltip>
														</div>
													{/each}
												</div>
											</div>

											<div class="flex w-full justify-end text-[11px]">
												<div class="text-gray-500">
													{$i18n.t(
															'Specify which nodes in the workflow correspond to the prompt, model, width, and height.'
													)}
												</div>
											</div>

											{#if config?.COMFYUI_WORKFLOW}
												<div class="flex w-full justify-end text-[11px]">
													<div>
														<button
																class=" hover:bg-gray-50 dark:hover:bg-gray-850 transition text-gray-400 hover:text-foreground text-xs px-2 py-1 rounded border border-dashed border-gray-200 dark:border-gray-700"
																type="button"
																on:click={() => {
															showComfyUIWorkflowEditor = true;
														}}
														>
															<div class="flex items-center gap-1">
																<span>{$i18n.t('View/Edit Workflow')}</span>
															</div>
														</button>
													</div>
												</div>
											{/if}
										</div>
									</div>
								</div>
							</div>

							{#if config.COMFYUI_WORKFLOW}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('ComfyUI Workflow Nodes')}
											</div>
										</div>
									</div>

									<div class="mt-1 text-[11px] text-gray-500">
									<pre class="bg-gray-50 dark:bg-gray-950 rounded p-2 overflow-auto max-h-40 whitespace-pre-wrap break-words">
										<code>{JSON.stringify(config.COMFYUI_WORKFLOW_NODES, null, 2)}</code>
									</pre>
									</div>
								</div>
							{/if}
						{:else if config?.IMAGE_GENERATION_ENGINE === 'gemini'}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Gemini Base URL')}
										</div>
									</div>

									<div class="flex w-full">
										<div class="flex-1">
											<input
													class="w-full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={config.IMAGES_GEMINI_API_BASE_URL}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class=" flex gap-1 items-center">
											<div class="">
												{$i18n.t('Gemini API Key')}
											</div>
										</div>
									</div>

									<div class="flex w-full flex-col gap-y-1">
										<div class="flex w-full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w-full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_GEMINI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w-full justify-end text-[11px]">
											<div class="text-gray-500">
												{$i18n.t(
														"Read your assistant's responses, learn your preferences, and personalize your experience."
												)}
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="shrink-0">
											{$i18n.t('Model')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter Model ID')} placement="top-start">
										<input
												class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
												placeholder={$i18n.t('Enter Model ID')}
												bind:value={config.IMAGE_GENERATION_MODEL}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Image Generation Endpoint Method')}
										</div>
									</div>

									<select
											class=" dark:bg-gray-900 w-fit pr-8 cursor-pointer rounded-sm px-2 text-xs bg-transparent outline-hidden text-right"
											bind:value={config.IMAGES_GEMINI_ENDPOINT_METHOD}
											placeholder={$i18n.t('Select Method')}
									>
										<option value="predict">predict</option>
										<option value="generateContent">generateContent</option>
									</select>
								</div>
							</div>
						{/if}
					</div>

					<div class="mb-3">
						<div class=" mt-0.5 mb-2.5 text-base font-medium">{$i18n.t('Edit Image')}</div>

						<hr class=" border-gray-100 dark:border-gray-850 my-2" />

						<div class=" space-y-1.5">
							<div class="flex w-full justify-between items-center">
								<div class="text-xs pr-2">
									<div class="">
										{$i18n.t('Enable Image Edit')}
									</div>
								</div>

								<Switch
										disabled={$user.role === 'user'}
										bind:enabled={config && config.ENABLE_IMAGE_EDIT}
										on:click={async () => {
									if (config) {
										config.ENABLE_IMAGE_EDIT = !config.ENABLE_IMAGE_EDIT;

										if (config.ENABLE_IMAGE_EDIT === false) {
											config.IMAGE_EDIT_ENGINE = 'openai';
										}

										if ($user.role === 'admin') {
											const res = await updateConfigHandler();

											if (res) {
												config = res;
											}
										}
									}
								}}
								/>
							</div>
						</div>

						{#if config.ENABLE_IMAGE_EDIT}
							<hr class=" border-gray-100 dark:border-gray-850 my-2" />

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('Image Edit Engine')}
										</div>
									</div>

									<select
											class=" dark:bg-gray-900 w-fit pr-8 cursor-pointer rounded-sm px-2 text-xs bg-transparent outline-hidden text-right"
											bind:value={config.IMAGE_EDIT_ENGINE}
											placeholder={$i18n.t('Select Engine')}
									>
										<option value="openai">{$i18n.t('Default (Open AI)')}</option>
										<option value="comfyui">{$i18n.t('ComfyUI')}</option>
										<option value="magic">{$i18n.t('MAGIC (Remote Open WebUI)')}</option>
										<option value="gemini">{$i18n.t('Gemini')}</option>
									</select>
								</div>
							</div>

							{#if config?.IMAGE_EDIT_ENGINE === 'openai'}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('OpenAI API Base URL')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1">
												<input
														class="w-full text-sm bg-transparent outline-hidden text-right"
														placeholder={$i18n.t('API Base URL')}
														bind:value={config.IMAGES_EDIT_OPENAI_API_BASE_URL}
												/>
											</div>

											<div class=" flex items-center text-[11px] gap-2">
												<div class="">
													{$i18n.t('Defaults to {{url}}', {
														replace: {
															url: 'https://api.openai.com/v1'
														}
													})}
												</div>
											</div>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class=" flex gap-1 items-center">
												<div class="">
													{$i18n.t('OpenAI API Key')}
												</div>
											</div>
										</div>

										<div class="flex w-full flex-col gap-y-1">
											<div class="flex w-full">
												<div class="flex-1">
													<SensitiveInput
															inputClassName="text-right w-full"
															placeholder={$i18n.t('API Key')}
															bind:value={config.IMAGES_EDIT_OPENAI_API_KEY}
															required={false}
													/>
												</div>
											</div>

											<div class="flex w-full justify-end text-[11px]">
												<div class="text-gray-500">
													{$i18n.t(
															"Read your assistant's responses, learn your preferences, and personalize your experience."
													)}
												</div>
											</div>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('OpenAI API Version')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1">
												<input
														class="w-full text-sm bg-transparent outline-hidden text-right"
														placeholder={$i18n.t('API Version')}
														bind:value={config.IMAGES_EDIT_OPENAI_API_VERSION}
												/>
											</div>
										</div>
									</div>
								</div>
							{:else if config?.IMAGE_EDIT_ENGINE === 'magic'}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('MAGIC Edit Base URL')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1 mr-2">
												<input
														class="w-full text-sm bg-transparent outline-hidden text-right"
														placeholder={$i18n.t('Enter URL (e.g. https://openwebui.example.com)')}
														bind:value={config.IMAGES_EDIT_MAGIC_API_BASE_URL}
												/>
											</div>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('MAGIC Edit API Key')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w-full"
														placeholder={$i18n.t('API Key (optional)')}
														bind:value={config.IMAGES_EDIT_MAGIC_API_KEY}
														required={false}
												/>
											</div>
										</div>
									</div>
								</div>
							{:else if config?.IMAGE_EDIT_ENGINE === 'comfyui'}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('ComfyUI Base URL')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1 mr-2">
												<input
														class="w-full text-sm bg-transparent outline-hidden text-right"
														placeholder={$i18n.t('Enter URL (e.g. http://127.0.0.1:7860/)')}
														bind:value={config.IMAGES_EDIT_COMFYUI_BASE_URL}
												/>
											</div>
											<button
													class="  transition"
													type="button"
													aria-label="verify connection"
													on:click={async () => {
											await updateConfigHandler();
											const res = await verifyConfigUrl(localStorage.token).catch((error) => {
												toast.error(`${error}`);
												return null;
											});

											if (res) {
												toast.success($i18n.t('Server connection verified'));
											}
										}}
											>
												<svg
														xmlns="http://www.w3.org/2000/svg"
														viewBox="0 0 20 20"
														fill="currentColor"
														class="w-4 h-4"
												>
													<path
															fill-rule="evenodd"
															d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.312-.31...6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
															clip-rule="evenodd"
													/>
												</svg>
											</button>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('ComfyUI API Key')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w-full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_EDIT_COMFYUI_API_KEY}
														required={false}
												/>
											</div>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2">
											<div class="shrink-0">
												{$i18n.t('Model')}
											</div>
										</div>

										<Tooltip content={$i18n.t('Enter Model ID')} placement="top-start">
											<input
													class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
													placeholder={$i18n.t('Enter Model ID')}
													bind:value={config.IMAGE_EDIT_MODEL}
											/>
										</Tooltip>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('Image Size')}
											</div>
										</div>

										<Tooltip content={$i18n.t('Enter image size (e.g. 512x512)')} placement="top-start">
											<input
													class=" text-right text-sm bg-transparent outline-hidden max-w-full w-52"
													placeholder={$i18n.t('Enter Image Size (e.g. 512x512)')}
													bind:value={config.IMAGE_EDIT_SIZE}
											/>
										</Tooltip>
									</div>
								</div>

								<div class="mb-2.5">
									<input
											id="upload-comfyui-edit-workflow-input"
											hidden
											type="file"
											accept=".json"
											on:change={(e) => {
									const file = e.target.files[0];
									const reader = new FileReader();

									reader.onload = (e) => {
										config.IMAGES_EDIT_COMFYUI_WORKFLOW = e.target.result;
										e.target.value = null;
									};

									reader.readAsText(file);
								}}
									/>
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('ComfyUI Edit Workflow')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1 flex flex-col gap-y-1">
												<div class="flex w-full">
													<div class="flex-1 flex justify-end items-center gap-2">
														<div class="text-[11px] text-gray-500 dark:text-gray-400">
															{$i18n.t(
																	'Make sure to export a workflow.json file as API format from ComfyUI.'
															)}
														</div>
														<div
																class=" inline-flex items-center align-middle justify-center w-fit rounded-full px-4 h-7 text-xs border border-solid border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-850 cursor-pointer"
																on:click={() => {
														const upload = document.getElementById(
															'upload-comfyui-edit-workflow-input'
														) as HTMLInputElement;

														upload?.click();
													}}
														>
															{$i18n.t('Upload Workflow')}
														</div>
													</div>
												</div>

												<div class="flex w-full justify-end">
													<div class="flex w-fit flex-wrap justify-end gap-1.5 text-[11px]">
														{#each REQUIRED_EDIT_WORKFLOW_NODES as node}
															<div class=" inline-flex items-center gap-1">
																<div
																		class=" inline-flex items-center align-middle justify-center w-fit rounded-full px-2 h-6 text-[11px] border border-solid border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-850"
																>
																	{#if node.type === 'image'}
																		{$i18n.t('Image Node')}
																	{:else if node.type === 'prompt'}
																		{$i18n.t('Prompt Node')}
																	{:else if node.type === 'model'}
																		{$i18n.t('Model Node')}
																	{:else if node.type === 'width'}
																		{$i18n.t('Width Node')}
																	{:else if node.type === 'height'}
																		{$i18n.t('Height Node')}
																	{/if}
																</div>

																<Tooltip
																		content={$i18n.t('Enter node IDs (e.g. 1,2)')}
																		placement="top-start"
																>
																	<input
																			class=" text-right text-xs bg-transparent outline-hidden max-w-32 w-full"
																			placeholder={$i18n.t('Enter Node IDs (e.g. 1,2)')}
																			bind:value={node.node_ids}
																	/>
																</Tooltip>
															</div>
														{/each}
													</div>
												</div>

												<div class="flex w-full justify-end text-[11px]">
													<div class="text-gray-500">
														{$i18n.t(
																'Specify which nodes in the workflow correspond to the image, prompt, model, width, and height.'
														)}
													</div>
												</div>

												{#if config?.IMAGES_EDIT_COMFYUI_WORKFLOW}
													<div class="flex w-full justify-end text-[11px]">
														<div>
															<button
																	class=" hover:bg-gray-50 dark:hover:bg-gray-850 transition text-gray-400 hover:text-foreground text-xs px-2 py-1 rounded border border-dashed border-gray-200 dark:border-gray-700"
																	type="button"
																	on:click={() => {
															showComfyUIEditWorkflowEditor = true;
														}}
															>
																<div class="flex items-center gap-1">
																	<span>{$i18n.t('View/Edit Workflow')}</span>
																</div>
															</button>
														</div>
													</div>
												{/if}
											</div>
										</div>
									</div>
								</div>

								{#if config.IMAGES_EDIT_COMFYUI_WORKFLOW}
									<div class="mb-2.5">
										<div class="flex w-full justify-between items-center">
											<div class="text-xs pr-2 shrink-0">
												<div class="">
													{$i18n.t('ComfyUI Edit Workflow Nodes')}
												</div>
											</div>
										</div>

										<div class="mt-1 text-[11px] text-gray-500">
									<pre class="bg-gray-50 dark:bg-gray-950 rounded p-2 overflow-auto max-h-40 whitespace-pre-wrap break-words">
										<code>{JSON.stringify(config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES, null, 2)}</code>
									</pre>
										</div>
									</div>
								{/if}
							{:else if config?.IMAGE_EDIT_ENGINE === 'gemini'}
								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class="">
												{$i18n.t('Gemini Base URL')}
											</div>
										</div>

										<div class="flex w-full">
											<div class="flex-1">
												<input
														class="w-full text-sm bg-transparent outline-hidden text-right"
														placeholder={$i18n.t('API Base URL')}
														bind:value={config.IMAGES_EDIT_GEMINI_API_BASE_URL}
												/>
											</div>
										</div>
									</div>
								</div>

								<div class="mb-2.5">
									<div class="flex w-full justify-between items-center">
										<div class="text-xs pr-2 shrink-0">
											<div class=" flex gap-1 items-center">
												<div class="">
													{$i18n.t('Gemini API Key')}
												</div>
											</div>
										</div>

										<div class="flex w-full flex-col gap-y-1">
											<div class="flex w-full">
												<div class="flex-1">
													<SensitiveInput
															inputClassName="text-right w-full"
															placeholder={$i18n.t('API Key')}
															bind:value={config.IMAGES_EDIT_GEMINI_API_KEY}
															required={false}
													/>
												</div>
											</div>

											<div class="flex w-full justify-end text-[11px]">
												<div class="text-gray-500">
													{$i18n.t(
															"Read your assistant's responses, learn your preferences, and personalize your experience."
													)}
												</div>
											</div>
										</div>
									</div>
								</div>
							{/if}
							</div>
						{/if}
					</div>
				</div>

				<div class="flex justify-end mt-4">
					<button
							class="btn btn-primary px-4 py-2 text-sm flex items-center"
							class:bg-gray-600={loading}
							class:hover:bg-gray-800={loading}
							class:cursor-not-allowed={loading}
							type="submit"
							disabled={loading}
					>
						{$i18n.t('Save')}

						{#if loading}
							<div class="ml-2 self-center">
								<Spinner />
							</div>
						{/if}
					</button>
				</div>

				{#if showComfyUIWorkflowEditor}
					<CodeEditorModal
							bind:show={showComfyUIWorkflowEditor}
							title={$i18n.t('ComfyUI Workflow JSON')}
							description={$i18n.t('View or edit the ComfyUI workflow JSON.')}
							language="json"
							bind:value={config.COMFYUI_WORKFLOW}
					/>
				{/if}

				{#if showComfyUIEditWorkflowEditor}
					<CodeEditorModal
							bind:show={showComfyUIEditWorkflowEditor}
							title={$i18n.t('ComfyUI Edit Workflow JSON')}
							description={$i18n.t('View or edit the ComfyUI edit workflow JSON.')}
							language="json"
							bind:value={config.IMAGES_EDIT_COMFYUI_WORKFLOW}
					/>
				{/if}
</form>
