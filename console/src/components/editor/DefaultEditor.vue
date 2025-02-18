<script lang="ts" setup>
import {
  Extension,
  RichTextEditor,
  ExtensionBlockquote,
  ExtensionBold,
  ExtensionBulletList,
  ExtensionCode,
  ExtensionDocument,
  ExtensionDropcursor,
  ExtensionGapcursor,
  ExtensionHardBreak,
  ExtensionHeading,
  ExtensionHistory,
  ExtensionHorizontalRule,
  ExtensionItalic,
  ExtensionOrderedList,
  ExtensionStrike,
  ExtensionText,
  ExtensionImage,
  ExtensionTaskList,
  ExtensionLink,
  ExtensionTextAlign,
  ExtensionUnderline,
  ExtensionTable,
  ExtensionSubscript,
  ExtensionSuperscript,
  ExtensionPlaceholder,
  ExtensionHighlight,
  ExtensionCommands,
  ExtensionIframe,
  ExtensionVideo,
  ExtensionAudio,
  ExtensionCodeBlock,
  ExtensionFontSize,
  ExtensionColor,
  ExtensionIndent,
  lowlight,
  type AnyExtension,
  Editor,
  ToolboxItem,
  ExtensionDraggable,
  ExtensionColumns,
  ExtensionColumn,
  ExtensionNodeSelected,
  ExtensionTrailingNode,
  ToolbarItem,
  Plugin,
  PluginKey,
  DecorationSet,
  ExtensionListKeymap,
  ExtensionSearchAndReplace,
} from "@halo-dev/richtext-editor";
// ui custom extension
import { UiExtensionImage, UiExtensionUpload } from "./extensions";
import {
  IconCalendar,
  IconCharacterRecognition,
  IconFolder,
  IconLink,
  IconUserFollow,
  VTabItem,
  VTabs,
} from "@halo-dev/components";
import AttachmentSelectorModal from "@console/modules/contents/attachments/components/AttachmentSelectorModal.vue";
import ExtensionCharacterCount from "@tiptap/extension-character-count";
import MdiFormatHeader1 from "~icons/mdi/format-header-1";
import MdiFormatHeader2 from "~icons/mdi/format-header-2";
import MdiFormatHeader3 from "~icons/mdi/format-header-3";
import MdiFormatHeader4 from "~icons/mdi/format-header-4";
import MdiFormatHeader5 from "~icons/mdi/format-header-5";
import MdiFormatHeader6 from "~icons/mdi/format-header-6";
import RiLayoutRightLine from "~icons/ri/layout-right-line";
import {
  inject,
  markRaw,
  ref,
  watch,
  onMounted,
  shallowRef,
  type ComputedRef,
} from "vue";
import { formatDatetime } from "@/utils/date";
import { useAttachmentSelect } from "./composables/use-attachment";
import type { Attachment } from "@halo-dev/api-client";
import { useI18n } from "vue-i18n";
import { i18n } from "@/locales";
import { OverlayScrollbarsComponent } from "overlayscrollbars-vue";
import { usePluginModuleStore } from "@/stores/plugin";
import type { AttachmentLike, PluginModule } from "@halo-dev/console-shared";
import { useDebounceFn, useLocalStorage } from "@vueuse/core";
import { onBeforeUnmount } from "vue";
import { usePermission } from "@/utils/permission";
import type { AxiosRequestConfig } from "axios";
import { getContents } from "./utils/attachment";

const { t } = useI18n();
const { currentUserHasPermission } = usePermission();

const props = withDefaults(
  defineProps<{
    raw?: string;
    content: string;
    uploadImage?: (
      file: File,
      options?: AxiosRequestConfig
    ) => Promise<Attachment>;
  }>(),
  {
    raw: "",
    content: "",
    uploadImage: undefined,
  }
);

const emit = defineEmits<{
  (event: "update:raw", value: string): void;
  (event: "update:content", value: string): void;
  (event: "update", value: string): void;
}>();

const owner = inject<ComputedRef<string | undefined>>("owner");
const publishTime = inject<ComputedRef<string | undefined>>("publishTime");
const permalink = inject<ComputedRef<string | undefined>>("permalink");

declare module "@halo-dev/richtext-editor" {
  interface Commands<ReturnType> {
    global: {
      openAttachmentSelector: (
        callback: (attachments: AttachmentLike[]) => void,
        options?: {
          accepts?: string[];
          min?: number;
          max?: number;
        }
      ) => ReturnType;
    };
  }
}

interface HeadingNode {
  id: string;
  level: number;
  text: string;
}

const headingIcons = {
  1: markRaw(MdiFormatHeader1),
  2: markRaw(MdiFormatHeader2),
  3: markRaw(MdiFormatHeader3),
  4: markRaw(MdiFormatHeader4),
  5: markRaw(MdiFormatHeader5),
  6: markRaw(MdiFormatHeader6),
};

const headingNodes = ref<HeadingNode[]>();
const selectedHeadingNode = ref<HeadingNode>();
const extraActiveId = ref("toc");
const attachmentSelectorModal = ref(false);

const { onAttachmentSelect, attachmentResult } = useAttachmentSelect();

const editor = shallowRef<Editor>();

const { pluginModules } = usePluginModuleStore();

const showSidebar = useLocalStorage("halo:editor:show-sidebar", true);

const initAttachmentOptions = {
  accepts: ["*/*"],
  min: undefined,
  max: undefined,
};
const attachmentOptions = ref<{
  accepts?: string[];
  min?: number;
  max?: number;
}>(initAttachmentOptions);

const handleCloseAttachmentSelectorModal = () => {
  attachmentOptions.value = initAttachmentOptions;
};

onMounted(() => {
  const extensionsFromPlugins: AnyExtension[] = [];
  pluginModules.forEach((pluginModule: PluginModule) => {
    const { extensionPoints } = pluginModule;
    if (!extensionPoints?.["default:editor:extension:create"]) {
      return;
    }

    const extensions = extensionPoints[
      "default:editor:extension:create"
    ]() as [];

    extensionsFromPlugins.push(...extensions);
  });

  // debounce OnUpdate
  const debounceOnUpdate = useDebounceFn(() => {
    const html = editor.value?.getHTML() + "";
    emit("update:raw", html);
    emit("update:content", html);
    emit("update", html);
  }, 250);

  const image = currentUserHasPermission(["uc:attachments:manage"])
    ? UiExtensionImage
    : ExtensionImage;

  editor.value = new Editor({
    content: props.raw,
    extensions: [
      ExtensionBlockquote,
      ExtensionBold,
      ExtensionBulletList,
      ExtensionCode,
      ExtensionDocument,
      ExtensionDropcursor.configure({
        width: 2,
        class: "dropcursor",
        color: "skyblue",
      }),
      ExtensionGapcursor,
      ExtensionHardBreak,
      ExtensionHeading,
      ExtensionHistory,
      ExtensionHorizontalRule,
      ExtensionItalic,
      ExtensionOrderedList,
      ExtensionStrike,
      ExtensionText,
      image.configure({
        inline: true,
        allowBase64: false,
        HTMLAttributes: {
          loading: "lazy",
        },
        uploadImage: props.uploadImage,
      }),
      ExtensionTaskList,
      ExtensionLink.configure({
        autolink: true,
        openOnClick: false,
      }),
      ExtensionTextAlign.configure({
        types: ["heading", "paragraph"],
      }),
      ExtensionUnderline,
      ExtensionTable.configure({
        resizable: true,
      }),
      ExtensionSubscript,
      ExtensionSuperscript,
      ExtensionPlaceholder.configure({
        placeholder: t(
          "core.components.default_editor.extensions.placeholder.options.placeholder"
        ),
      }),
      ExtensionHighlight,
      ExtensionCommands,
      ExtensionCodeBlock.configure({
        lowlight,
      }),
      ExtensionIframe,
      ExtensionVideo,
      ExtensionAudio,
      ExtensionCharacterCount,
      ExtensionFontSize,
      ExtensionColor,
      ExtensionIndent,
      ...extensionsFromPlugins,
      Extension.create({
        addGlobalAttributes() {
          return [
            {
              types: ["heading"],
              attributes: {
                id: {
                  default: null,
                },
              },
            },
          ];
        },
      }),
      Extension.create({
        addOptions() {
          // If user has no permission to view attachments, return
          if (!currentUserHasPermission(["system:attachments:view"])) {
            return this;
          }

          return {
            getToolboxItems({ editor }: { editor: Editor }) {
              return [
                {
                  priority: 0,
                  component: markRaw(ToolboxItem),
                  props: {
                    editor,
                    icon: markRaw(IconFolder),
                    title: i18n.global.t(
                      "core.components.default_editor.toolbox.attachment"
                    ),
                    action: () => {
                      editor.commands.openAttachmentSelector((attachment) => {
                        editor
                          .chain()
                          .focus()
                          .insertContent(getContents(attachment))
                          .run();
                      });
                      return true;
                    },
                  },
                },
              ];
            },
            getToolbarItems({ editor }: { editor: Editor }) {
              return {
                priority: 1000,
                component: markRaw(ToolbarItem),
                props: {
                  editor,
                  isActive: showSidebar.value,
                  icon: markRaw(RiLayoutRightLine),
                  title: i18n.global.t(
                    "core.components.default_editor.toolbox.show_hide_sidebar"
                  ),
                  action: () => {
                    showSidebar.value = !showSidebar.value;
                  },
                },
              };
            },
          };
        },
        addCommands() {
          return {
            openAttachmentSelector: (callback, options) => () => {
              if (options) {
                attachmentOptions.value = options;
              }
              attachmentSelectorModal.value = true;
              attachmentResult.updateAttachment = (
                attachments: AttachmentLike[]
              ) => {
                callback(attachments);
              };
              return true;
            },
          };
        },
      }),
      ExtensionDraggable,
      ExtensionColumns,
      ExtensionColumn,
      ExtensionNodeSelected,
      ExtensionTrailingNode,
      Extension.create({
        addProseMirrorPlugins() {
          return [
            new Plugin({
              key: new PluginKey("get-heading-id"),
              props: {
                decorations: (state) => {
                  const headings: HeadingNode[] = [];
                  const { doc } = state;
                  doc.descendants((node) => {
                    if (node.type.name === ExtensionHeading.name) {
                      headings.push({
                        level: node.attrs.level,
                        text: node.textContent,
                        id: node.attrs.id,
                      });
                    }
                  });
                  headingNodes.value = headings;
                  if (!selectedHeadingNode.value) {
                    selectedHeadingNode.value = headings[0];
                  }
                  return DecorationSet.empty;
                },
              },
            }),
          ];
        },
      }),
      ExtensionListKeymap,
      UiExtensionUpload,
      ExtensionSearchAndReplace,
    ],
    autofocus: "start",
    onUpdate: () => {
      debounceOnUpdate();
    },
  });
});

onBeforeUnmount(() => {
  editor.value?.destroy();
});

const handleSelectHeadingNode = (node: HeadingNode) => {
  selectedHeadingNode.value = node;
  document.getElementById(node.id)?.scrollIntoView({ behavior: "smooth" });
};

watch(
  () => props.raw,
  () => {
    if (props.raw !== editor.value?.getHTML()) {
      editor.value?.commands.setContent(props.raw);
    }
  },
  {
    immediate: true,
  }
);

// fixme: temporary solution
const currentLocale = i18n.global.locale.value as
  | "zh-CN"
  | "en"
  | "zh"
  | "en-US";
</script>

<template>
  <div>
    <AttachmentSelectorModal
      v-bind="attachmentOptions"
      v-model:visible="attachmentSelectorModal"
      @select="onAttachmentSelect"
      @close="handleCloseAttachmentSelectorModal"
    />
    <RichTextEditor v-if="editor" :editor="editor" :locale="currentLocale">
      <template v-if="showSidebar" #extra>
        <OverlayScrollbarsComponent
          element="div"
          :options="{ scrollbars: { autoHide: 'scroll' } }"
          class="h-full border-l bg-white"
          defer
        >
          <VTabs v-model:active-id="extraActiveId" type="outline">
            <VTabItem
              id="toc"
              :label="$t('core.components.default_editor.tabs.toc.title')"
            >
              <div class="p-1 pt-0">
                <ul v-if="headingNodes?.length" class="space-y-1">
                  <li
                    v-for="(node, index) in headingNodes"
                    :key="index"
                    :class="[
                      { 'bg-gray-100': node.id === selectedHeadingNode?.id },
                    ]"
                    class="group cursor-pointer truncate rounded-base px-1.5 py-1 text-sm text-gray-600 hover:bg-gray-100 hover:text-gray-900"
                    @click="handleSelectHeadingNode(node)"
                  >
                    <div
                      :style="{
                        paddingLeft: `${(node.level - 1) * 0.8}rem`,
                      }"
                      class="flex items-center gap-2"
                    >
                      <component
                        :is="headingIcons[node.level]"
                        class="h-4 w-4 rounded-sm bg-gray-100 p-0.5 group-hover:bg-white"
                        :class="[
                          {
                            '!bg-white': node.id === selectedHeadingNode?.id,
                          },
                        ]"
                      />
                      <span class="flex-1 truncate">{{ node.text }}</span>
                    </div>
                  </li>
                </ul>
                <div v-else class="flex flex-col items-center py-10">
                  <span class="text-sm text-gray-600">
                    {{ $t("core.components.default_editor.tabs.toc.empty") }}
                  </span>
                </div>
              </div>
            </VTabItem>
            <VTabItem
              id="information"
              :label="$t('core.components.default_editor.tabs.detail.title')"
            >
              <div class="flex flex-col gap-2 p-1 pt-0">
                <div class="grid grid-cols-2 gap-2">
                  <div
                    class="group flex cursor-pointer flex-col gap-y-5 rounded-md bg-gray-100 px-1.5 py-1 transition-all"
                  >
                    <div class="flex items-center justify-between">
                      <div
                        class="text-sm text-gray-500 group-hover:text-gray-900"
                      >
                        {{
                          $t(
                            "core.components.default_editor.tabs.detail.fields.character_count"
                          )
                        }}
                      </div>
                      <div class="rounded bg-gray-200 p-0.5">
                        <IconCharacterRecognition
                          class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                        />
                      </div>
                    </div>
                    <div class="text-base font-medium text-gray-900">
                      {{ editor.storage.characterCount.characters() }}
                    </div>
                  </div>
                  <div
                    class="group flex cursor-pointer flex-col gap-y-5 rounded-md bg-gray-100 px-1.5 py-1 transition-all"
                  >
                    <div class="flex items-center justify-between">
                      <div
                        class="text-sm text-gray-500 group-hover:text-gray-900"
                      >
                        {{
                          $t(
                            "core.components.default_editor.tabs.detail.fields.word_count"
                          )
                        }}
                      </div>
                      <div class="rounded bg-gray-200 p-0.5">
                        <IconCharacterRecognition
                          class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                        />
                      </div>
                    </div>
                    <div class="text-base font-medium text-gray-900">
                      {{ editor.storage.characterCount.words() }}
                    </div>
                  </div>
                </div>

                <div v-if="publishTime" class="grid grid-cols-1 gap-2">
                  <div
                    class="group flex cursor-pointer flex-col gap-y-5 rounded-md bg-gray-100 px-1.5 py-1 transition-all"
                  >
                    <div class="flex items-center justify-between">
                      <div
                        class="text-sm text-gray-500 group-hover:text-gray-900"
                      >
                        {{
                          $t(
                            "core.components.default_editor.tabs.detail.fields.publish_time"
                          )
                        }}
                      </div>
                      <div class="rounded bg-gray-200 p-0.5">
                        <IconCalendar
                          class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                        />
                      </div>
                    </div>
                    <div class="text-base font-medium text-gray-900">
                      {{
                        formatDatetime(publishTime) ||
                        $t(
                          "core.components.default_editor.tabs.detail.fields.draft"
                        )
                      }}
                    </div>
                  </div>
                </div>
                <div v-if="owner" class="grid grid-cols-1 gap-2">
                  <div
                    class="group flex cursor-pointer flex-col gap-y-5 rounded-md bg-gray-100 px-1.5 py-1 transition-all"
                  >
                    <div class="flex items-center justify-between">
                      <div
                        class="text-sm text-gray-500 group-hover:text-gray-900"
                      >
                        {{
                          $t(
                            "core.components.default_editor.tabs.detail.fields.owner"
                          )
                        }}
                      </div>
                      <div class="rounded bg-gray-200 p-0.5">
                        <IconUserFollow
                          class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                        />
                      </div>
                    </div>
                    <div class="text-base font-medium text-gray-900">
                      {{ owner }}
                    </div>
                  </div>
                </div>
                <div v-if="permalink" class="grid grid-cols-1 gap-2">
                  <div
                    class="group flex cursor-pointer flex-col gap-y-5 rounded-md bg-gray-100 px-1.5 py-1 transition-all"
                  >
                    <div class="flex items-center justify-between">
                      <div
                        class="text-sm text-gray-500 group-hover:text-gray-900"
                      >
                        {{
                          $t(
                            "core.components.default_editor.tabs.detail.fields.permalink"
                          )
                        }}
                      </div>
                      <div class="rounded bg-gray-200 p-0.5">
                        <IconLink
                          class="h-4 w-4 text-gray-600 group-hover:text-gray-900"
                        />
                      </div>
                    </div>
                    <div>
                      <a
                        :href="permalink"
                        :title="permalink"
                        target="_blank"
                        class="text-sm text-gray-900 hover:text-blue-600"
                      >
                        {{ permalink }}
                      </a>
                    </div>
                  </div>
                </div>
              </div>
            </VTabItem>
          </VTabs>
        </OverlayScrollbarsComponent>
      </template>
    </RichTextEditor>
  </div>
</template>
