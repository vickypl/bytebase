<template>
  <div class="textlabel">
    <div class="flex items-center justify-between">
      <div class="radio-set-row">
        <template v-if="project.name !== DEFAULT_PROJECT_V1_NAME">
          <label class="radio">
            <input
              v-model="state.transferSource"
              tabindex="-1"
              type="radio"
              class="btn"
              value="DEFAULT"
            />
            <span class="label">
              {{ $t("quick-action.from-unassigned-databases") }}
            </span>
          </label>
          <label class="radio">
            <input
              v-model="state.transferSource"
              tabindex="-1"
              type="radio"
              class="btn"
              value="OTHER"
            />
            <span class="label">
              {{ $t("quick-action.from-projects") }}
            </span>
          </label>
        </template>
      </div>
      <NInputGroup style="width: auto">
        <InstanceSelect
          v-if="state.transferSource == 'DEFAULT'"
          class="!w-48"
          :instance="instanceFilter?.uid ?? String(UNKNOWN_ID)"
          :include-all="true"
          :filter="filterInstance"
          @update:instance="changeInstanceFilter"
        />
        <ProjectSelect
          v-else-if="state.transferSource == 'OTHER'"
          :include-all="true"
          :project="projectFilter?.uid ?? String(UNKNOWN_ID)"
          :allowed-project-role-list="[PresetRoleType.OWNER]"
          :filter="filterSourceProject"
          @update:project="changeProjectFilter"
        />
        <SearchBox
          :value="searchText"
          :placeholder="$t('database.filter-database')"
          @update:value="$emit('search-text-change', $event)"
        />
      </NInputGroup>
    </div>
    <div v-if="state.transferSource == 'DEFAULT'" class="textinfolabel mt-2">
      {{ $t("quick-action.unassigned-db-hint") }}
    </div>
  </div>
</template>

<script lang="ts" setup>
import { NInputGroup } from "naive-ui";
import { type PropType, computed, reactive, watch } from "vue";
import { InstanceSelect, ProjectSelect, SearchBox } from "@/components/v2";
import { useInstanceV1Store, useProjectV1Store } from "@/store";
import {
  UNKNOWN_ID,
  ComposedDatabase,
  ComposedInstance,
  DEFAULT_PROJECT_V1_NAME,
  PresetRoleType,
} from "@/types";
import { Project } from "@/types/proto/v1/project_service";
import { TransferSource } from "./utils";

interface LocalState {
  transferSource: TransferSource;
}

const props = defineProps({
  project: {
    required: true,
    type: Object as PropType<Project>,
  },
  rawDatabaseList: {
    type: Array as PropType<ComposedDatabase[]>,
    default: () => [],
  },
  transferSource: {
    type: String as PropType<TransferSource>,
    required: true,
  },
  instanceFilter: {
    type: Object as PropType<ComposedInstance>,
    default: undefined,
  },
  projectFilter: {
    type: Object as PropType<Project>,
    default: undefined,
  },
  searchText: {
    type: String,
    default: "",
  },
});

const emit = defineEmits<{
  (event: "change", src: TransferSource): void;
  (event: "select-instance", instance: ComposedInstance | undefined): void;
  (event: "select-project", project: Project | undefined): void;
  (event: "search-text-change", searchText: string): void;
}>();

const state = reactive<LocalState>({
  transferSource: props.transferSource,
});

const filterSourceProject = (project: Project) => {
  return project.uid !== props.project.uid;
};

const nonEmptyInstanceUidSet = computed(() => {
  const instanceList = props.rawDatabaseList.map((db) => db.instanceEntity);
  return new Set(instanceList.map((instance) => instance.uid));
});

const changeInstanceFilter = (uid: string | undefined) => {
  if (!uid || uid === String(UNKNOWN_ID)) {
    return emit("select-instance", undefined);
  }
  emit("select-instance", useInstanceV1Store().getInstanceByUID(uid));
};

const filterInstance = (instance: ComposedInstance) => {
  if (instance.uid === String(UNKNOWN_ID)) return true; // "ALL" can be displayed.
  return nonEmptyInstanceUidSet.value.has(instance.uid);
};

const changeProjectFilter = (uid: string | undefined) => {
  if (!uid || uid === String(UNKNOWN_ID)) {
    return emit("select-project", undefined);
  }
  emit("select-project", useProjectV1Store().getProjectByUID(uid));
};

watch(
  () => props.transferSource,
  (src) => (state.transferSource = src)
);

watch(
  () => state.transferSource,
  (src) => {
    if (src !== props.transferSource) {
      emit("change", src);
    }
  }
);
</script>
