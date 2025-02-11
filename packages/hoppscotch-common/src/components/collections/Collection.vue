<template>
  <div class="flex flex-col" :class="[{ 'bg-primaryLight': dragging }]">
    <div
      class="flex items-stretch group"
      @dragover.prevent
      @drop.prevent="dropEvent"
      @dragover="dragging = true"
      @drop="dragging = false"
      @dragleave="dragging = false"
      @dragend="dragging = false"
      @contextmenu.prevent="options?.tippy.show()"
    >
      <span
        class="flex items-center justify-center px-4 cursor-pointer"
        @click="emit('toggle-children')"
      >
        <component
          :is="collectionIcon"
          class="svg-icons"
          :class="{ 'text-accent': isSelected }"
        />
      </span>
      <span
        class="flex flex-1 min-w-0 py-2 pr-2 transition cursor-pointer group-hover:text-secondaryDark"
        @click="emit('toggle-children')"
      >
        <span class="truncate" :class="{ 'text-accent': isSelected }">
          {{ collectionName }}
        </span>
      </span>
      <div v-if="!hasNoTeamAccess" class="flex">
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :icon="IconFilePlus"
          :title="t('request.new')"
          class="hidden group-hover:inline-flex"
          @click="emit('add-request')"
        />
        <HoppButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          :icon="IconFolderPlus"
          :title="t('folder.new')"
          class="hidden group-hover:inline-flex"
          @click="emit('add-folder')"
        />
        <span>
          <tippy
            ref="options"
            interactive
            trigger="click"
            theme="popover"
            :on-shown="() => tippyActions!.focus()"
          >
            <HoppButtonSecondary
              v-tippy="{ theme: 'tooltip' }"
              :title="t('action.more')"
              :icon="IconMoreVertical"
            />
            <template #content="{ hide }">
              <div
                ref="tippyActions"
                class="flex flex-col focus:outline-none"
                tabindex="0"
                @keyup.r="requestAction?.$el.click()"
                @keyup.n="folderAction?.$el.click()"
                @keyup.e="edit?.$el.click()"
                @keyup.delete="deleteAction?.$el.click()"
                @keyup.x="exportAction?.$el.click()"
                @keyup.escape="hide()"
              >
                <HoppSmartItem
                  ref="requestAction"
                  :icon="IconFilePlus"
                  :label="t('request.new')"
                  :shortcut="['R']"
                  @click="
                    () => {
                      emit('add-request')
                      hide()
                    }
                  "
                />
                <HoppSmartItem
                  ref="folderAction"
                  :icon="IconFolderPlus"
                  :label="t('folder.new')"
                  :shortcut="['N']"
                  @click="
                    () => {
                      emit('add-folder')
                      hide()
                    }
                  "
                />
                <HoppSmartItem
                  ref="edit"
                  :icon="IconEdit"
                  :label="t('action.edit')"
                  :shortcut="['E']"
                  @click="
                    () => {
                      emit('edit-collection')
                      hide()
                    }
                  "
                />
                <HoppSmartItem
                  ref="exportAction"
                  :icon="IconDownload"
                  :label="t('export.title')"
                  :shortcut="['X']"
                  :loading="exportLoading"
                  @click="
                    () => {
                      emit('export-data'),
                        collectionsType === 'my-collections' ? hide() : null
                    }
                  "
                />
                <HoppSmartItem
                  ref="deleteAction"
                  :icon="IconTrash2"
                  :label="t('action.delete')"
                  :shortcut="['⌫']"
                  @click="
                    () => {
                      emit('remove-collection')
                      hide()
                    }
                  "
                />
              </div>
            </template>
          </tippy>
        </span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import IconCheckCircle from "~icons/lucide/check-circle"
import IconFolderPlus from "~icons/lucide/folder-plus"
import IconFilePlus from "~icons/lucide/file-plus"
import IconMoreVertical from "~icons/lucide/more-vertical"
import IconDownload from "~icons/lucide/download"
import IconTrash2 from "~icons/lucide/trash-2"
import IconEdit from "~icons/lucide/edit"
import IconFolder from "~icons/lucide/folder"
import IconFolderOpen from "~icons/lucide/folder-open"
import { PropType, ref, computed, watch } from "vue"
import { HoppCollection, HoppRESTRequest } from "@hoppscotch/data"
import { useI18n } from "@composables/i18n"
import { TippyComponent } from "vue-tippy"
import { TeamCollection } from "~/helpers/teams/TeamCollection"

type CollectionType = "my-collections" | "team-collections"
type FolderType = "collection" | "folder"

const t = useI18n()

const props = defineProps({
  data: {
    type: Object as PropType<HoppCollection<HoppRESTRequest> | TeamCollection>,
    default: () => ({}),
    required: true,
  },
  collectionsType: {
    type: String as PropType<CollectionType>,
    default: "my-collections",
    required: true,
  },
  /**
   * Collection component can be used for both collections and folders.
   * folderType is used to determine which one it is.
   */
  folderType: {
    type: String as PropType<FolderType>,
    default: "collection",
    required: true,
  },
  isOpen: {
    type: Boolean,
    default: false,
    required: true,
  },
  isSelected: {
    type: Boolean,
    default: false,
    required: false,
  },
  exportLoading: {
    type: Boolean,
    default: false,
    required: false,
  },
  hasNoTeamAccess: {
    type: Boolean,
    default: false,
    required: false,
  },
})

const emit = defineEmits<{
  (event: "toggle-children"): void
  (event: "add-request"): void
  (event: "add-folder"): void
  (event: "edit-collection"): void
  (event: "export-data"): void
  (event: "remove-collection"): void
  (event: "drop-event", payload: DataTransfer): void
}>()

const tippyActions = ref<TippyComponent | null>(null)
const requestAction = ref<HTMLButtonElement | null>(null)
const folderAction = ref<HTMLButtonElement | null>(null)
const edit = ref<HTMLButtonElement | null>(null)
const deleteAction = ref<HTMLButtonElement | null>(null)
const exportAction = ref<HTMLButtonElement | null>(null)
const options = ref<TippyComponent | null>(null)

const dragging = ref(false)

const collectionIcon = computed(() => {
  if (props.isSelected) return IconCheckCircle
  else if (!props.isOpen) return IconFolder
  else if (props.isOpen) return IconFolderOpen
  else return IconFolder
})

const collectionName = computed(() => {
  if ((props.data as HoppCollection<HoppRESTRequest>).name)
    return (props.data as HoppCollection<HoppRESTRequest>).name
  else return (props.data as TeamCollection).title
})

watch(
  () => props.exportLoading,
  (val) => {
    if (!val) {
      options.value!.tippy.hide()
    }
  }
)

const dropEvent = ({ dataTransfer }: DragEvent) => {
  if (dataTransfer) {
    dragging.value = !dragging.value
    emit("drop-event", dataTransfer)
  }
}
</script>
