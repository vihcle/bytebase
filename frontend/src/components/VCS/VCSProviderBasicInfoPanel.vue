<!-- eslint-disable vue/no-mutating-props -->
<template>
  <div class="textlabel">
    {{ $t("gitops.setting.add-git-provider.choose") }}
    <span class="text-red-600">*</span>
  </div>
  <div class="flex flex-wrap pt-4 radio-set-row gap-4">
    <NRadioGroup v-model:value="config.uiType">
      <NRadio
        v-for="vcsWithUIType in vcsListByUIType"
        :key="vcsWithUIType.uiType"
        :value="vcsWithUIType.uiType"
        @change="changeUIType()"
      >
        <div class="flex space-x-1">
          <VCSIcon custom-class="h-6" :type="vcsWithUIType.type" />
          <span class="whitespace-nowrap">
            {{ vcsWithUIType.title }}
          </span>
        </div>
      </NRadio>
    </NRadioGroup>
  </div>
  <div class="mt-6 pt-6 border-t border-block-border textlabel">
    {{ instanceUrlLabel }} <span class="text-red-600">*</span>
  </div>
  <p class="mt-1 textinfolabel">
    {{
      $t("gitops.setting.add-git-provider.basic-info.gitlab-instance-url-label")
    }}
  </p>
  <BBTextField
    class="mt-2 w-full"
    :value="config.instanceUrl"
    :placeholder="instanceUrlPlaceholder"
    :disabled="instanceUrlDisabled"
    @update:value="changeUrl($event)"
  />
  <p v-if="state.showUrlError" class="mt-2 text-sm text-error">
    {{ $t("gitops.setting.add-git-provider.basic-info.instance-url-error") }}
  </p>
  <div class="mt-4 textlabel">
    {{ $t("gitops.setting.add-git-provider.basic-info.display-name") }}
  </div>
  <p class="mt-1 textinfolabel">
    {{ $t("gitops.setting.add-git-provider.basic-info.display-name-label") }}
  </p>
  <BBTextField
    v-model:value="config.name"
    class="mt-2 w-full"
    :placeholder="namePlaceholder"
  />
</template>

<script lang="ts" setup>
import isEmpty from "lodash-es/isEmpty";
import { NRadio, NRadioGroup } from "naive-ui";
import { computed, onUnmounted, reactive } from "vue";
import { useI18n } from "vue-i18n";
import { TEXT_VALIDATION_DELAY, VCSConfig } from "@/types";
import { ExternalVersionControl_Type } from "@/types/proto/v1/externalvs_service";
import { isUrl } from "@/utils";
import { vcsListByUIType } from "./utils";

interface LocalState {
  urlValidationTimer?: ReturnType<typeof setTimeout>;
  showUrlError: boolean;
}

const props = defineProps<{
  config: VCSConfig;
}>();

const { t } = useI18n();
const state = reactive<LocalState>({
  showUrlError:
    !isEmpty(props.config.instanceUrl) && !isUrl(props.config.instanceUrl),
});

onUnmounted(() => {
  if (state.urlValidationTimer) {
    clearInterval(state.urlValidationTimer);
  }
});

const namePlaceholder = computed((): string => {
  if (props.config.type === ExternalVersionControl_Type.GITLAB) {
    if (props.config.uiType == "GITLAB_SELF_HOST") {
      return t("gitops.setting.add-git-provider.gitlab-self-host");
    } else if (props.config.uiType == "GITLAB_COM") {
      return "GitLab.com";
    }
  } else if (props.config.type === ExternalVersionControl_Type.GITHUB) {
    if (props.config.uiType == "GITHUB_COM") {
      return "GitHub.com";
    } else if (props.config.uiType === "GITHUB_ENTERPRISE") {
      return "Self Host GitHub Enterprise";
    }
  } else if (props.config.type === ExternalVersionControl_Type.BITBUCKET) {
    return "Bitbucket.org";
  } else if (props.config.type === ExternalVersionControl_Type.AZURE_DEVOPS) {
    return "Azure DevOps";
  }
  return "";
});

const instanceUrlLabel = computed((): string => {
  switch (props.config.type) {
    case ExternalVersionControl_Type.GITLAB:
      return t(
        "gitops.setting.add-git-provider.basic-info.gitlab-instance-url"
      );
    case ExternalVersionControl_Type.GITHUB:
      return t(
        "gitops.setting.add-git-provider.basic-info.github-instance-url"
      );
    case ExternalVersionControl_Type.BITBUCKET:
      return t(
        "gitops.setting.add-git-provider.basic-info.bitbucket-instance-url"
      );
    case ExternalVersionControl_Type.AZURE_DEVOPS:
      return t("gitops.setting.add-git-provider.basic-info.azure-instance-url");
    default:
      return "";
  }
});

const instanceUrlPlaceholder = computed((): string => {
  if (props.config.type === ExternalVersionControl_Type.GITLAB) {
    if (props.config.uiType == "GITLAB_SELF_HOST") {
      return "https://gitlab.example.com";
    } else if (props.config.uiType == "GITLAB_COM") {
      return "https://gitlab.com";
    }
  } else if (props.config.type === ExternalVersionControl_Type.GITHUB) {
    if (props.config.uiType == "GITHUB_COM") {
      return "https://github.com";
    } else if (props.config.uiType == "GITHUB_ENTERPRISE") {
      return "https://github.companyname.com";
    }
  } else if (props.config.type === ExternalVersionControl_Type.BITBUCKET) {
    return "https://bitbucket.org";
  }
  return "";
});

// github.com instance url is always https://github.com
const instanceUrlDisabled = computed((): boolean => {
  return (
    (props.config.type === ExternalVersionControl_Type.GITHUB &&
      props.config.uiType == "GITHUB_COM") ||
    props.config.type === ExternalVersionControl_Type.BITBUCKET ||
    props.config.type === ExternalVersionControl_Type.AZURE_DEVOPS ||
    (props.config.type === ExternalVersionControl_Type.GITLAB &&
      props.config.uiType == "GITLAB_COM")
  );
});

const changeUrl = (value: string) => {
  // eslint-disable-next-line vue/no-mutating-props
  props.config.instanceUrl = value;

  if (state.urlValidationTimer) {
    clearInterval(state.urlValidationTimer);
  }
  // If text becomes valid, we immediately clear the error.
  // otherwise, we delay TEXT_VALIDATION_DELAY to do the validation in case there is continous keystroke.
  if (isUrl(props.config.instanceUrl)) {
    state.showUrlError = false;
  } else {
    state.urlValidationTimer = setTimeout(() => {
      // If error is already displayed, we hide the error only if there is valid input.
      // Otherwise, we hide the error if input is either empty or valid.
      if (state.showUrlError) {
        state.showUrlError = !isUrl(props.config.instanceUrl);
      } else {
        state.showUrlError =
          !isEmpty(props.config.instanceUrl) &&
          !isUrl(props.config.instanceUrl);
      }
    }, TEXT_VALIDATION_DELAY);
  }
};

// FIXME: Unexpected mutation of "config" prop. Do we care?
/* eslint-disable vue/no-mutating-props */
const changeUIType = () => {
  switch (props.config.uiType) {
    case "GITLAB_SELF_HOST":
      props.config.type = ExternalVersionControl_Type.GITLAB;
      props.config.instanceUrl = "";
      props.config.name = t("gitops.setting.add-git-provider.gitlab-self-host");
      break;
    case "GITLAB_COM":
      props.config.type = ExternalVersionControl_Type.GITLAB;
      props.config.instanceUrl = "https://gitlab.com";
      props.config.name = "GitLab.com";
      break;
    case "GITHUB_COM":
      props.config.type = ExternalVersionControl_Type.GITHUB;
      props.config.instanceUrl = "https://github.com";
      props.config.name = "GitHub.com";
      break;
    case "GITHUB_ENTERPRISE":
      props.config.type = ExternalVersionControl_Type.GITHUB;
      props.config.instanceUrl = "";
      props.config.name = "Self Host GitHub Enterprise";
      break;
    case "BITBUCKET_ORG":
      props.config.type = ExternalVersionControl_Type.BITBUCKET;
      props.config.instanceUrl = "https://bitbucket.org";
      props.config.name = "Bitbucket.org";
      break;
    case "AZURE_DEVOPS":
      props.config.type = ExternalVersionControl_Type.AZURE_DEVOPS;
      props.config.instanceUrl = "https://dev.azure.com";
      props.config.name = "Azure DevOps";
      break;
    default:
      break;
  }
};
</script>
