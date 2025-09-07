<script setup lang="ts">
import { useRunWorkflow } from '@/composables/useRunWorkflow';
import LogsOverviewRow from '@/features/logs/components/LogsOverviewRow.vue';
import type { LatestNodeInfo, LogEntry } from '@/features/logs/logs.types';
import { computed } from 'vue';
import { useRouter } from 'vue-router';

const {
	isReadOnly,
	selected,
	isCompact,
	latestNodeInfo,
	flatLogEntries,
	shouldShowTokenCountColumn,
} = defineProps<{
	selected?: LogEntry;
	isReadOnly: boolean;
	isCompact: boolean;
	shouldShowTokenCountColumn: boolean;
	canOpenNdv: boolean;
	flatLogEntries: LogEntry[];
	latestNodeInfo: Record<string, LatestNodeInfo>;
}>();

const emit = defineEmits<{
	select: [LogEntry | undefined];
	openNdv: [LogEntry];
	toggleExpanded: [LogEntry];
}>();

const router = useRouter();
const runWorkflow = useRunWorkflow({ router });

const isExpanded = computed(() =>
	flatLogEntries.reduce<Record<string, boolean>>((acc, entry, index, arr) => {
		acc[entry.id] = arr[index + 1]?.parent?.id === entry.id;
		return acc;
	}, {}),
);
async function handleTriggerPartialExecution(treeNode: LogEntry) {
	const latestName = latestNodeInfo[treeNode.node.id]?.name ?? treeNode.node.name;

	if (latestName) {
		await runWorkflow.runWorkflow({ destinationNode: latestName });
	}
}
</script>

<template>
	<div role="tree">
		<LogsOverviewRow
			v-for="data of flatLogEntries"
			:key="data.id"
			:data="data"
			:is-read-only="isReadOnly"
			:is-selected="data.id === selected?.id"
			:is-compact="isCompact"
			:should-show-token-count-column="shouldShowTokenCountColumn"
			:latest-info="latestNodeInfo[data.node.id]"
			:expanded="isExpanded[data.id]"
			:can-open-ndv="canOpenNdv"
			@toggle-expanded="emit('toggleExpanded', data)"
			@open-ndv="emit('openNdv', data)"
			@trigger-partial-execution="handleTriggerPartialExecution(data)"
			@toggle-selected="emit('select', selected?.id === data.id ? undefined : data)"
		/>
	</div>
</template>
