<template>
  <div class="space-y-2">
    <InstanceOperations :instance-list="selectedInstanceList" />
    <BBGrid
      class="border-y"
      :column-list="columnList"
      :data-source="instanceList"
      :custom-header="true"
      :show-placeholder="true"
      @click-row="clickInstance"
    >
      <template #header>
        <div role="table-row" class="bb-grid-row bb-grid-header-row group">
          <div
            v-for="(column, index) in columnList"
            :key="index"
            role="table-cell"
            class="bb-grid-header-cell capitalize"
            :class="[column.class]"
          >
            <template
              v-if="index === 0 && allowSelection && instanceList.length > 0"
            >
              <input
                type="checkbox"
                class="h-4 w-4 text-accent rounded disabled:cursor-not-allowed border-control-border focus:ring-accent"
                :checked="allSelectionState.checked"
                :indeterminate="allSelectionState.indeterminate"
                @input="
                  selectAllInstances(
                    ($event.target as HTMLInputElement).checked
                  )
                "
              />
            </template>
            <template v-else>{{ column.title }}</template>
          </div>
        </div>
      </template>
      <template #item="{ item: instance }: InstanceRow">
        <div v-if="allowSelection" class="bb-grid-cell">
          <input
            type="checkbox"
            class="h-4 w-4 text-accent rounded disabled:cursor-not-allowed border-control-border focus:ring-accent"
            :checked="isInstanceSelected(instance)"
            @click.stop="
              toggleSelectInstance(instance, !isInstanceSelected(instance))
            "
          />
        </div>
        <div class="bb-grid-cell">
          <InstanceV1Name :instance="instance" :link="false" tag="div" />
        </div>
        <div class="bb-grid-cell">
          <EnvironmentV1Name
            :environment="instance.environmentEntity"
            :link="false"
          />
        </div>
        <div class="bb-grid-cell">
          {{ hostPortOfInstanceV1(instance) }}
        </div>
        <div class="bb-grid-cell hidden sm:flex">
          <button
            v-if="instance.externalLink?.trim().length != 0"
            class="btn-icon"
            @click.stop="window.open(urlfy(instance.externalLink), '_blank')"
          >
            <heroicons-outline:external-link class="w-4 h-4" />
          </button>
        </div>
        <div v-if="canAssignLicense" class="bb-grid-cell hover:underline">
          {{ instance.activation ? "Y" : "" }}
        </div>
      </template>
    </BBGrid>
  </div>
</template>

<script lang="ts" setup>
import { computed, reactive } from "vue";
import { useI18n } from "vue-i18n";
import { useRouter } from "vue-router";
import { BBGrid, BBGridColumn, BBGridRow } from "@/bbkit";
import { ComposedInstance } from "@/types";
import { urlfy, instanceV1Slug, hostPortOfInstanceV1 } from "@/utils";
import EnvironmentV1Name from "../../EnvironmentV1Name.vue";

export type InstanceRow = BBGridRow<ComposedInstance>;

interface LocalState {
  selectedInstance: Set<string>;
  processing: boolean;
}

const props = defineProps<{
  instanceList: ComposedInstance[];
  canAssignLicense: boolean;
  allowSelection: boolean;
}>();

const { t } = useI18n();
const router = useRouter();
const state = reactive<LocalState>({
  selectedInstance: new Set(),
  processing: false,
});

const columnList = computed((): BBGridColumn[] => {
  const list = [
    // checkbox
    {
      title: "",
      width: "minmax(auto, 3rem)",
      hide: !props.allowSelection,
    },
    {
      title: t("common.name"),
      width: "minmax(min-content, auto)",
    },
    {
      title: t("common.environment"),
      width: "minmax(min-content, auto)",
    },
    {
      title: t("common.Address"),
      width: "minmax(min-content, auto)",
    },
    {
      title: t("instance.external-link"),
      width: { sm: "minmax(min-content, auto)" },
      class: "hidden sm:flex",
    },
    {
      title: t("subscription.instance-assignment.license"),
      width: "minmax(min-content, auto)",
      hide: !props.canAssignLicense,
    },
  ];
  return list.filter((col) => !col.hide);
});

const clickInstance = (
  instance: ComposedInstance,
  section: number,
  row: number,
  e: MouseEvent
) => {
  const url = `/instance/${instanceV1Slug(instance)}`;
  if (e.ctrlKey || e.metaKey) {
    window.open(url, "_blank");
  } else {
    router.push(url);
  }
};

const isInstanceSelected = (instance: ComposedInstance): boolean => {
  return state.selectedInstance.has(instance.name);
};

const allSelectionState = computed(() => {
  const checked =
    state.selectedInstance.size > 0 &&
    props.instanceList.every((instance) =>
      state.selectedInstance.has(instance.name)
    );
  const indeterminate =
    !checked &&
    props.instanceList.some((instance) =>
      state.selectedInstance.has(instance.name)
    );

  return {
    checked,
    indeterminate,
  };
});

const toggleSelectInstance = (
  instance: ComposedInstance,
  selected: boolean
) => {
  if (selected) {
    state.selectedInstance.add(instance.name);
  } else {
    state.selectedInstance.delete(instance.name);
  }
};

const selectAllInstances = (selected: boolean): void => {
  for (const instance of props.instanceList) {
    toggleSelectInstance(instance, selected);
  }
};

const selectedInstanceList = computed(() => {
  return props.instanceList.filter((instance) =>
    state.selectedInstance.has(instance.name)
  );
});
</script>
