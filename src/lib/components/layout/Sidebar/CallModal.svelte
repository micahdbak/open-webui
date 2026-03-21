<script lang="ts">
	import { getContext } from 'svelte';
	const i18n = getContext('i18n');

	import { toast } from 'svelte-sonner';
	import { page } from '$app/stores';
	import { goto } from '$app/navigation';

	//import { user } from '$lib/stores';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import Modal from '$lib/components/common/Modal.svelte';
	import DeleteConfirmDialog from '$lib/components/common/ConfirmDialog.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';
	import MemberSelector from '$lib/components/workspace/common/MemberSelector.svelte';
	import Visibility from '$lib/components/workspace/common/Visibility.svelte';

	export let show = false;
	export let onSubmit: Function = () => {};
	export let onUpdate: Function = () => {};

	export let call: any = null;
	export let edit = false;

	let isPrivate = null;
    let userIds = [];

	let loading = false;

	const submitHandler = async () => {
		loading = true;

		await onSubmit({
			is_private: isPrivate ?? true,
			user_ids: userIds
		});
        
		show = false;
		loading = false;
	};

	const init = () => {
		if (call) {
            isPrivate = call?.is_private ?? true;
            userIds = call?.user_ids ?? [];
		}
	};

	$: if (show) {
		init();
	} else {
		resetHandler();
	}

	let showDeleteConfirmDialog = false;

	const deleteHandler = async () => {
		showDeleteConfirmDialog = false;
		if (!call?.id) {
			show = false;
			return;
		}

		const callId = call.id;

        const res = {};
		//const res = await deleteCallById(localStorage.token, callId).catch((error) => {
		//	toast.error(error.message);
		//});

		if (res) {
			toast.success($i18n.t('Call deleted successfully'));
			onUpdate();

			if ($page.url.pathname === `/calls/${callId}`) {
				goto('/');
			}
		}

		show = false;
	};

	const resetHandler = () => {
		call = null;
        isPrivate = null;
        userIds = [];
		loading = false;
	};
</script>

<Modal size="md" bind:show>
	<div>
		<div class=" flex justify-between dark:text-gray-300 px-5 pt-4 pb-1">
			<div class=" text-lg font-medium self-center">
				{#if edit}
					{$i18n.t('Edit Call')}
				{:else}
					{$i18n.t('Start Call')}
				{/if}
			</div>
			<button
				class="self-center"
				on:click={() => {
					show = false;
				}}
			>
				<XMark className={'size-5'} />
			</button>
		</div>

		<div class="flex flex-col md:flex-row w-full px-5 pb-4 md:space-x-4 dark:text-gray-200">
			<div class=" flex flex-col w-full sm:flex-row sm:justify-center sm:space-x-6">
				<form
					class="flex flex-col w-full"
					on:submit|preventDefault={() => {
						submitHandler();
					}}
				>
                    <div class="-mx-2 mb-1 mt-2.5 px-2">
                        <Visibility
                            state={isPrivate ? 'private' : 'public'}
                            onChange={(value: string) => {
                                if (value === 'private') {
                                    isPrivate = true;
                                } else {
                                    isPrivate = false;
                                }
                            }}
                        />
                    </div>

                    <div class="">
                        <MemberSelector bind:userIds includeGroups={false} />
                    </div>

					<div class="flex justify-end pt-3 text-sm font-medium gap-1.5">
						{#if edit}
							<button
								class="px-3.5 py-1.5 text-sm font-medium dark:bg-black dark:hover:bg-black/90 dark:text-white bg-white text-black hover:bg-gray-100 transition rounded-full flex flex-row space-x-1 items-center"
								type="button"
								on:click={() => {
									showDeleteConfirmDialog = true;
								}}
							>
								{$i18n.t('Delete')}
							</button>
						{/if}

						<button
							class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-950 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full flex flex-row space-x-1 items-center {loading
								? ' cursor-not-allowed'
								: ''}"
							type="submit"
							disabled={loading}
						>
							{#if edit}
								{$i18n.t('Update')}
							{:else}
								{$i18n.t('Create')}
							{/if}

							{#if loading}
								<div class="ml-2 self-center">
									<Spinner />
								</div>
							{/if}
						</button>
					</div>
				</form>
			</div>
		</div>
	</div>
</Modal>

<DeleteConfirmDialog
	bind:show={showDeleteConfirmDialog}
	message={$i18n.t('Are you sure you want to delete this call?')}
	confirmLabel={$i18n.t('Delete')}
	on:confirm={() => {
		deleteHandler();
	}}
/>
