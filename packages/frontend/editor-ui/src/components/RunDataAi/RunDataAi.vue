<script lang="ts" setup>
import { computed, shallowRef } from 'vue';
import type { IAiDataContent, INodeUi } from '@/Interface';
import { useI18n } from '@n8n/i18n';
import LogsOverviewRows from '@/features/logs/components/LogsOverviewRows.vue';
import { useLogsExecutionData } from '@/features/logs/composables/useLogsExecutionData';
import { useLogsTreeExpand } from '@/features/logs/composables/useLogsTreeExpand';
import type { LogTreeFilter, LogEntry, LogEntrySelection } from '@/features/logs/logs.types';
import { findSelectedLogEntry } from '@/features/logs/logs.utils';
import { N8nText } from '@n8n/design-system';
import RunDataAiContent from '@/components/RunDataAi/RunDataAiContent.vue';
import { getReferencedData } from '@/components/RunDataAi/utils';

const { node, runIndex = 0 } = defineProps<{
	node: INodeUi;
	runIndex?: number;
}>();

const i18n = useI18n();

const filter = computed<LogTreeFilter>(() => ({
	rootNodeName: node.name,
	rootNodeRunIndex: runIndex,
}));
const { entries, execution, latestNodeNameById, loadSubExecution } = useLogsExecutionData(filter);
const { flatLogEntries, toggleExpanded } = useLogsTreeExpand(entries, loadSubExecution);
const isExecutionStopped = computed(() => execution.value?.stoppedAt !== undefined);
const manualLogEntrySelection = shallowRef<LogEntrySelection>({ type: 'initial' });
const selected = computed(() =>
	findSelectedLogEntry(manualLogEntrySelection.value, entries.value, !isExecutionStopped.value),
);
const selectedContent = computed<IAiDataContent[]>(() =>
	selected.value?.runData ? getReferencedData(selected.value.runData) : [],
);
const emptyText = computed(() =>
	i18n.baseText('ndv.output.ai.empty', { interpolate: { node: node.name } }),
);

function select(entry: LogEntry | undefined) {
	manualLogEntrySelection.value = entry ? { type: 'selected', entry } : { type: 'none' };
}
</script>

<template>
	<div :class="$style.container">
		<template v-if="flatLogEntries.length > 0">
			<LogsOverviewRows
				:class="$style.tree"
				is-compact
				is-read-only
				:flat-log-entries="flatLogEntries"
				:should-show-token-count-column="false"
				:latest-node-info="latestNodeNameById"
				:selected="selected"
				:can-open-ndv="false"
				@toggle-expanded="toggleExpanded"
				@select="select"
			/>
			<div :class="$style.runData">
				<div v-if="selected">
					<RunDataAiContent
						v-for="(data, index) in selectedContent"
						:key="index"
						data-test-id="lm-chat-logs-entry"
						:input-data="data"
						:content-index="index"
						:node="selected.node.name"
						:run-index="selected.runIndex"
					/>
				</div>
				<div v-else :class="$style.empty">
					<N8nText size="large">
						{{ emptyText }}
					</N8nText>
				</div>
			</div>
		</template>
		<div v-else :class="$style.noData">
			{{ i18n.baseText('ndv.output.ai.waiting') }}
		</div>
	</div>
</template>

<style lang="scss" module>
.noData {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 100%;
	color: var(--color-text-light);
}

.empty {
	padding: var(--spacing-l);
}

.tree {
	flex-shrink: 0;
	flex-grow: 0;
	width: 30%;
	min-width: 180px;
}

.runData {
	width: 100%;
	height: 100%;
	overflow: auto;
}

.container {
	height: 100%;
	padding: 0 var(--spacing-xs);
	display: flex;
	gap: var(--spacing-xs);
}
</style>
