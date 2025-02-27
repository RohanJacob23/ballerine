<script lang="ts">
  import type { WorkflowOptionsBrowser } from '@ballerine/workflow-browser-sdk';
  import {
    createMutation,
    createQuery,
    type CreateQueryOptions,
    useQueryClient,
  } from '@tanstack/svelte-query';
  import { fetchJson, makeWorkflow } from '@/utils';
  import SignUp from './SignUp.svelte';
  import Workflow from './Workflow.svelte';
  import { NO_AUTH_USER_KEY } from '@/constants';
  import { writable } from 'svelte/store';
  import Card from '@/components/Card.svelte';
  import Approved from '@/components/Approved.svelte';
  import Rejected from '@/components/Rejected.svelte';
  import Resubmission from '@/components/Resubmission.svelte';
  import ThankYou from '@/components/ThankYou.svelte';
  import Intent from '@/components/Intent.svelte';
  import { BallerineBackOfficeService } from '@/services/ballerine-backoffice.service';
  import DevSidebar from '@/visualiser/dev-sidebar.svelte';

  let noAuthUserId = sessionStorage.getItem(NO_AUTH_USER_KEY);

  const { fetchEndUser, fetchIntent, fetchSignUp, fetchWorkflow, fetchWorkflows } =
    new BallerineBackOfficeService();

  const createEndUserQuery = (id: string) =>
    createQuery({
      queryKey: ['end-user', { id }],
      queryFn: async () => fetchEndUser(id),
      onSuccess(data) {
        const cached = sessionStorage.getItem(NO_AUTH_USER_KEY);

        if ((cached && cached === data?.id) || !data?.id) return;

        noAuthUserId = data?.id;
        sessionStorage.setItem(NO_AUTH_USER_KEY, noAuthUserId);
      },
      onError(error) {
        if (error.message !== 'Not Found (404)') {
          throw error;
        }

        sessionStorage.removeItem(NO_AUTH_USER_KEY);
        noAuthUserId = undefined;
      },
      enabled: typeof id === 'string' && id.length > 0,
    });
  const createWorkflowsQuery = (
    options: CreateQueryOptions<Awaited<ReturnType<typeof fetchWorkflows>>> = {},
  ) =>
    createQuery({
      queryKey: ['workflows'],
      queryFn: fetchWorkflows,
      enabled: typeof noAuthUserId === 'string' && noAuthUserId.length > 0,
      ...options,
    });
  const createIntentQuery = () =>
    createQuery({
      queryKey: ['intent'],
      queryFn: async () => {
        const data = await fetchIntent();

        if (!data?.[0]) return;

        return data?.[0];
      },
      enabled: false,
    });
  const createFirstWorkflowQuery = () =>
    createWorkflowsQuery({
      select: workflows => {
        return Array.isArray(workflows)
          ? workflows?.find(
              workflow => workflow?.workflowDefinition?.name === 'onboarding_client_collect_data',
            )
          : undefined;
      },
    });
  const createWorkflowQuery = (id: string) =>
    createQuery({
      queryKey: ['workflows', { id }],
      queryFn: async () => {
        const data = await fetchWorkflow(id);

        if (!data) return;

        return data;
      },
      refetchInterval(data) {
        if (
          endUserState === 'REJECTED' ||
          endUserState === 'APPROVED' ||
          (endUserState === 'NEW' && data?.workflowRuntimeData?.status === 'created') ||
          (isProcessing && data?.workflowRuntimeData?.status !== 'completed')
        ) {
          return false;
        }

        return parseInt(import.meta.env.VITE_POLLING_INTERVAL) * 1000 || false;
      },
      enabled: typeof id === 'string' && id.length > 0,
    });
  const queryClient = useQueryClient();
  const createSignUpMutation = () =>
    createMutation({
      mutationFn: fetchSignUp,
      onSuccess: data => {
        sessionStorage.setItem(NO_AUTH_USER_KEY, data?.id);
        noAuthUserId = data?.id;
        queryClient.invalidateQueries();
      },
    });
  $: endUserQuery = createEndUserQuery(noAuthUserId);
  const firstWorkflowQuery = createFirstWorkflowQuery();
  $: workflowQuery = createWorkflowQuery($firstWorkflowQuery?.data?.workflowRuntimeData?.id);
  const intentQuery = createIntentQuery();

  const signUpMutation = createSignUpMutation();
  const onSubmit = async ({
    fname: firstName,
    lname: lastName,
  }: {
    fname: string;
    lname: string;
  }) =>
    $signUpMutation.mutate({
      firstName,
      lastName,
    });

  const workflow = writable<WorkflowOptionsBrowser | undefined>();
  const debugWf = writable<{ definition: unknown }>();

  workflow.subscribe(w => {
    debugWf.set(w as any);
  });

  const workflowComponentStateUpdated = ({ detail }: { detail: any }) => {
    const debugState = {
      ...$workflow,
      definition: {
        ...$workflow?.definition,
        context: detail.context,
        initial: detail.value,
      },
    };
    debugWf.set(debugState);
  };

  const mergeWorkflow = () => makeWorkflow($workflowQuery?.data || $intentQuery?.data);
  const handleResubmit = () => {
    workflow.set(mergeWorkflow());
  };

  let nextWorkflow;
  let shouldResubmit = false;
  $: isCompleted = $workflowQuery.data?.workflowRuntimeData?.status === 'completed';
  $: endUserId = $endUserQuery.data?.id;
  $: endUserState = $endUserQuery.data?.state;
  $: isProcessing = endUserState === 'PROCESSING';
  $: isValidWorkflow = endUserId && !isCompleted;

  $: {
    if (
      endUserId &&
      ($workflowQuery?.data?.workflowDefinition || $intentQuery?.data?.workflowDefinition)
    ) {
      nextWorkflow = mergeWorkflow();

      if (
        nextWorkflow?.definition?.initial !== $workflow?.definition?.initial &&
        nextWorkflow?.definition?.context?.id?.resubmissionReason
      ) {
        shouldResubmit = true;
      } else {
        workflow.set(nextWorkflow);
        shouldResubmit = false;
      }
    } else {
      workflow.set(undefined);
      shouldResubmit = false;
    }
  }

  let message;

  $: {
    switch (endUserState) {
      case 'PROCESSING':
        message = '';
        break;
      case 'REJECTED':
        message = 'Your request was declined.';
        break;
      case 'APPROVED':
        message = 'Your request was approved :)';
        break;
      default:
        message = '';
    }
  }
</script>

<div class="h-full flex flex-row items-center justify-center">
  <main class="h-full w-full flex flex-col items-center justify-center p-6">
    {#if !endUserId}
      <SignUp {onSubmit} />
    {/if}

    {#if $workflow && !isCompleted && !shouldResubmit}
      <Workflow workflow={$workflow} on:workflow-updated={workflowComponentStateUpdated} />
    {/if}

    {#if endUserId && !$workflow && !isProcessing}
      <Intent disabled={!endUserId} refetch={$intentQuery.refetch} />
    {/if}

    {#if endUserId && isProcessing && isCompleted}
      <ThankYou />
    {/if}

    {#if isValidWorkflow && shouldResubmit}
      <Resubmission
        {handleResubmit}
        reason={nextWorkflow?.definition?.context?.id?.resubmissionReason
          ?.toLowerCase()
          ?.replace(/_/g, ' ')}
      />
    {/if}

    {#if endUserState === 'REJECTED'}
      <Rejected />
    {/if}

    {#if endUserState === 'APPROVED'}
      <Approved />
    {/if}
  </main>
  {#if $debugWf}
    <DevSidebar workflowDefinition={$debugWf?.definition} />
  {/if}
</div>
