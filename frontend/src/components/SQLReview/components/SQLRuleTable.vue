<template>
  <div>
    <template v-for="category in categoryList" :key="category.id">
      <div class="flex my-3 items-center">
        <span class="text-xl text-main font-semibold">
          {{ $t(`sql-review.category.${category.id.toLowerCase()}`) }}
        </span>
        <span class="text-control-light text-md ml-1"
          >({{ category.ruleList.length }})</span
        >
      </div>
      <BBGrid
        :column-list="columnList"
        :data-source="category.ruleList"
        :row-clickable="false"
        class="border hidden lg:grid"
      >
        <template #item="{ item: rule }: { item: RuleTemplate }">
          <div class="bb-grid-cell justify-center">
            <NSwitch
              size="small"
              :disabled="!editable || !isRuleAvailable(rule)"
              :value="rule.level !== SQLReviewRuleLevel.DISABLED"
              @update-value="(val) => toggleActivity(rule, val)"
            />
          </div>
          <div class="bb-grid-cell gap-x-1">
            <NTooltip
              v-if="!isRuleAvailable(rule)"
              trigger="hover"
              :show-arrow="false"
            >
              <template #trigger>
                <div class="flex justify-center">
                  <heroicons-outline:exclamation
                    class="h-5 w-5 text-yellow-600"
                  />
                </div>
              </template>
              <span class="whitespace-nowrap">
                {{
                  $t("sql-review.not-available-for-free", {
                    plan: $t(
                      `subscription.plan.${planTypeToString(currentPlan)}.title`
                    ),
                  })
                }}
              </span>
            </NTooltip>
            <span>
              {{ getRuleLocalization(rule.type).title }}
            </span>
            <a
              :href="`https://www.bytebase.com/docs/sql-review/review-rules#${rule.type}`"
              target="_blank"
              class="flex flex-row space-x-2 items-center text-base text-gray-500 hover:text-gray-900"
            >
              <heroicons-outline:external-link class="w-4 h-4" />
            </a>
          </div>
          <div class="bb-grid-cell gap-x-2">
            <RuleEngineIcons :rule="rule" />
          </div>
          <div class="bb-grid-cell">
            <RuleLevelSwitch
              :level="rule.level"
              :disabled="!isRuleAvailable(rule)"
              :editable="editable"
              @level-change="$emit('level-change', rule, $event)"
            />
          </div>
          <div class="bb-grid-cell justify-center">
            <NButton
              :disabled="!isRuleAvailable(rule)"
              @click="setActiveRule(rule)"
            >
              {{ editable ? $t("common.edit") : $t("common.view") }}
            </NButton>
          </div>
          <div
            v-if="rule.comment || getRuleLocalization(rule.type).description"
            class="bb-grid-cell col-span-full pl-24 border-t-0"
          >
            <p class="w-full text-left pl-2 text-gray-500 -mt-2 mb-1">
              {{ rule.comment || getRuleLocalization(rule.type).description }}
            </p>
          </div>
        </template>
      </BBGrid>
      <div
        class="flex flex-col lg:hidden border px-2 pb-4 divide-y space-y-4 divide-block-border"
      >
        <div
          v-for="rule in category.ruleList"
          :key="rule.type"
          class="pt-4 space-y-3"
        >
          <div class="flex justify-between items-center gap-x-2">
            <div class="flex items-center gap-x-1">
              <NTooltip
                v-if="!isRuleAvailable(rule)"
                trigger="hover"
                :show-arrow="false"
              >
                <template #trigger>
                  <div class="flex justify-center">
                    <heroicons-outline:exclamation
                      class="h-5 w-5 text-yellow-600"
                    />
                  </div>
                </template>
                <span class="whitespace-nowrap">
                  {{
                    $t("sql-review.not-available-for-free", {
                      plan: $t(
                        `subscription.plan.${planTypeToString(
                          currentPlan
                        )}.title`
                      ),
                    })
                  }}
                </span>
              </NTooltip>
              <span>
                {{ getRuleLocalization(rule.type).title }}
                <a
                  :href="`https://www.bytebase.com/docs/sql-review/review-rules#${rule.type}`"
                  target="_blank"
                  class="inline-block"
                >
                  <ExternalLinkIcon class="w-4 h-4" />
                </a>
              </span>
            </div>
            <div class="flex items-center space-x-2">
              <PencilIcon
                v-if="editable"
                class="w-4 h-4"
                @click="setActiveRule(rule)"
              />
              <NSwitch
                size="small"
                :disabled="!editable || !isRuleAvailable(rule)"
                :value="rule.level !== SQLReviewRuleLevel.DISABLED"
                @update-value="(val) => toggleActivity(rule, val)"
              />
            </div>
          </div>
          <div class="flex gap-x-2 items-center">
            <RuleEngineIcons :rule="rule" />
          </div>
          <RuleLevelSwitch
            class="text-xs"
            :level="rule.level"
            :disabled="!isRuleAvailable(rule)"
            :editable="editable"
            @level-change="$emit('level-change', rule, $event)"
          />
          <p class="textinfolabel">
            {{ getRuleLocalization(rule.type).description }}
          </p>
        </div>
      </div>
    </template>

    <SQLRuleEditDialog
      v-if="state.activeRule"
      :editable="editable"
      :rule="state.activeRule"
      :disabled="!isRuleAvailable(state.activeRule)"
      @cancel="state.activeRule = undefined"
      @update:payload="updatePayload(state.activeRule!, $event)"
      @update:level="updateLevel(state.activeRule!, $event)"
      @update:comment="updateComment(state.activeRule!, $event)"
    />
  </div>
</template>

<script lang="ts" setup>
import { ExternalLinkIcon, PencilIcon } from "lucide-vue-next";
import { NSwitch } from "naive-ui";
import { computed, reactive } from "vue";
import { useI18n } from "vue-i18n";
import { BBGrid, type BBGridColumn } from "@/bbkit";
import { useCurrentPlan } from "@/store";
import {
  convertToCategoryList,
  getRuleLocalization,
  ruleIsAvailableInSubscription,
  planTypeToString,
  RuleTemplate,
} from "@/types";
import { SQLReviewRuleLevel } from "@/types/proto/v1/org_policy_service";
import { PayloadForEngine } from "./RuleConfigComponents";
import RuleLevelSwitch from "./RuleLevelSwitch.vue";
import SQLRuleEditDialog from "./SQLRuleEditDialog.vue";

type LocalState = {
  activeRule: RuleTemplate | undefined;
};

const props = withDefaults(
  defineProps<{
    ruleList?: RuleTemplate[];
    editable: boolean;
  }>(),
  {
    ruleList: () => [],
    editable: true,
  }
);

const emit = defineEmits<{
  (
    event: "payload-change",
    rule: RuleTemplate,
    payload: PayloadForEngine
  ): void;
  (event: "level-change", rule: RuleTemplate, level: SQLReviewRuleLevel): void;
  (event: "comment-change", rule: RuleTemplate, comment: string): void;
}>();

const { t } = useI18n();
const currentPlan = useCurrentPlan();
const state = reactive<LocalState>({
  activeRule: undefined,
});

const categoryList = computed(() => {
  return convertToCategoryList(props.ruleList);
});

const columnList = computed((): BBGridColumn[] => {
  const columns: BBGridColumn[] = [
    {
      title: t("sql-review.rule.active"),
      width: "6rem",
      class: "justify-center",
    },
    { title: t("common.name"), width: "1fr" },
    { title: t("common.databases"), width: "12rem" },
    { title: t("sql-review.level.name"), width: "12rem" },
    {
      title: t("common.operations"),
      width: "10rem",
      class: "justify-center",
    },
  ];
  return columns;
});

const isRuleAvailable = (rule: RuleTemplate) => {
  return ruleIsAvailableInSubscription(rule.type, currentPlan.value);
};

const setActiveRule = (rule: RuleTemplate) => {
  state.activeRule = rule;
};

const toggleActivity = (rule: RuleTemplate, on: boolean) => {
  emit(
    "level-change",
    rule,
    on ? SQLReviewRuleLevel.WARNING : SQLReviewRuleLevel.DISABLED
  );
};

const updatePayload = (rule: RuleTemplate, payload: PayloadForEngine) => {
  emit("payload-change", rule, payload);
};
const updateLevel = (rule: RuleTemplate, level: SQLReviewRuleLevel) => {
  emit("level-change", rule, level);
};
const updateComment = (rule: RuleTemplate, comment: string) => {
  emit("comment-change", rule, comment);
};
</script>
