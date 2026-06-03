<template>
    <FlowPlaygroundToggle
        v-if="isEditTab && isPlaygroundAllowed && editorIsAllowedEdit && !deleted"
    />
    <NavBarActions :loading="tab === 'logs' && logsStore.logs === undefined">
        <Dashboards
            v-if="showDashboards"
            @dashboard="onSelectDashboard"
        />
        <NavBarAction
            v-if="deleted"
            :icon="BackupRestore"
            :label="t('restore')"
            @click="restoreFlow"
        />
        <NavBarAction
            v-if="canEdit && !deleted && !isEditTab"
            :icon="Pencil"
            :label="t('edit flow')"
            @click="editFlow"
        />
        <NavBarAction
            v-if="tab === 'logs' && hasLogs"
            :icon="TrashCan"
            :label="t('delete logs')"
            @click="deleteLogs"
        />

        <NavBarAction
            v-if="isEditTab && canEdit && !deleted && !flowStore.isCreating && editorHaveChange"
            :icon="PlayBoxOutline"
            :label="t('save_and_execute')"
            :disabled="editorHasErrors || editorIsReadOnly"
            @click="editorSaveAndExecute"
        />
        <NavBarAction
            v-if="isEditTab && canEdit && !deleted && !flowStore.isCreating"
            :icon="ContentCopy"
            :label="t('copy')"
            @click="editorCopyFlow"
        />
        <NavBarAction
            v-if="isEditTab && editorIsAllowedEdit && !deleted"
            :icon="Download"
            :label="t('flow_export')"
            @click="editorExportYaml"
        />
        <NavBarAction
            v-if="isEditTab && canDelete && !deleted && !flowStore.isCreating"
            :icon="Delete"
            :label="t('delete')"
            @click="editorDeleteFlow"
        />

        <template #primary>
            <KsDropdown
                v-if="isEditTab && editorIsAllowedEdit && !deleted"
                splitButton
                :type="editorIsPlaygroundEnabled ? undefined : 'primary'"
                :class="{'el-button--playground': editorIsPlaygroundEnabled, 'is-save-disabled': isSaveButtonDisabled}"
                @click="onMainSaveClick"
                @command="onSaveDropdownCommand"
            >
                <component :is="currentSaveActionMeta.icon" class="me-1" />
                {{ t(currentSaveActionMeta.labelKey) }}
                <template #dropdown>
                    <KsDropdownMenu>
                        <KsDropdownItem
                            v-for="opt in saveActionOptions"
                            :key="opt.value"
                            :command="opt.value"
                            :class="{'is-active': currentSaveAction === opt.value}"
                        >
                            <component :is="opt.icon" class="me-2" />
                            {{ t(opt.labelKey) }}
                        </KsDropdownItem>
                    </KsDropdownMenu>
                </template>
            </KsDropdown>

            <TriggerFlow
                v-if="shouldShowExecute"
                :iconOnly="isEditTab"
                type="primary"
                :flowId="flow?.id"
                :namespace="flow?.namespace"
                :flowSource="flow?.source"
            />
        </template>
    </NavBarActions>
</template>

<script setup lang="ts">
    import {computed, ref} from "vue"
    import {useI18n} from "vue-i18n"
    import {useRoute, useRouter} from "vue-router"
    import {useFlowStore} from "../../../stores/flow"
    import {flowYamlUtils as YAML_UTILS} from "@kestra-io/topology"
    import Pencil from "vue-material-design-icons/Pencil.vue"
    import BackupRestore from "vue-material-design-icons/BackupRestore.vue"
    import TrashCan from "vue-material-design-icons/TrashCan.vue"
    import ContentCopy from "vue-material-design-icons/ContentCopy.vue"
    import ContentSave from "vue-material-design-icons/ContentSave.vue"
    import FileDocumentEditOutline from "vue-material-design-icons/FileDocumentEditOutline.vue"
    import Download from "vue-material-design-icons/Download.vue"
    import Delete from "vue-material-design-icons/Delete.vue"
    import PlayBoxOutline from "vue-material-design-icons/PlayBoxOutline.vue"
    import NavBarActions from "../../../components/layout/NavBarActions.vue"
    import NavBarAction from "../../../components/layout/NavBarAction.vue"
    import FlowPlaygroundToggle from "../../../components/inputs/FlowPlaygroundToggle.vue"
    import TriggerFlow from "../../../components/flows/TriggerFlow.vue"
    import Dashboards from "../../../components/dashboard/components/selector/Selector.vue"
    import {ALLOWED_CREATION_ROUTES} from "../../../components/dashboard/composables/useDashboards"
    import resource from "../../../models/resource"
    import action from "../../../models/action"
    import {useAuthStore} from "override/stores/auth"
    import {useUnsavedChangesStore} from "../../../stores/unsavedChanges"
    import {useDashboardStore} from "../../../stores/dashboard.ts"
    import {useLogsStore} from "../../../stores/logs"
    import {useToast} from "../../../utils/toast"
    import {useFlowEditorActions} from "../../../components/flows/useFlowEditorActions"
    import {saveDefaultActions, storageKeys} from "../../../utils/constants"

    const {t} = useI18n({useScope: "global"})

    const unsavedChangesStore = useUnsavedChangesStore()
    const flowStore = useFlowStore()
    const logsStore = useLogsStore()
    const router = useRouter()
    const route = useRoute()
    const toast = useToast()

    const flow = computed(() => flowStore.flow)
    const deleted = computed(() => flow.value?.deleted || false)
    const tab = computed(() => route.params?.tab as string)
    const isEditTab = computed(() => tab.value === "edit" || flowStore.isCreating)

    const authStore = useAuthStore()
    const dashboardStore = useDashboardStore()

    const {
        haveChange: editorHaveChange,
        canSave: editorCanSave,
        hasErrors: editorHasErrors,
        isReadOnly: editorIsReadOnly,
        isAllowedEdit: editorIsAllowedEdit,
        isPlaygroundAllowed,
        isPlaygroundEnabled: editorIsPlaygroundEnabled,
        save: editorSave,
        saveAsDraft: editorSaveAsDraft,
        saveAndExecute: editorSaveAndExecute,
        exportYaml: editorExportYaml,
        copyFlow: editorCopyFlow,
        deleteFlow: editorDeleteFlow,
    } = useFlowEditorActions()

    // --- Save split-button dropdown ---
    type SaveAction = typeof saveDefaultActions[keyof typeof saveDefaultActions]

    const saveActionOptions: Array<{value: SaveAction; labelKey: string; icon: any; action: () => void}> = [
        {value: saveDefaultActions.SAVE, labelKey: "save", icon: ContentSave, action: editorSave},
        {value: saveDefaultActions.SAVE_AS_DRAFT, labelKey: "save_as_draft", icon: FileDocumentEditOutline, action: editorSaveAsDraft},
    ]

    function readSaveDefault(): SaveAction {
        const stored = localStorage.getItem(storageKeys.SAVE_DEFAULT_ACTION) as SaveAction | null
        return saveActionOptions.some(o => o.value === stored) ? (stored as SaveAction) : saveDefaultActions.SAVE
    }

    const currentSaveAction = ref<SaveAction>(readSaveDefault())
    const currentSaveActionMeta = computed(() => saveActionOptions.find(o => o.value === currentSaveAction.value) ?? saveActionOptions[0])

    const isSaveButtonDisabled = computed(() => {
        if (!editorCanSave.value || editorIsReadOnly.value) return true
        if (currentSaveAction.value === saveDefaultActions.SAVE_AS_DRAFT) return false
        return editorHasErrors.value
    })

    function onMainSaveClick() {
        if (isSaveButtonDisabled.value) return
        currentSaveActionMeta.value.action()
    }

    function onSaveDropdownCommand(command: SaveAction) {
        currentSaveAction.value = command
    }

    const onSelectDashboard = (value: any) => {
        const key = dashboardStore.getUserDashboardStorageKey(route)
        localStorage.setItem(key, value)
        router.replace({
            params: {...route.params, dashboard: value},
        })
    }

    const showDashboards = computed(() =>
        tab.value === "overview" && ALLOWED_CREATION_ROUTES.includes(String(route.name)),
    )

    const canExecute = computed(() =>
        flow.value && authStore.user?.isAllowed(resource.EXECUTION, action.CREATE, flow.value.namespace),
    )

    const shouldShowExecute = computed(() => {
        if (!flow.value || deleted.value) return false
        if (flowStore.isCreating) return false
        if (!canExecute.value) return false
        if (!isEditTab.value && tab.value === "apps") return false
        return true
    })

    const canEdit = computed(() =>
        authStore.user?.isAllowed(resource.FLOW, action.UPDATE, flow.value?.namespace),
    )

    const canDelete = computed(() =>
        authStore.user?.isAllowed(resource.FLOW, action.DELETE, flow.value?.namespace),
    )

    const editFlow = () => {
        router.push({
            name: "flows/update",
            params: {
                namespace: flow.value?.namespace,
                id: flow.value?.id,
                tab: "edit",
                tenant: route.params.tenant,
            },
        })
    }

    const hasLogs = computed(() =>
        logsStore.logs !== undefined && logsStore.logs.length > 0,
    )

    const deleteLogs = () => {
        toast.confirm(
            t("delete_all_logs"),
            async () => {
                if (!flow.value) return
                return logsStore.deleteLogs({
                    namespace: flow.value.namespace,
                    flowId: flow.value.id,
                })
            },
        )
    }

    const restoreFlow = () => {
        flowStore.createFlow({
            flow: YAML_UTILS.deleteMetadata(flow.value?.source, "deleted"),
        }).then(() => {
            unsavedChangesStore.unsavedChange = false
            router.go(0)
        })
    }


</script>

<style scoped lang="scss">
    .is-save-disabled {
        :deep(.el-button:not(.el-dropdown__caret-button)) {
            opacity: var(--el-disabled-opacity, 0.5);
            cursor: not-allowed;
            pointer-events: none;
        }
    }
</style>
