<template>
  <div class="flex divide-x divide-dividerLight">
    <tippy
      interactive
      trigger="click"
      theme="popover"
      :on-shown="() => envSelectorActions!.focus()"
    >
      <span
        v-tippy="{ theme: 'tooltip' }"
        :title="`${t('environment.select')}`"
        class="bg-transparent border-b border-dividerLight select-wrapper"
      >
        <HoppButtonSecondary
          :icon="IconLayers"
          :label="
            mdAndLarger
              ? selectedEnv.type !== 'NO_ENV_SELECTED'
                ? selectedEnv.name
                : `${t('environment.select')}`
              : ''
          "
          class="flex-1 !justify-start pr-8 rounded-none"
        />
      </span>
      <template #content="{ hide }">
        <div
          ref="envSelectorActions"
          role="menu"
          class="flex flex-col focus:outline-none"
          tabindex="0"
          @keyup.escape="hide()"
        >
          <HoppSmartItem
            :label="`${t('environment.no_environment')}`"
            :info-icon="
              selectedEnvironmentIndex.type === 'NO_ENV_SELECTED'
                ? IconCheck
                : undefined
            "
            :active-info-icon="
              selectedEnvironmentIndex.type === 'NO_ENV_SELECTED'
            "
            @click="
              () => {
                selectedEnvironmentIndex = { type: 'NO_ENV_SELECTED' }
                hide()
              }
            "
          />
          <HoppSmartTabs
            v-model="selectedEnvTab"
            styles="sticky overflow-x-auto my-2 border border-divider rounded flex-shrink-0 z-0 top-0 bg-primary"
            render-inactive-tabs
          >
            <HoppSmartTab
              :id="'my-environments'"
              :label="`${t('environment.my_environments')}`"
            >
              <HoppSmartItem
                v-for="(gen, index) in myEnvironments"
                :key="`gen-${index}`"
                :icon="IconLayers"
                :label="gen.name"
                :info-icon="index === selectedEnv.index ? IconCheck : undefined"
                :active-info-icon="index === selectedEnv.index"
                @click="
                  () => {
                    selectedEnvironmentIndex = {
                      type: 'MY_ENV',
                      index: index,
                    }
                    hide()
                  }
                "
              />
              <div
                v-if="myEnvironments.length === 0"
                class="flex flex-col items-center justify-center text-secondaryLight"
              >
                <img
                  :src="`/images/states/${colorMode.value}/blockchain.svg`"
                  loading="lazy"
                  class="inline-flex flex-col object-contain object-center w-16 h-16 mb-2"
                  :alt="`${t('empty.environments')}`"
                />
                <span class="pb-2 text-center">
                  {{ t("empty.environments") }}
                </span>
              </div>
            </HoppSmartTab>
            <HoppSmartTab
              :id="'team-environments'"
              :label="`${t('environment.team_environments')}`"
              :disabled="!isTeamSelected || workspace.type === 'personal'"
            >
              <div
                v-if="teamListLoading"
                class="flex flex-col items-center justify-center p-4"
              >
                <HoppSmartSpinner class="my-4" />
                <span class="text-secondaryLight">
                  {{ t("state.loading") }}
                </span>
              </div>
              <div v-if="isTeamSelected" class="flex flex-col">
                <HoppSmartItem
                  v-for="(gen, index) in teamEnvironmentList"
                  :key="`gen-team-${index}`"
                  :icon="IconLayers"
                  :label="gen.environment.name"
                  :info-icon="
                    gen.id === selectedEnv.teamEnvID ? IconCheck : undefined
                  "
                  :active-info-icon="gen.id === selectedEnv.teamEnvID"
                  @click="
                    () => {
                      selectedEnvironmentIndex = {
                        type: 'TEAM_ENV',
                        teamEnvID: gen.id,
                        teamID: gen.teamID,
                        environment: gen.environment,
                      }
                      hide()
                    }
                  "
                />
                <div
                  v-if="teamEnvironmentList.length === 0"
                  class="flex flex-col items-center justify-center text-secondaryLight"
                >
                  <img
                    :src="`/images/states/${colorMode.value}/blockchain.svg`"
                    loading="lazy"
                    class="inline-flex flex-col object-contain object-center w-16 h-16 mb-2"
                    :alt="`${t('empty.environments')}`"
                  />
                  <span class="pb-2 text-center">
                    {{ t("empty.environments") }}
                  </span>
                </div>
              </div>
              <div
                v-if="!teamListLoading && teamAdapterError"
                class="flex flex-col items-center py-4"
              >
                <icon-lucide-help-circle class="mb-4 svg-icons" />
                {{ getErrorMessage(teamAdapterError) }}
              </div>
            </HoppSmartTab>
          </HoppSmartTabs>
        </div>
      </template>
    </tippy>
    <span class="flex">
      <tippy
        interactive
        trigger="click"
        theme="popover"
        :on-shown="() => envQuickPeekActions!.focus()"
      >
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :title="`${t('environment.quick_peek')}`"
          :icon="IconEye"
          class="!px-4"
        />
        <template #content="{ hide }">
          <div
            ref="envQuickPeekActions"
            role="menu"
            class="flex flex-col focus:outline-none"
            tabindex="0"
            @keyup.escape="hide()"
          >
            <label
              class="sticky top-0 font-semibold truncate flex items-center justify-between text-secondaryDark bg-primary border border-divider rounded pl-4"
            >
              {{ t("environment.global_variables") }}
              <HoppButtonSecondary
                v-tippy="{ theme: 'tooltip' }"
                :title="t('action.edit')"
                :icon="IconEdit"
              />
            </label>
            <div
              v-if="sampleGlobalVariables.length === 0"
              class="text-secondaryLight my-2 flex flex-col flex-1 pl-4"
            >
              {{ t("environment.empty_variables") }}
            </div>
            <div v-else class="my-2 flex flex-col flex-1 space-y-2 pl-4">
              <div class="flex flex-1 space-x-4">
                <span class="w-20 truncate text-tiny font-semibold">
                  {{ t("environment.name") }}
                </span>
                <span class="w-36 truncate text-tiny font-semibold">
                  {{ t("environment.value") }}
                </span>
              </div>
              <div
                v-for="(variable, index) in sampleGlobalVariables"
                :key="index"
                class="flex flex-1 space-x-4"
              >
                <span class="text-secondaryLight w-20 truncate">
                  {{ variable.key }}
                </span>
                <span class="text-secondaryLight w-36 truncate">
                  {{ variable.value }}
                </span>
              </div>
            </div>
            <label
              class="sticky top-0 font-semibold truncate flex items-center justify-between text-secondaryDark bg-primary border border-divider rounded pl-4"
            >
              {{ t("environment.list") }}
              <HoppButtonSecondary
                v-tippy="{ theme: 'tooltip' }"
                :title="t('action.edit')"
                :icon="IconEdit"
              />
            </label>
            <div
              v-if="selectedEnv.type === 'NO_ENV_SELECTED'"
              class="text-secondaryLight my-2 flex flex-col flex-1 pl-4"
            >
              {{ t("environment.select") }}
            </div>
            <div v-else class="my-2 flex flex-col flex-1 space-y-2 pl-4">
              <div class="flex flex-1 space-x-4">
                <span class="w-20 truncate text-tiny font-semibold">
                  {{ t("environment.name") }}
                </span>
                <span class="w-36 truncate text-tiny font-semibold">
                  {{ t("environment.value") }}
                </span>
              </div>
              <div
                v-for="(variable, index) in sampleEnvironmentVariables"
                :key="index"
                class="flex flex-1 space-x-4"
              >
                <span class="text-secondaryLight w-20 truncate">
                  {{ variable.key }}
                </span>
                <span class="text-secondaryLight w-36 truncate">
                  {{ variable.value }}
                </span>
              </div>
              <div
                v-if="sampleEnvironmentVariables.length === 0"
                class="text-secondaryLight"
              >
                {{ t("environment.empty_variables") }}
              </div>
            </div>
          </div>
        </template>
      </tippy>
    </span>
  </div>
</template>

<script lang="ts" setup>
import { computed, ref, watch } from "vue"
import IconCheck from "~icons/lucide/check"
import IconLayers from "~icons/lucide/layers"
import IconEye from "~icons/lucide/eye"
import IconEdit from "~icons/lucide/edit"
import { TippyComponent } from "vue-tippy"
import { useI18n } from "~/composables/i18n"
import { GQLError } from "~/helpers/backend/GQLClient"
import { useReadonlyStream, useStream } from "~/composables/stream"
import {
  environments$,
  selectedEnvironmentIndex$,
  setSelectedEnvironmentIndex,
} from "~/newstore/environments"
import { workspaceStatus$ } from "~/newstore/workspace"
import TeamEnvironmentAdapter from "~/helpers/teams/TeamEnvironmentAdapter"
import { useColorMode } from "@composables/theming"
import { breakpointsTailwind, useBreakpoints } from "@vueuse/core"

const breakpoints = useBreakpoints(breakpointsTailwind)
const mdAndLarger = breakpoints.greater("md")

const t = useI18n()

const colorMode = useColorMode()

type EnvironmentType = "my-environments" | "team-environments"

const myEnvironments = useReadonlyStream(environments$, [])

const workspace = useReadonlyStream(workspaceStatus$, { type: "personal" })

const teamEnvListAdapter = new TeamEnvironmentAdapter(undefined)
const teamListLoading = useReadonlyStream(teamEnvListAdapter.loading$, false)
const teamAdapterError = useReadonlyStream(teamEnvListAdapter.error$, null)
const teamEnvironmentList = useReadonlyStream(
  teamEnvListAdapter.teamEnvironmentList$,
  []
)

const selectedEnvironmentIndex = useStream(
  selectedEnvironmentIndex$,
  { type: "NO_ENV_SELECTED" },
  setSelectedEnvironmentIndex
)

const isTeamSelected = computed(
  () => workspace.value.type === "team" && workspace.value.teamID !== undefined
)

const selectedEnvTab = ref<EnvironmentType>("my-environments")

watch(
  () => workspace.value,
  (newVal) => {
    if (newVal.type === "personal") {
      selectedEnvTab.value = "my-environments"
    } else {
      selectedEnvTab.value = "team-environments"
      if (newVal.teamID) {
        teamEnvListAdapter.changeTeamID(newVal.teamID)
      }
    }
  }
)

const selectedEnv = computed(() => {
  if (selectedEnvironmentIndex.value.type === "MY_ENV") {
    return {
      type: "MY_ENV",
      index: selectedEnvironmentIndex.value.index,
      name: myEnvironments.value[selectedEnvironmentIndex.value.index].name,
    }
  } else if (selectedEnvironmentIndex.value.type === "TEAM_ENV") {
    const teamEnv = teamEnvironmentList.value.find(
      (env) =>
        env.id ===
        (selectedEnvironmentIndex.value.type === "TEAM_ENV" &&
          selectedEnvironmentIndex.value.teamEnvID)
    )
    if (teamEnv) {
      return {
        type: "TEAM_ENV",
        name: teamEnv.environment.name,
        teamEnvID: selectedEnvironmentIndex.value.teamEnvID,
      }
    } else {
      return { type: "NO_ENV_SELECTED" }
    }
  } else {
    return { type: "NO_ENV_SELECTED" }
  }
})

// Template refs
const envSelectorActions = ref<TippyComponent | null>(null)
const envQuickPeekActions = ref<TippyComponent | null>(null)

const getErrorMessage = (err: GQLError<string>) => {
  if (err.type === "network_error") {
    return t("error.network_error")
  } else {
    switch (err.error) {
      case "team_environment/not_found":
        return t("team_environment.not_found")
      default:
        return t("error.something_went_wrong")
    }
  }
}

const sampleGlobalVariables = [
  {
    key: "Staging",
    value: "https://staging.example.com",
  },
  {
    key: "Production",
    value: "https://example.com",
  },
  {
    key: "Development",
    value: "http://localhost:3000",
  },
]

const sampleEnvironmentVariables = [
  {
    key: "Staging",
    value: "https://staging.example.com",
  },
  {
    key: "Production",
    value: "https://example.com",
  },
  {
    key: "Development",
    value: "http://localhost:3000",
  },
  {
    key: "Testing",
    value: "http://localhost:3000",
  },
  {
    key: "Staging",
    value: "https://staging.example.com",
  },
  {
    key: "Production",
    value: "https://example.com",
  },
  {
    key: "Development",
    value: "http://localhost:3000",
  },
  {
    key: "Testing",
    value: "http://localhost:3000",
  },
]
</script>
