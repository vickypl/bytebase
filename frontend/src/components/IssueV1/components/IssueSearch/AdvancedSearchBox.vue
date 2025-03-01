<template>
  <div ref="containerRef" class="bb-advanced-issue-search-box w-full relative">
    <NInput
      ref="inputRef"
      v-model:value="inputText"
      :placeholder="placeholder ?? $t('issue.advanced-search.self')"
      class="bb-advanced-issue-search-box__input"
      style="--n-padding-left: 8px; --n-padding-right: 4px"
      @click="handleInputClick"
      @blur="hideMenu"
      @keyup="handleKeyUp"
      @keydown="handleKeyDown"
    >
      <template #prefix>
        <div
          class="flex flex-row items-center justify-start gap-x-2"
          :style="{
            'max-width': `calc(${containerWidth}px - 14rem)`,
          }"
        >
          <div class="flex items-center space-x-2">
            <FilterIcon class="w-4 h-4 text-control-placeholder" />
            <span class="textinfolabel">
              {{ $t("issue.advanced-search.filter") }}
            </span>
          </div>
          <div
            ref="tagsContainerRef"
            class="flex-1 flex flex-row items-center flex-nowrap gap-1 overflow-auto hide-scrollbar"
          >
            <ScopeTags
              :params="params"
              :focused-tag-id="focusedTagId"
              :readonly-scopes="readonlyScopes"
              @select-scope="selectScopeFromTag"
              @remove-scope="removeScope"
            />
          </div>
        </div>
      </template>
      <template #suffix>
        <NButton
          v-show="clearable"
          quaternary
          circle
          size="tiny"
          @click.stop.prevent="handleClear"
        >
          <template #icon>
            <XIcon class="w-3 h-3" />
          </template>
        </NButton>
      </template>
    </NInput>

    <Transition name="fade-slide-up" :appear="true">
      <div
        v-show="showMenu"
        v-zindexable="{ enabled: true }"
        class="absolute top-[36px] w-full bg-gray-100 shadow-xl origin-top-left rounded-[3px] overflow-clip"
        @wheel="console.log('wheel', $event)"
      >
        <ScopeMenu
          :show="menuView === 'scope'"
          :params="params"
          :options="visibleScopeOptions"
          :menu-index="menuIndex"
          @select-scope="selectScope"
          @hover-item="menuIndex = $event"
        />
        <ValueMenu
          :show="menuView === 'value'"
          :params="params"
          :scope-option="currentScopeOption"
          :value-options="visibleValueOptions"
          :menu-index="menuIndex"
          @select-value="selectValue"
          @hover-item="menuIndex = $event"
        />
      </div>
    </Transition>
  </div>
</template>

<script lang="ts" setup>
import { useElementSize } from "@vueuse/core";
import { cloneDeep, last } from "lodash-es";
import { FilterIcon } from "lucide-vue-next";
import { XIcon } from "lucide-vue-next";
import { matchSorter } from "match-sorter";
import { InputInst, NInput } from "naive-ui";
import scrollIntoView from "scroll-into-view-if-needed";
import { zindexable as vZindexable } from "vdirs";
import {
  reactive,
  watch,
  onMounted,
  ref,
  toRef,
  computed,
  nextTick,
} from "vue";
import {
  SearchParams,
  SearchScope,
  SearchScopeId,
  emptySearchParams,
  getValueFromSearchParams,
  minmax,
  upsertScope,
} from "@/utils";
import ScopeMenu from "./ScopeMenu.vue";
import ScopeTags from "./ScopeTags.vue";
import ValueMenu from "./ValueMenu.vue";
import { useSearchScopeOptions } from "./useSearchScopeOptions";

const props = withDefaults(
  defineProps<{
    placeholder?: string | undefined;
    params: SearchParams;
    readonlyScopes?: SearchScope[];
    autofocus?: boolean;
    supportOptionIdList?: SearchScopeId[];
  }>(),
  {
    readonlyScopes: () => [],
    supportOptionIdList: () => [],
    autofocus: false,
    placeholder: undefined,
  }
);

const emit = defineEmits<{
  (event: "update:params", params: SearchParams): void;
  (event: "select-unsupported-scope", id: SearchScopeId): void;
}>();

interface LocalState {
  searchText: string;
  showSearchScopes: boolean;
  currentScope?: SearchScopeId;
}

const defaultSearchParams = () => {
  const params = emptySearchParams();
  props.readonlyScopes.forEach((s) => {
    params.scopes.push({ ...s });
  });
  return params;
};

const buildSearchTextByParams = (params: SearchParams | undefined): string => {
  const prefix = (params?.scopes ?? [])
    .map((scope) => `${scope.id}:${scope.value}`)
    .join(" ");
  const query = params?.query ?? "";
  if (!prefix && !query) {
    return "";
  }
  return `${prefix} ${query}`;
};

const state = reactive<LocalState>({
  searchText: buildSearchTextByParams(props.params),
  showSearchScopes: props.autofocus,
});
const containerRef = ref<HTMLElement>();
const tagsContainerRef = ref<HTMLElement>();
const inputText = ref(props.params.query);
const inputRef = ref<InputInst>();
const menuIndex = ref(0);
const { width: containerWidth } = useElementSize(containerRef);
const focusedTagId = ref<SearchScopeId>();
const readonlyScopeIds = computed(() => {
  return new Set(props.readonlyScopes.map((s) => s.id));
});
const editableScopes = computed(() => {
  return props.params.scopes.filter((s) => !readonlyScopeIds.value.has(s.id));
});

watch(
  () => state.showSearchScopes,
  (show) => {
    if (show) state.currentScope = undefined;
  }
);

watch(
  () => props.params,
  (params) => {
    state.searchText = buildSearchTextByParams(params);
  }
);

const {
  menuView,
  fullScopeOptions,
  availableScopeOptions,
  currentScope,
  currentScopeOption,
  valueOptions,
} = useSearchScopeOptions(toRef(props, "params"), props.supportOptionIdList);

const visibleScopeOptions = computed(() => {
  if (currentScopeOption.value) {
    return [currentScopeOption.value];
  }

  const keyword = inputText.value.trim().replace(/:.*$/, "").toLowerCase();
  if (!keyword) return availableScopeOptions.value;

  const filtered = matchSorter(availableScopeOptions.value, keyword, {
    keys: ["id", "title", "description"],
  });

  return filtered;
});

const visibleValueOptions = computed(() => {
  if (!currentScope.value) return [];
  const scopePrefix = `${currentScope.value}:`;
  const keyword = inputText.value
    .trim()
    .toLowerCase()
    .substring(scopePrefix.length);

  if (!keyword) return valueOptions.value;

  // Apply multiple segments of keyword splitted by whitespace
  const terms = keyword.split(/\s+/g);
  const filtered = terms.reduceRight((options, term) => {
    return matchSorter(options, term, { keys: ["value", "keywords"] });
  }, valueOptions.value);

  const currentValue = getValueFromSearchParams(
    props.params,
    currentScope.value
  );
  const option = valueOptions.value.find((opt) => opt.value === currentValue);
  if (currentValue && option) {
    // If we have current value, put it to the first if it doesn't match the keyword
    const index = filtered.findIndex((opt) => opt.value === currentValue);
    if (index < 0) {
      filtered.unshift(option);
    }
  }
  return filtered;
});

const visibleOptions = computed(() => {
  return menuView.value === "scope"
    ? visibleScopeOptions.value
    : menuView.value === "value"
    ? visibleValueOptions.value
    : ([] as unknown[]);
});

const showMenu = computed(() => {
  if (menuView.value === "scope") {
    return visibleScopeOptions.value.length > 0;
  }
  if (menuView.value === "value") {
    return true;
  }
  return false;
});

const clearable = computed(() => {
  return (
    props.params.query.trim().length > 0 || editableScopes.value.length > 0
  );
});

const hideMenu = () => {
  nextTick(() => {
    menuView.value = undefined;
    currentScope.value = undefined;
    focusedTagId.value = undefined;
  });
};

const moveMenuIndex = (delta: -1 | 1) => {
  const options = visibleOptions.value;
  if (options.length === 0) return;

  const target = minmax(menuIndex.value + delta, 0, options.length - 1);
  menuIndex.value = target;
};

const removeScope = (id: SearchScopeId) => {
  const updated = upsertScope(props.params, {
    id,
    value: "",
  });
  emit("update:params", updated);
};
const selectScope = (
  id: SearchScopeId | undefined,
  value: string | undefined = undefined
) => {
  currentScope.value = id;
  if (id) {
    menuView.value = "value";
    // Fill-in the scope prefix if needed
    if (!inputText.value.startsWith(`${id}:`)) {
      inputText.value = `${id}:${value ?? ""}`;
    }
    scrollScopeTagIntoViewIfNeeded(id);
  } else {
    menuView.value = "scope";
  }
};
const selectValue = (value: string) => {
  const id = currentScope.value;
  if (!id) {
    menuView.value = undefined;
    return;
  }
  const updated = upsertScope(props.params, {
    id,
    value,
  });
  updated.query = "";
  inputText.value = "";
  selectScope(undefined);
  emit("update:params", updated);

  scrollScopeTagIntoViewIfNeeded(id);
  hideMenu();
};
const selectScopeFromTag = (id: SearchScopeId) => {
  if (fullScopeOptions.value.find((opt) => opt.id === id)) {
    // For AdvancedSearchBox supported scopes
    selectScope(id);
    return;
  }

  // Unsupported scope for AdvancedSearchBox
  // emit an event and wish the parent UI can handle this
  emit("select-unsupported-scope", id);
  hideMenu();
};

const maybeSelectMatchedScope = () => {
  if (!menuView.value || menuView.value === "scope") {
    const matchedScope = visibleScopeOptions.value.find((opt) =>
      inputText.value.startsWith(`${opt.id}:`)
    );
    if (matchedScope) {
      // select the scope if the inputText matches its prefix
      selectScope(matchedScope.id);
      return true;
    }
    if (!menuView.value) {
      // Show scope menu if none of the menus are shown
      menuView.value = "scope";
      return true;
    }
  }
  return false;
};
const maybeDeselectMismatchedScope = () => {
  if (menuView.value === "value" && currentScope.value) {
    if (!inputText.value.startsWith(`${currentScope.value}:`)) {
      // de-select current scope since the inputText doesn't match its prefix.
      menuView.value = "scope";
      selectScope(undefined);
      return true;
    }
  }
  return false;
};

const maybeEmitIncompleteValue = () => {
  const updated = cloneDeep(props.params);
  updated.query = inputText.value;
  emit("update:params", updated);
};

const handleInputClick = () => {
  maybeSelectMatchedScope();
  maybeDeselectMismatchedScope();
  maybeEmitIncompleteValue();
};

const handleKeyDown = (e: KeyboardEvent) => {
  if (e.isComposing) return;
  if (e.defaultPrevented) return;
  const { key } = e;
  if (key === "Backspace" && inputText.value === "") {
    // Pressing "backspace" when the input box is empty
    if (focusedTagId.value) {
      e.stopPropagation();
      e.preventDefault();
      // Delete the focusedTag if it exists
      const id = focusedTagId.value;
      focusedTagId.value = undefined;
      removeScope(id);
      return;
    } else {
      e.stopPropagation();
      e.preventDefault();
      // Otherwise mark the last tag as 'focusedTag'
      const id = last(props.params.scopes)?.id;
      if (id) {
        focusedTagId.value = id;
        scrollScopeTagIntoViewIfNeeded(id);
      }
      return;
    }
  }
  focusedTagId.value = undefined;

  if (key === "ArrowUp") {
    moveMenuIndex(-1);
    e.preventDefault();
    return;
  }
  if (key === "ArrowDown") {
    moveMenuIndex(1);
    e.preventDefault();
    return;
  }
};

const handleKeyUp = (e: KeyboardEvent) => {
  if (e.isComposing) return;
  if (e.defaultPrevented) return;
  const { key } = e;
  if (key === "Escape") {
    maybeEmitIncompleteValue();
    menuView.value = undefined;
    return;
  }
  if (key === "Backspace" && inputText.value === "") {
    // backspace key might be processed by KeyDown
    if (focusedTagId.value) {
      return;
    }
  }
  if (maybeSelectMatchedScope()) {
    maybeEmitIncompleteValue();
    return;
  }
  if (maybeDeselectMismatchedScope()) {
    maybeEmitIncompleteValue();
    return;
  }
  if (key === "Enter") {
    // Press enter to select scope (dive into the next step)
    // or select value
    const index = menuIndex.value;
    if (menuView.value === "scope") {
      const option = visibleScopeOptions.value[index];
      if (option) {
        selectScope(option.id);
        maybeEmitIncompleteValue();
        return;
      }
    }
    if (menuView.value === "value") {
      const option = visibleValueOptions.value[index];
      if (option) {
        selectValue(option.value);
        return;
      }
    }
  }

  maybeEmitIncompleteValue();
};

const handleClear = () => {
  const params = defaultSearchParams();
  emit("update:params", params);
  hideMenu();
};

const scrollScopeTagIntoViewIfNeeded = (id: SearchScopeId) => {
  nextTick(() => {
    const tagsContainerEl = tagsContainerRef.value;
    if (!tagsContainerEl) return;
    const tagEl = tagsContainerEl.querySelector(
      `[data-search-scope-id="${id}"]`
    );
    if (tagEl) {
      scrollIntoView(tagEl, {
        scrollMode: "if-needed",
      });
    }
  });
};

onMounted(() => {
  if (props.autofocus) {
    inputRef.value?.inputElRef?.focus();
  }
});
watch(menuView, () => {
  focusedTagId.value = undefined;
  menuIndex.value = 0;
  if (menuView.value === "value" && currentScope.value) {
    const value = getValueFromSearchParams(props.params, currentScope.value);
    if (value) {
      const index = valueOptions.value.findIndex(
        (option) => option.value === value
      );
      if (index >= 0) menuIndex.value = index;
    }
  }
});
watch(visibleScopeOptions, (newOptions, oldOptions) => {
  if (menuView.value !== "scope") return;
  const highlightedScope = oldOptions[menuIndex.value]?.id;
  if (highlightedScope) {
    const index = newOptions.findIndex((opt) => opt.id === highlightedScope);
    if (index >= 0) {
      menuIndex.value = index;
      return;
    }
  }
  menuIndex.value = minmax(menuIndex.value, 0, newOptions.length - 1);
});
watch(visibleValueOptions, (newOptions, oldOptions) => {
  if (menuView.value !== "value") return;
  const highlightedValue = oldOptions[menuIndex.value]?.value;
  if (highlightedValue) {
    const index = newOptions.findIndex((opt) => opt.value === highlightedValue);
    if (index >= 0) {
      menuIndex.value = index;
      return;
    }
  }
  menuIndex.value = minmax(menuIndex.value, 0, newOptions.length - 1);
});
watch(
  () => props.params,
  (params) => {
    inputText.value = params.query;
  }
);
</script>

<style lang="postcss" scoped>
.bb-advanced-issue-search-box
  .bb-advanced-issue-search-box__input
  :deep(.n-input__input) {
  @apply flex flex-row items-center;
}
</style>
