<template>
  <div class="flex flex-col h-full py-0.5 gap-y-1">
    <div class="flex items-center gap-x-1 px-2 pt-1.5">
      <NInput
        v-model:value="keyword"
        size="small"
        :placeholder="$t('common.filter-by-name')"
        :clearable="true"
      >
        <template #prefix>
          <heroicons-outline:search class="h-5 w-5 text-gray-300" />
        </template>
      </NInput>
    </div>

    <div class="w-full h-full relative overflow-hidden">
      <template v-if="databaseMetadata && !isLoading">
        <DatabaseSchema
          :database="database"
          :database-metadata="databaseMetadata"
          :header-clickable="selected !== undefined"
          @click-header="selected = undefined"
          @select-table="handleSelectTable"
          @select-external-table="handleSelectExternalTable"
        />
        <Transition name="slide-up">
          <template v-if="selected">
            <TableSchema
              v-if="selected.table"
              class="absolute bottom-0 w-full h-[calc(100%-33px)] bg-white"
              :db="database"
              :database="databaseMetadata"
              :schema="selected.schema"
              :table="selected.table"
              @close="selected = undefined"
            />
            <ExternalTableSchema
              v-else-if="selected.externalTable"
              class="absolute bottom-0 w-full h-[calc(100%-33px)] bg-white"
              :db="database"
              :database="databaseMetadata"
              :schema="selected.schema"
              :external-table="selected.externalTable"
              @close="selected = undefined"
            />
          </template>
        </Transition>
      </template>

      <div
        v-else
        class="absolute inset-0 bg-white/50 flex flex-col items-center justify-center"
      >
        <BBSpin />
      </div>
    </div>

    <HoverPanel :offset-x="-24" :offset-y="0" />
  </div>
</template>

<script lang="ts" setup>
import { asyncComputed } from "@vueuse/core";
import { storeToRefs } from "pinia";
import { computed, ref } from "vue";
import { useDatabaseV1ByUID, useDBSchemaV1Store, useTabStore } from "@/store";
import {
  SchemaMetadata,
  TableMetadata,
  DatabaseMetadataView,
  ExternalTableMetadata,
} from "@/types/proto/v1/database_service";
import { useSQLEditorContext } from "@/views/sql-editor/context";
import DatabaseSchema from "./DatabaseSchema.vue";
import ExternalTableSchema from "./ExternalTableSchema.vue";
import { provideHoverStateContext, HoverPanel } from "./HoverPanel";
import TableSchema from "./TableSchema.vue";
import { provideSchemaPanelContext } from "./context";

const { selectedDatabaseSchemaByDatabaseName } = useSQLEditorContext();

const dbSchemaStore = useDBSchemaV1Store();
const { currentTab } = storeToRefs(useTabStore());
const conn = computed(() => currentTab.value.connection);
const { keyword } = provideSchemaPanelContext();
provideHoverStateContext();

const { database } = useDatabaseV1ByUID(computed(() => conn.value.databaseId));
const isLoading = ref(false);
const databaseMetadata = asyncComputed(
  async () => {
    const { name } = database.value;
    const databaseMetadata = await dbSchemaStore.getOrFetchDatabaseMetadata({
      database: name,
      skipCache: false,
      view: DatabaseMetadataView.DATABASE_METADATA_VIEW_FULL,
    });
    return databaseMetadata;
  },
  undefined,
  {
    evaluating: isLoading,
  }
);

const selected = computed({
  get() {
    return selectedDatabaseSchemaByDatabaseName.value.get(database.value.name);
  },
  set(selected) {
    if (!selected) {
      selectedDatabaseSchemaByDatabaseName.value.delete(database.value.name);
    } else {
      selectedDatabaseSchemaByDatabaseName.value.set(
        database.value.name,
        selected
      );
    }
  },
});

const handleSelectTable = async (
  schema: SchemaMetadata,
  table: TableMetadata
) => {
  const tableMetadata = await dbSchemaStore.getOrFetchTableMetadata({
    database: database.value.name,
    schema: schema.name,
    table: table.name,
  });
  const databaseMetadata = useDBSchemaV1Store().getDatabaseMetadata(
    database.value.name
  );

  selected.value = {
    db: database.value,
    database: databaseMetadata,
    schema,
    table: tableMetadata,
  };
};

const handleSelectExternalTable = async (
  schema: SchemaMetadata,
  externalTable: ExternalTableMetadata
) => {
  const databaseMetadata = useDBSchemaV1Store().getDatabaseMetadata(
    database.value.name
  );

  selected.value = {
    db: database.value,
    database: databaseMetadata,
    schema,
    externalTable,
  };
};
</script>
