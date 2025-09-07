<script lang="ts" setup>
import type { IAiDataContent } from '@/Interface';
import { useNodeTypesStore } from '@/stores/nodeTypes.store';
import { useWorkflowsStore } from '@/stores/workflows.store';
import type { INodeTypeDescription, NodeConnectionType, NodeError } from 'n8n-workflow';
import { computed } from 'vue';
import NodeIcon from '@/components/NodeIcon.vue';
import AiRunContentBlock from './AiRunContentBlock.vue';
import { useI18n } from '@n8n/i18n';
import ConsumedTokensDetails from '@/components/ConsumedTokensDetails.vue';
import ViewSubExecution from '../ViewSubExecution.vue';
import { formatTokenUsageCount } from '@/utils/aiUtils';
import { getConsumedTokens } from '@/features/logs/logs.utils';

interface RunMeta {
	startTimeMs: number;
	executionTimeMs: number;
	node: INodeTypeDescription | null;
	type: 'input' | 'output';
	connectionType: NodeConnectionType;
	subExecution?: {
		workflowId: string;
		executionId: string;
	};
}

const props = defineProps<{
	node: string;
	runIndex: number;
	inputData: IAiDataContent;
	contentIndex: number;
}>();

const nodeTypesStore = useNodeTypesStore();
const workflowsStore = useWorkflowsStore();

const i18n = useI18n();

const consumedTokensSum = computed(() => getConsumedTokens(props.inputData.data ?? []));

function extractRunMeta(run: IAiDataContent) {
	const uiNode = workflowsStore.getNodeByName(props.node);
	const nodeType = nodeTypesStore.getNodeType(uiNode?.type ?? '');

	const runMeta: RunMeta = {
		startTimeMs: run.metadata.startTime,
		executionTimeMs: run.metadata.executionTime,
		node: nodeType,
		type: run.inOut,
		connectionType: run.type,
		subExecution: run.metadata?.subExecution,
	};

	return runMeta;
}

const runMeta = computed(() => {
	if (props.inputData.inOut === 'input') {
		return;
	}
	return extractRunMeta(props.inputData);
});

const executionRunData = computed(() => {
	return workflowsStore.getWorkflowExecution?.data?.resultData?.runData;
});

const outputError = computed(() => {
	return executionRunData.value?.[props.node]?.[props.runIndex]?.error as NodeError | undefined;
});
</script>

<template>
	<div :class="$style.container">
		<header :class="$style.header">
			<NodeIcon
				v-if="runMeta?.node"
				:class="$style.nodeIcon"
				:node-type="runMeta.node"
				:size="20"
			/>
			<div :class="$style.headerWrap">
				<p :class="$style.title">
					{{ props.node }}
				</p>
				<ul :class="$style.meta">
					<li v-if="runMeta?.startTimeMs">{{ runMeta?.executionTimeMs }}ms</li>
					<li v-if="runMeta?.startTimeMs">
						<n8n-tooltip>
							<template #content>
								{{ new Date(runMeta?.startTimeMs).toLocaleString() }}
							</template>
							{{
								i18n.baseText('runData.aiContentBlock.startedAt', {
									interpolate: {
										startTime: new Date(runMeta?.startTimeMs).toLocaleTimeString(),
									},
								})
							}}
						</n8n-tooltip>
					</li>
					<li v-if="runMeta">
						<ViewSubExecution :task-metadata="runMeta" :display-mode="'ai'" :inline="true" />
					</li>
					<li v-if="(consumedTokensSum?.totalTokens ?? 0) > 0" :class="$style.tokensUsage">
						{{
							i18n.baseText('runData.aiContentBlock.tokens', {
								interpolate: {
									count: formatTokenUsageCount(consumedTokensSum, 'total'),
								},
							})
						}}
						<n8n-info-tip type="tooltip" theme="info-light" tooltip-placement="right">
							<ConsumedTokensDetails :consumed-tokens="consumedTokensSum" />
						</n8n-info-tip>
					</li>
				</ul>
			</div>
		</header>

		<main :class="$style.content">
			<AiRunContentBlock
				:run-data="props.inputData"
				:error="props.inputData.inOut === 'input' ? undefined : outputError"
			/>
		</main>
	</div>
</template>

<style type="scss" module>
.container {
	padding: 0 var(--spacing-s) var(--spacing-s);
}
.nodeIcon {
	margin-top: calc(var(--spacing-3xs) * -1);
}
.header {
	display: flex;
	align-items: center;
	gap: var(--spacing-3xs);
	margin-bottom: var(--spacing-s);
}
.headerWrap {
	display: flex;
	flex-direction: column;
}
.title {
	display: flex;
	align-items: center;
	font-size: var(--font-size-s);
	gap: var(--spacing-3xs);
	color: var(--color-text-dark);
}
.meta {
	list-style: none;
	display: flex;
	align-items: center;
	flex-wrap: wrap;
	font-size: var(--font-size-xs);

	& > li:not(:last-child) {
		border-right: 1px solid var(--color-text-base);
		padding-right: var(--spacing-3xs);
	}

	& > li:not(:first-child) {
		padding-left: var(--spacing-3xs);
	}
}
.tokensUsage {
	display: flex;
	align-items: center;
	gap: var(--spacing-3xs);
}
</style>
