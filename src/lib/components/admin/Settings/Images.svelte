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
		},
		{
			type: 'steps',
			key: 'steps',
			node_ids: ''
		},
		{
			type: 'cfg_scale',
			key: 'cfg_scale',
			node_ids: ''
		},
		{
			type: 'sampler_name',
			key: 'sampler_name',
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

		// 校验 MAGIC 编辑引擎的 Base URL
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
			dispatch('save');
		}

		loading = false;
	};

	onMount(async () => {
		if ($user?.role === 'admin') {
			const res = await getConfig(localStorage.token).catch((error) => {
				toast.error(`${error}`);
				return null;
			});

			if (res) {
				config = res;

				if (config?.COMFYUI_WORKFLOW) {
					try {
						config.COMFYUI_WORKFLOW = JSON.stringify(
								JSON.parse(config.COMFYUI_WORKFLOW),
								null,
								2
						);
					} catch (e) {
						console.error(e);
					}
				}

				if (config?.COMFYUI_WORKFLOW_NODES?.length === 0) {
					config.COMFYUI_WORKFLOW_NODES = REQUIRED_WORKFLOW_NODES;
				}

				if (config?.IMAGES_EDIT_COMFYUI_WORKFLOW) {
					try {
						config.IMAGES_EDIT_COMFYUI_WORKFLOW = JSON.stringify(
								JSON.parse(config.IMAGES_EDIT_COMFYUI_WORKFLOW),
								null,
								2
						);
					} catch (e) {
						console.error(e);
					}
				}

				if (config?.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES?.length === 0) {
					config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES = REQUIRED_EDIT_WORKFLOW_NODES;
				}

				if (config.AUTOMATIC1111_PARAMS && typeof config.AUTOMATIC1111_PARAMS !== 'string') {
					try {
						config.AUTOMATIC1111_PARAMS = JSON.stringify(
								config.AUTOMATIC1111_PARAMS ?? {},
								null,
								2
						);
					} catch (e) {
						console.error(e);
					}
				}

				if (config.IMAGES_OPENAI_API_PARAMS && typeof config.IMAGES_OPENAI_API_PARAMS !== 'string') {
					try {
						config.IMAGES_OPENAI_API_PARAMS = JSON.stringify(
								config.IMAGES_OPENAI_API_PARAMS ?? {},
								null,
								2
						);
					} catch (e) {
						console.error(e);
					}
				}

				config.IMAGES_OPENAI_API_PARAMS =
						typeof config.IMAGES_OPENAI_API_PARAMS === 'object'
								? JSON.stringify(config.IMAGES_OPENAI_API_PARAMS ?? {}, null, 2)
								: config.IMAGES_OPENAI_API_PARAMS;
			}

			const imageGenerationConfig = await getImageGenerationConfig(
					localStorage.token
			).catch((error) => {
				toast.error(`${error}`);
				return null;
			});

			if (imageGenerationConfig) {
				if (config) {
					config.IMAGE_GENERATION_ENGINE = imageGenerationConfig.IMAGE_GENERATION_ENGINE;
					config.IMAGE_GENERATION_MODEL = imageGenerationConfig.IMAGE_GENERATION_MODEL;
					config.IMAGE_SIZE = imageGenerationConfig.IMAGE_SIZE;
					config.IMAGE_STEPS = imageGenerationConfig.IMAGE_STEPS;
					config.IMAGE_GUIDANCE_SCALE = imageGenerationConfig.IMAGE_GUIDANCE_SCALE;
					config.IMAGE_CFG_SCALE = imageGenerationConfig.IMAGE_CFG_SCALE;
					config.IMAGE_SCHEDULER = imageGenerationConfig.IMAGE_SCHEDULER;
				}
			}

			if (config.ENABLE_IMAGE_GENERATION) {
				await getModels();
			}
		}
	});

	$: if (config?.COMFYUI_WORKFLOW_NODES?.length > 0) {
		REQUIRED_WORKFLOW_NODES = REQUIRED_WORKFLOW_NODES.map((node) => {
			const n = config.COMFYUI_WORKFLOW_NODES.find((n) => n.type === node.type) ?? node;
			return {
				type: n.type,
				key: n.key,
				node_ids: typeof n.node_ids === 'string' ? n.node_ids : n.node_ids.join(',')
			};
		});
	}

	$: if (config?.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES?.length > 0) {
		REQUIRED_EDIT_WORKFLOW_NODES = REQUIRED_EDIT_WORKFLOW_NODES.map((node) => {
			const n =
					config.IMAGES_EDIT_COMFYUI_WORKFLOW_NODES.find((n) => n.type === node.type) ?? node;
			console.debug(n);

			return {
				type: n.type,
				key: n.key,
				node_ids: typeof n.node_ids === 'string' ? n.node_ids : n.node_ids.join(',')
			};
		});
	}
</script>

<form
		class="flex flex-col h-full justify-between space-y-3 text-sm"
		on:submit|preventDefault={() => {
		saveHandler();
	}}
>
	<div class="flex flex-col space-y-2">
		<div class="flex justify-between items-center mt-1 gap-2">
			<div class="flex flex-col gap-1">
				<div class="text-base font-semibold leading-6">
					{$i18n.t('Images')}
				</div>
				<div class="text-xs text-gray-600 dark:text-gray-400">
					{$i18n.t('Control how images are generated and edited.')}
				</div>
			</div>

			<div class="flex space-x-2">
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
			<div class="flex flex-col lg:flex-row gap-8">
				<div class="flex-1 space-y-4">
					<div class="mb-3">
						<div class=" mt-0.5 mb-2.5 text-base font-medium">{$i18n.t('Create Image')}</div>

						<hr class=" border-gray-100 dark:border-gray-850 my-2" />

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
											bind:value={config.IMAGE_GENERATION_MODEL}
											placeholder={$i18n.t('Select a model')}
											required
									/>

									<datalist id="model-list">
										{#each models ?? [] as model}
											<option value={model.id}>{model.name}</option>
										{/each}
									</datalist>
								</Tooltip>
							</div>
						</div>

						<div class="mb-2.5">
							<div class="flex w-full justify-between items-center">
								<div class="text-xs pr-2">
									<div class="shrink-0">
										{$i18n.t('Image Size')}
									</div>
								</div>

								<Tooltip content={$i18n.t('Enter Image Size (e.g. 512x512)')} placement="top-start">
									<input
											class="  text-right text-sm bg-transparent outline-hidden max-w-full w-52"
											placeholder={$i18n.t('Enter Image Size (e.g. 512x512)')}
											bind:value={config.IMAGE_SIZE}
									/>
								</Tooltip>
							</div>
						</div>

						{#if ['comfyui', 'automatic1111', ''].includes(config?.IMAGE_GENERATION_ENGINE)}
							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('Steps')}
										</div>
									</div>

									<Tooltip
											content={$i18n.t('Enter Number of Steps (e.g. 50)')}
											placement="top-start"
									>
										<input
												class=" text-right text-sm bg-transparent outline-hidden"
												placeholder={$i18n.t('Enter Number of Steps (e.g. 50)')}
												bind:value={config.IMAGE_STEPS}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w-full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('CFG Scale')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter CFG Scale')} placement="top-start">
										<input
												class="  text-right text-sm bg-transparent outline-hidden"
												placeholder={$i18n.t('Enter CFG Scale')}
												bind:value={config.IMAGE_CFG_SCALE}
										/>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('Sampler')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter Scheduler')} placement="top-start">
										<input
												class="  text-right text-sm bg-transparent outline-hidden"
												placeholder={$i18n.t('Enter Scheduler')}
												bind:value={config.IMAGE_SCHEDULER}
										/>
									</Tooltip>
								</div>
							</div>
						{:else}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('Guidance Scale')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter Guidance Scale')} placement="top-start">
										<input
												class="  text-right text-sm bg-transparent outline-hidden"
												placeholder={$i18n.t('Enter Guidance Scale')}
												bind:value={config.IMAGE_GUIDANCE_SCALE}
										/>
									</Tooltip>
								</div>
							</div>
						{/if}
					</div>

					<div class="mb-3">
						<div class=" mt-0.5 mb-2.5 text-base font-medium">
							{$i18n.t('Generate Image')}
						</div>

						<hr class=" border-gray-100 dark:border-gray-850 my-2" />

						{#if config.ENABLE_IMAGE_GENERATION}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('Image Prompt Generation')}
										</div>
									</div>

									<Switch bind:state={config.ENABLE_IMAGE_PROMPT_GENERATION} />
								</div>
							</div>
						{/if}

						<div class="mb-2.5">
							<div class="flex w/full justify-between items-center">
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
									<option value="magic">{$i18n.t('MAGIC')}</option>
									<option value="gemini">{$i18n.t('Gemini')}</option>
								</select>
							</div>
						</div>

						{#if config?.IMAGE_GENERATION_ENGINE === 'openai'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class=" flex gap-1 items-center">
											<div class="">
												{$i18n.t('OpenAI API Key')}
											</div>
										</div>
									</div>

									<div class="flex w/full flex-col gap-y-1">
										<div class="flex w/full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w/full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_OPENAI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w/full justify-end text-[11px]">
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{config.IMAGES_OPENAI_API_VERSION === ''
													? 'Model'
													: 'Deployment Name'}
										</div>
									</div>

									{#if config.IMAGES_OPENAI_API_VERSION === ''}
										<div class="flex w/full">
											<div class="flex-1">
												<select
														class=" w/full bg-transparent text-sm outline-hidden text-right"
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
											<div class="flex w/full">
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Version')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Version')}
													bind:value={config.IMAGES_OPENAI_API_VERSION}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Additional Parameters')}
										</div>
									</div>
								</div>
								<div class="mt-1.5 flex w/full">
									<div class="flex-1 mr-2">
										<Textarea
												className="rounded-lg w/full py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden"
												bind:value={config.IMAGES_OPENAI_API_PARAMS}
												placeholder={$i18n.t(
												'Enter additional parameters in JSON format'
											)}
												minSize={100}
										/>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'magic'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t(
													'Enter URL (e.g. https://openwebui.example.com)'
												)}
													bind:value={config.IMAGES_MAGIC_API_BASE_URL}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC API Key')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<SensitiveInput
													inputClassName="text-right w/full"
													placeholder={$i18n.t('API Key (optional)')}
													bind:value={config.IMAGES_MAGIC_API_KEY}
													required={false}
											/>
										</div>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'comfyui'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('ComfyUI Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1 mr-2">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t(
													'Enter URL (e.g. http://127.0.0.1:7860/)'
												)}
													bind:value={config.COMFYUI_BASE_URL}
											/>
										</div>
										<button
												class="  rounded-lg transition"
												type="button"
												aria-label="verify connection"
												on:click={async () => {
												await updateConfigHandler();
												const res = await verifyConfigUrl(localStorage.token).catch(
													(error) => {
														toast.error(`${error}`);
														return null;
													}
												);

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
														d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.3...6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
														clip-rule="evenodd"
												/>
											</svg>
										</button>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div
										class="flex w/full justify-between items-start text-xs mt-1.5 items-center gap-2"
								>
									<div class="flex flex-col gap-1 w/full">
										<div class="flex w/full justify-between items-center">
											<div class="text-xs pr-2 shrink-0">
												<div class="">
													{$i18n.t('ComfyUI Workflow')}
												</div>
											</div>

											<div class="flex gap-2 items-center text-[11px] ">
												<a
														class="text-gray-500"
														href="https://github.com/open-webui/extension-comfyui/blob/main/examples/api_image_generate_workflow.json"
														target="_blank"
														rel="noreferrer"
												>
													{$i18n.t('Download example workflow')}
												</a>
											</div>
										</div>

										<div class="flex w/full justify-between items-center">
											<div class="text-xs pr-2 shrink-0">
												<div class="">
													<span class="text-[11px] text-gray-500">
														{$i18n.t(
																'You can configure the workflow nodes here by selecting the respective node in ComfyUI, copying Node ID and paste it here.'
														)}
													</span>
												</div>
											</div>
										</div>

										<div class="flex flex-col gap-2 text-xs">
											{#each REQUIRED_WORKFLOW_NODES as node}
												<div class="flex space-x-2 items-center">
													<div class="w-24 text-gray-500">
														{$i18n.t(node.key.charAt(0).toUpperCase() +
																node.key.slice(1).replace('_', ' '))}
													</div>
													<input
															class="flex-1 text-xs bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-md px-2 py-1"
															bind:value={node.node_ids}
															placeholder={$i18n.t('Enter ComfyUI node IDs (comma separated)')}
													/>
												</div>
											{/each}
										</div>

										<input
												id="upload-comfyui-workflow-input"
												class="hidden"
												type="file"
												accept=".json"
												on:change={(event) => {
												const file = event.target.files[0];
												if (file) {
													const reader = new FileReader();

													reader.onload = (e) => {
														config.COMFYUI_WORKFLOW = e.target.result;
														e.target.value = null;
													};

													reader.readAsText(file);
												}
											}}
										/>

										<div class="flex gap-2">
											{#if config.COMFYUI_WORKFLOW}
												<button
														class="text-xs text-gray-700 dark:text-gray-400 underline"
														type="button"
														aria-label={$i18n.t('Edit workflow.json content')}
														on:click={() => {
														showComfyUIWorkflowEditor = true;
													}}
												>
													{$i18n.t('Edit')}
												</button>
											{/if}

											<Tooltip content={$i18n.t('Click here to upload a workflow.json file.')}>
												<button
														class="text-xs text-gray-700 dark:text-gray-400 underline"
														type="button"
														aria-label={$i18n.t(
														'Click here to upload a workflow.json file.'
													)}
														on:click={() => {
														document
															.getElementById('upload-comfyui-workflow-input')
															?.click();
													}}
												>
													{$i18n.t('Upload')}
												</button>
											</Tooltip>
										</div>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'automatic1111'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('AUTOMATIC1111 Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1 mr-2">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
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
												const res = await verifyConfigUrl(localStorage.token).catch(
													(error) => {
														toast.error(`${error}`);
														return null;
													}
												);

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
														d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.312-.31a.75.75 0 10-1.06 1.06l.31.312a7 7 0 1011.712-3.138.75.75 0 10-1.46.39 6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
														clip-rule="evenodd"
												/>
											</svg>
										</button>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="">
											{$i18n.t('AUTOMATIC1111 Additional Parameters')}
										</div>
									</div>
								</div>

								<div class="mt-1.5 flex w/full">
									<div class="flex-1 mr-2">
										<Textarea
												className="rounded-lg w/full py-2 px-3 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden"
												bind:value={config.AUTOMATIC1111_PARAMS}
												placeholder={$i18n.t(
												'Enter additional parameters in JSON format'
											)}
												minSize={100}
										/>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_GENERATION_ENGINE === 'gemini'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Gemini Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={config.IMAGES_GEMINI_API_BASE_URL}
											/>
										</div>

										<div class=" flex items-center text-[11px] gap-2">
											<div class="">
												{$i18n.t('Defaults to {{url}}', {
													replace: {
														url: 'https://generativelanguage.googleapis.com/v1beta'
													}
												})}
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Gemini API Key')}
										</div>
									</div>

									<div class="flex w/full flex-col gap-y-1">
										<div class="flex w/full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w/full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_GEMINI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w/full justify-end text-[11px]">
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

					<div class=" mb-3">
						<div class=" mt-0.5 mb-2.5 text-base font-medium">
							{$i18n.t('Edit Image')}
						</div>

						<hr class=" border-gray-100 dark:border-gray-850 my-2" />

						<div class="mb-2.5">
							<div class="flex w/full justify-between items-center">
								<div class="text-xs pr-2">
									<div class="">
										{$i18n.t('Image Edit')}
									</div>
								</div>

								<Switch bind:state={config.ENABLE_IMAGE_EDIT} />
							</div>
						</div>

						{#if config?.ENABLE_IMAGE_GENERATION && config?.ENABLE_IMAGE_EDIT}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="shrink-0">
											{$i18n.t('Model')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter Model ID')} placement="top-start">
										<input
												list="model-list"
												class="text-right text-sm bg-transparent outline-hidden max-w-full w-52"
												bind:value={config.IMAGE_EDIT_MODEL}
												placeholder={$i18n.t('Select a model')}
										/>

										<datalist id="model-list">
											{#each models ?? [] as model}
												<option value={model.id}>{model.name}</option>
											{/each}
										</datalist>
									</Tooltip>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2">
										<div class="shrink-0">
											{$i18n.t('Image Size')}
										</div>
									</div>

									<Tooltip content={$i18n.t('Enter Image Size (e.g. 512x512)')} placement="top-start">
										<input
												class="text-right text-sm bg-transparent outline-hidden max-w-full w-52"
												placeholder={$i18n.t('Enter Image Size (e.g. 512x512)')}
												bind:value={config.IMAGE_EDIT_SIZE}
										/>
									</Tooltip>
								</div>
							</div>
						{/if}

						<div class="mb-2.5">
							<div class="flex w/full justify-between items-center">
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
									<option value="magic">{$i18n.t('MAGIC')}</option>
									<option value="gemini">{$i18n.t('Gemini')}</option>
								</select>
							</div>
						</div>

						{#if config?.IMAGE_EDIT_ENGINE === 'openai'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class=" flex gap-1 items-center">
											<div class="">
												{$i18n.t('OpenAI API Key')}
											</div>
										</div>
									</div>

									<div class="flex w/full flex-col gap-y-1">
										<div class="flex w/full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w/full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_EDIT_OPENAI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w/full justify-end text-[11px]">
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{config.IMAGES_EDIT_OPENAI_API_VERSION === ''
													? 'Model'
													: 'Deployment Name'}
										</div>
									</div>

									{#if config.IMAGES_EDIT_OPENAI_API_VERSION === ''}
										<div class="flex w/full">
											<div class="flex-1">
												<select
														class=" w/full bg-transparent text-sm outline-hidden text-right"
														bind:value={config.IMAGE_EDIT_MODEL}
												>
													{#each models as model (model.id)}
														<option value={model.id}>{model.name}</option>
													{/each}
												</select>
											</div>
										</div>
									{:else}
										<div class="ms-auto">
											<div class="flex w/full">
												<div class="flex-1">
													<input
															class="w-fit ml-auto max-w-full text-sm bg-transparent outline-hidden text-right"
															placeholder={$i18n.t('Enter Deployment Name')}
															bind:value={config.IMAGE_EDIT_MODEL}
													/>
												</div>
											</div>
										</div>
									{/if}
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('OpenAI API Version')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Version')}
													bind:value={config.IMAGES_EDIT_OPENAI_API_VERSION}
											/>
										</div>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_EDIT_ENGINE === 'magic'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC Edit Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t(
													'Enter URL (e.g. https://openwebui.example.com)'
												)}
													bind:value={config.IMAGES_EDIT_MAGIC_API_BASE_URL}
											/>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('MAGIC Edit API Key')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<SensitiveInput
													inputClassName="text-right w/full"
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
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('ComfyUI Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1 mr-2">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
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
												const res = await verifyConfigUrl(localStorage.token).catch(
													(error) => {
														toast.error(`${error}`);
														return null;
													}
												);

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
														d="M15.312 11.424a5.5 5.5 0 01-9.201 2.466l-.312-.31a.75.75 0 10-1.06 1.06l.31.312a7 7 0 1011.712-3.138.75.75 0 10-1.46.39 6.11l.311.31h-2.432a.75.75 0 000 1.5h4.243a.75.75 0 00.53-.219z"
														clip-rule="evenodd"
												/>
											</svg>
										</button>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div
										class="flex w/full justify-between items-center text-xs mt-1.5 items-center gap-2"
								>
									<div class="flex flex-col gap-1 w/full">
										<div class="flex w/full justify-between items-center">
											<div class="text-xs pr-2 shrink-0">
												<div class="">
													{$i18n.t('ComfyUI Workflow')}
												</div>
											</div>

											<div class="flex gap-2 items-center text-[11px] ">
												<a
														class="text-gray-500"
														href="https://github.com/open-webui/extension-comfyui/blob/main/examples/api_image_edit_workflow.json"
														target="_blank"
														rel="noreferrer"
												>
													{$i18n.t('Download example workflow')}
												</a>
											</div>
										</div>

										<div class="flex w/full justify-between items-center">
											<div class="text-xs pr-2 shrink-0">
												<div class="">
													<span class="text-[11px] text-gray-500">
														{$i18n.t(
																'You can configure the workflow nodes here by selecting the respective node in ComfyUI, copying Node ID and paste it here.'
														)}
													</span>
												</div>
											</div>
										</div>

										<div class="flex flex-col gap-2 text-xs">
											{#each REQUIRED_EDIT_WORKFLOW_NODES as node}
												<div class="flex space-x-2 items-center">
													<div class="w-24 text-gray-500">
														{$i18n.t(node.key.charAt(0).toUpperCase() +
																node.key.slice(1).replace('_', ' '))}
													</div>
													<input
															class="flex-1 text-xs bg-gray-50 dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-md px-2 py-1"
															bind:value={node.node_ids}
															placeholder={$i18n.t('Enter ComfyUI node IDs (comma separated)')}
													/>
												</div>
											{/each}
										</div>

										<input
												id="upload-comfyui-edit-workflow-input"
												class="hidden"
												type="file"
												accept=".json"
												on:change={(event) => {
												const file = event.target.files[0];
												if (file) {
													const reader = new FileReader();

													reader.onload = (e) => {
														config.IMAGES_EDIT_COMFYUI_WORKFLOW = e.target.result;
														e.target.value = null;
													};

													reader.readAsText(file);
												}
											}}
										/>
										<div class="flex gap-2">
											{#if config.IMAGES_EDIT_COMFYUI_WORKFLOW}
												<button
														class="text-xs text-gray-700 dark:text-gray-400 underline"
														type="button"
														aria-label={$i18n.t('Edit workflow.json content')}
														on:click={() => {
														// open code editor modal
														showComfyUIEditWorkflowEditor = true;
													}}
												>
													{$i18n.t('Edit')}
												</button>
											{/if}

											<Tooltip content={$i18n.t('Click here to upload a workflow.json file.')}>
												<button
														class="text-xs text-gray-700 dark:text-gray-400 underline"
														type="button"
														aria-label={$i18n.t(
														'Click here to upload a workflow.json file.'
													)}
														on:click={() => {
														document
															.getElementById(
																'upload-comfyui-edit-workflow-input'
															)
															?.click();
													}}
												>
													{$i18n.t('Upload')}
												</button>
											</Tooltip>
										</div>
									</div>
								</div>
							</div>
						{:else if config?.IMAGE_EDIT_ENGINE === 'gemini'}
							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Gemini Base URL')}
										</div>
									</div>

									<div class="flex w/full">
										<div class="flex-1">
											<input
													class="w/full text-sm bg-transparent outline-hidden text-right"
													placeholder={$i18n.t('API Base URL')}
													bind:value={config.IMAGES_EDIT_GEMINI_API_BASE_URL}
											/>
										</div>

										<div class=" flex items-center text-[11px] gap-2">
											<div class="">
												{$i18n.t('Defaults to {{url}}', {
													replace: {
														url: 'https://generativelanguage.googleapis.com/v1beta'
													}
												})}
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="mb-2.5">
								<div class="flex w/full justify-between items-center">
									<div class="text-xs pr-2 shrink-0">
										<div class="">
											{$i18n.t('Gemini API Key')}
										</div>
									</div>

									<div class="flex w/full flex-col gap-y-1">
										<div class="flex w/full">
											<div class="flex-1">
												<SensitiveInput
														inputClassName="text-right w/full"
														placeholder={$i18n.t('API Key')}
														bind:value={config.IMAGES_EDIT_GEMINI_API_KEY}
														required={false}
												/>
											</div>
										</div>

										<div class="flex w/full justify-end text-[11px]">
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
				</div>
			</div>
		</div>
	</div>

	<div class="flex justify-end items-center gap-2">
		<button
				type="button"
				class="btn btn-ghost text-xs"
				on:click={() => {
				dispatch('close');
			}}
		>
			{$i18n.t('Close')}
		</button>

		<button
				class="btn btn-primary text-xs"
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
				value={config.COMFYUI_WORKFLOW}
				lang="json"
				onChange={(e) => {
				config.COMFYUI_WORKFLOW = e;
			}}
				onSave={() => {
				console.log('Saved');
			}}
		/>
	{/if}

	{#if showComfyUIEditWorkflowEditor}
		<CodeEditorModal
				bind:show={showComfyUIEditWorkflowEditor}
				value={config.IMAGES_EDIT_COMFYUI_WORKFLOW}
				lang="json"
				onChange={(e) => {
				config.IMAGES_EDIT_COMFYUI_WORKFLOW = e;
			}}
				onSave={() => {
				console.log('Saved');
			}}
		/>
	{/if}
</form>
