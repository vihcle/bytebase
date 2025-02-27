<template>
  <DataExportButton
    size="tiny"
    :file-type="'zip'"
    :support-formats="[
      ExportFormat.CSV,
      ExportFormat.JSON,
      ExportFormat.SQL,
      ExportFormat.XLSX,
    ]"
    @export="handleExportData"
  />
</template>

<script lang="ts" setup>
import { BinaryLike } from "node:crypto";
import { useProjectIamPolicyStore } from "@/store";
import { useExportData } from "@/store/modules/export";
import { ExportFormat } from "@/types/proto/v1/common";
import { ExportRecord } from "./types";

const props = defineProps<{
  exportRecord: ExportRecord;
}>();

const projectIamPolicyStore = useProjectIamPolicyStore();
const { exportData } = useExportData();

const handleExportData = async (
  { format, password }: { format: ExportFormat; password: string },
  callback: (content: BinaryLike | Blob, format: ExportFormat) => void
) => {
  const exportRecord = props.exportRecord;
  const database = exportRecord.database;

  const content = await exportData({
    database: database.name,
    instance: database.instance,
    format,
    statement: exportRecord.statement,
    limit: exportRecord.maxRowCount,
    admin: false,
    password,
  });

  callback(content, format);

  // Fetch the latest iam policy.
  await projectIamPolicyStore.fetchProjectIamPolicy(database.project, true);
};
</script>
