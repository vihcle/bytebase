<template>
  <div class="overflow-hidden">
    <VirtualList
      :items="filteredColumnList"
      :key-field="`name`"
      :item-resizable="true"
      :item-size="24"
    >
      <template #default="{ item: column }: VirtualListItem">
        <div
          class="bb-column-list--column-item flex items-start mx-3 px-1 gap-2 cursor-pointer hover:bg-control-bg-hover/50"
          @mouseenter="handleMouseEnter($event, column)"
          @mouseleave="handleMouseLeave($event, column)"
        >
          <NEllipsis class="max-w-2/3">
            <!-- eslint-disable vue/no-v-html -->
            <span
              class="text-sm leading-6 text-gray-600whitespace-pre-wrap break-words flex-2 min-w-[4rem]"
              v-html="renderColumnName(column)"
            />
          </NEllipsis>
          <NEllipsis
            class="shrink-0 max-w-1/2 text-right text-sm leading-6 text-gray-400 flex-1 min-w-[4rem]"
          >
            {{ column.type }}
          </NEllipsis>
        </div>
      </template>
    </VirtualList>
  </div>
</template>

<script setup lang="ts">
import { escape } from "lodash-es";
import { NEllipsis } from "naive-ui";
import { computed, nextTick } from "vue";
import { VirtualList } from "vueuc";
import { ComposedDatabase } from "@/types";
import {
  ColumnMetadata,
  DatabaseMetadata,
  SchemaMetadata,
} from "@/types/proto/v1/database_service";
import { findAncestor, getHighlightHTMLByRegExp } from "@/utils";
import { useHoverStateContext } from "./HoverPanel";
import { useSchemaPanelContext } from "./context";

export type VirtualListItem = {
  item: ColumnMetadata;
  index: number;
};

const props = defineProps<{
  db: ComposedDatabase;
  database: DatabaseMetadata;
  schema: SchemaMetadata;
  columns: ColumnMetadata[];
}>();

const {
  state: hoverState,
  position: hoverPosition,
  update: updateHoverState,
} = useHoverStateContext();
const { keyword } = useSchemaPanelContext();
const filteredColumnList = computed(() => {
  const kw = keyword.value.toLowerCase().trim();
  if (!kw) {
    return props.columns;
  }
  return props.columns.filter((column) => {
    return column.name.toLowerCase().includes(kw);
  });
});

const renderColumnName = (column: ColumnMetadata) => {
  if (!keyword.value.trim()) {
    return column.name;
  }

  return getHighlightHTMLByRegExp(
    escape(column.name),
    escape(keyword.value.trim()),
    false /* !caseSensitive */
  );
};

const handleMouseEnter = (e: MouseEvent, column: ColumnMetadata) => {
  const { db, database, schema } = props;
  if (hoverState.value) {
    updateHoverState(
      { db, database, schema, column },
      "before",
      0 /* overrideDelay */
    );
  } else {
    updateHoverState({ db, database, schema, column }, "before");
  }
  nextTick().then(() => {
    // Find the node element and put the database panel to the top-left corner
    // of the node
    const wrapper = findAncestor(
      e.target as HTMLElement,
      ".bb-column-list--column-item"
    );
    if (!wrapper) {
      updateHoverState(undefined, "after", 0 /* overrideDelay */);
      return;
    }
    const bounding = wrapper.getBoundingClientRect();
    hoverPosition.value.x = bounding.left;
    hoverPosition.value.y = bounding.top;
  });
};

const handleMouseLeave = (e: MouseEvent, column: ColumnMetadata) => {
  updateHoverState(undefined, "after");
};
</script>
