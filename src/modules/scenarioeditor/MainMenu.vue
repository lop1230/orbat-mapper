<script setup lang="ts">
import {
  DropdownMenu,
  DropdownMenuCheckboxItem,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuRadioGroup,
  DropdownMenuRadioItem,
  DropdownMenuSeparator,
  DropdownMenuShortcut,
  DropdownMenuSub,
  DropdownMenuSubContent,
  DropdownMenuSubTrigger,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { ChevronDownIcon } from "@heroicons/vue/20/solid";
import { useUiStore } from "@/stores/uiStore";
import { LANDING_PAGE_ROUTE, MAP_EDIT_MODE_ROUTE } from "@/router/names";

import type { ScenarioActions, UiAction } from "@/types/constants";
import { useRoute } from "vue-router";
import { useMapSettingsStore } from "@/stores/mapSettingsStore";
import { storeToRefs } from "pinia";
import { useMeasurementsStore } from "@/stores/geoStore";
import { breakpointsTailwind, useBreakpoints } from "@vueuse/core";
import { injectStrict } from "@/utils";
import { activeScenarioKey } from "@/components/injects";
import { useOwnedUndoRedo } from "@/modules/scenarioeditor/useOwnedUndoRedo";
import { useShareHistory } from "@/composables/scenarioShare";
import { LockIcon, MoonStarIcon, SunIcon } from "@lucide/vue";
import { UseDark } from "@vueuse/components";
import TerrainMenu from "@/modules/maplibreview/TerrainMenu.vue";
import dayjs from "dayjs";
import relativeTime from "dayjs/plugin/relativeTime";

dayjs.extend(relativeTime);

const breakpoints = useBreakpoints(breakpointsTailwind);
const isMobile = breakpoints.smallerOrEqual("md");

const emit = defineEmits<{
  action: [value: ScenarioActions];
  uiAction: [value: UiAction];
}>();

const activeScenario = injectStrict(activeScenarioKey);
const {
  io: { hasDistinctOpenedBaseline, hasSavedBaseline },
} = activeScenario;
// Routed through the armed-tool owner, like the toolbar's Undo button and Ctrl+Z.
const { undo, redo, canUndo, canRedo } = useOwnedUndoRedo(activeScenario);

const route = useRoute();
const uiSettings = useUiStore();
const mapSettings = useMapSettingsStore();

const { coordinateFormat, showLocation, showScaleLine, showDayNightTerminator } =
  storeToRefs(useMapSettingsStore());

const { measurementUnit } = storeToRefs(useMeasurementsStore());

const { history: shareHistory, clearHistory: clearShareHistory } = useShareHistory();
</script>

<template>
  <DropdownMenu>
    <DropdownMenuTrigger as="div" class="relative">
      <button class="group flex items-center">
        <svg
          class="block h-7 w-auto shrink-0 fill-[#aab074]/90 stroke-gray-900 dark:fill-gray-700 dark:stroke-gray-300"
          stroke="currentColor"
          viewBox="41 41 118 118"
        >
          <path d="m100 45 55 25v60l-55 25-55-25V70z" stroke-width="6" />
          <path d="m45 70 110 60m-110 0 110-60" stroke-width="6" />
          <circle cx="100" cy="70" r="10" class="fill-gray-900 dark:fill-gray-300" />
        </svg>
        <span
          class="ml-2 hidden font-medium tracking-tight whitespace-nowrap xl:inline-flex"
          >ORBAT-Mapper</span
        >
        <ChevronDownIcon
          class="text-muted-foreground group-hover:text-muted-foreground/80 ml-1 h-5 w-5"
          aria-hidden="true"
        />
      </button>
    </DropdownMenuTrigger>
    <DropdownMenuContent class="" align="start" :side-offset="10">
      <DropdownMenuItem as-child>
        <router-link :to="{ name: LANDING_PAGE_ROUTE }" class="font-medium"
          >首页
        </router-link>
      </DropdownMenuItem>

      <DropdownMenuSeparator />
      <UseDark v-if="isMobile" v-slot="{ isDark, toggleDark }">
        <DropdownMenuItem @select="toggleDark()">
          <SunIcon v-if="isDark" class="mr-2 h-4 w-4" />
          <MoonStarIcon v-else class="mr-2 h-4 w-4" />
          <span>{{ isDark ? "亮色模式" : "暗色模式" }}</span>
        </DropdownMenuItem>
      </UseDark>
      <DropdownMenuSeparator v-if="isMobile" />
      <DropdownMenuItem @select="emit('uiAction', 'showSearch')"
        >搜索
        <DropdownMenuShortcut class="ml-4">Ctrl/⌘ K</DropdownMenuShortcut>
      </DropdownMenuItem>
      <DropdownMenuSeparator />
      <DropdownMenuSub>
        <DropdownMenuSubTrigger>文件</DropdownMenuSubTrigger>
        <DropdownMenuSubContent>
          <DropdownMenuItem @select="emit('action', 'exportJson')"
            >下载方案
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'exportEncrypted')"
            >下载加密方案…
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'save')">
            保存方案
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'loadNew')">
            加载方案…
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'createNew')">
            新建方案…
          </DropdownMenuItem>

          <DropdownMenuSeparator />
          <DropdownMenuItem @select="emit('action', 'share')">
            在线分享方案…
          </DropdownMenuItem>

          <DropdownMenuItem @select="emit('action', 'shareAsUrl')">
            分享方案为链接…
          </DropdownMenuItem>
          <DropdownMenuSub v-if="shareHistory.length > 0">
            <DropdownMenuSubTrigger>最近分享</DropdownMenuSubTrigger>
            <DropdownMenuSubContent class="w-80">
              <DropdownMenuItem v-for="item in shareHistory" :key="item.id" as-child>
                <a
                  :href="item.url"
                  target="_blank"
                  class="flex flex-col items-start gap-1 overflow-hidden"
                >
                  <div class="flex w-full items-center gap-2">
                    <LockIcon v-if="item.encrypted" class="h-3 w-3 shrink-0" />
                    <span class="truncate">{{ item.name }}</span>
                    <span class="text-muted-foreground ml-auto shrink-0 text-xs"
                      >{{ dayjs(item.timestamp).fromNow() }}
                    </span>
                  </div>
                  <span class="text-muted-foreground w-full truncate text-xs">{{
                    item.url
                  }}</span>
                </a>
              </DropdownMenuItem>
              <DropdownMenuSeparator />
              <DropdownMenuItem @select="clearShareHistory"
                >清除历史记录</DropdownMenuItem
              >
            </DropdownMenuSubContent>
          </DropdownMenuSub>

          <DropdownMenuItem @select="emit('action', 'export')">
            导出方案数据…
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'exportToImage')">
            将地图导出为图片
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'import')">
            导入数据…
          </DropdownMenuItem>
          <DropdownMenuSeparator />
          <DropdownMenuItem @select="emit('action', 'duplicate')">
            复制方案
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'showInfo')">
            显示方案信息
          </DropdownMenuItem>
        </DropdownMenuSubContent>
      </DropdownMenuSub>
      <DropdownMenuSub>
        <DropdownMenuSubTrigger><span>编辑</span></DropdownMenuSubTrigger>
        <DropdownMenuSubContent>
          <DropdownMenuItem @select="undo()" :disabled="!canUndo">
            撤销
            <DropdownMenuShortcut class="ml-4">Ctrl/⌘ Z</DropdownMenuShortcut>
          </DropdownMenuItem>
          <DropdownMenuItem @select="redo()" :disabled="!canRedo">
            重做
            <DropdownMenuShortcut class="ml-4">Ctrl/⌘ shift Z</DropdownMenuShortcut>
          </DropdownMenuItem>
          <DropdownMenuSeparator />
          <DropdownMenuItem
            v-if="hasDistinctOpenedBaseline"
            @select="emit('action', 'restoreOriginal')"
          >
            恢复为打开时的状态
          </DropdownMenuItem>
          <DropdownMenuItem
            @select="emit('action', 'revertToSaved')"
            :disabled="!hasSavedBaseline"
          >
            恢复为已保存的版本
          </DropdownMenuItem>
          <DropdownMenuSeparator />
          <DropdownMenuItem @select="emit('action', 'exportToClipboard')">
            复制方案到剪贴板
          </DropdownMenuItem>
          <DropdownMenuItem @select="emit('action', 'pasteFromClipboard')">
            从剪贴板粘贴
            <DropdownMenuShortcut class="ml-4">Ctrl/⌘ V</DropdownMenuShortcut>
          </DropdownMenuItem>
        </DropdownMenuSubContent>
      </DropdownMenuSub>
      <DropdownMenuSub>
        <DropdownMenuSubTrigger><span class="mr-4">视图</span></DropdownMenuSubTrigger>
        <DropdownMenuSubContent>
          <DropdownMenuCheckboxItem v-model="uiSettings.showToolbar" @select.prevent
            >地图工具栏
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem v-model="uiSettings.showTimeline" @select.prevent
            >时间轴
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem
            v-if="!isMobile"
            v-model="uiSettings.showLeftPanel"
            @select.prevent
            >ORBAT 面板
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem
            v-model="uiSettings.showOrbatBreadcrumbs"
            @select.prevent
            >单位面包屑</DropdownMenuCheckboxItem
          >
          <DropdownMenuSeparator />
          <DropdownMenuCheckboxItem v-model="showScaleLine" @select.prevent>
            比例尺
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem v-model="showLocation" @select.prevent>
            指针位置
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem v-model="showDayNightTerminator" @select.prevent>
            昼夜晨昏线
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem
            v-model="mapSettings.mapUnitLabelBelow"
            @select.prevent
          >
            在图标下方显示单位标签
          </DropdownMenuCheckboxItem>
          <DropdownMenuCheckboxItem
            v-model="mapSettings.mapWrapUnitLabels"
            @select.prevent
            v-if="mapSettings.mapUnitLabelBelow"
          >
            长单位标签换行
          </DropdownMenuCheckboxItem>
          <!-- Only the MapLibre map in map edit mode renders terrain. -->
          <TerrainMenu kind="dropdown" v-if="route.name === MAP_EDIT_MODE_ROUTE" />

          <DropdownMenuSub>
            <DropdownMenuSubTrigger inset
              ><span class="pr-4">测量单位</span></DropdownMenuSubTrigger
            >
            <DropdownMenuSubContent>
              <DropdownMenuRadioGroup v-model="measurementUnit">
                <DropdownMenuRadioItem value="metric" @select.prevent
                  >公制
                </DropdownMenuRadioItem>
                <DropdownMenuRadioItem value="imperial" @select.prevent
                  >英制
                </DropdownMenuRadioItem>
                <DropdownMenuRadioItem value="nautical" @select.prevent
                  >航海单位
                </DropdownMenuRadioItem>
              </DropdownMenuRadioGroup>
            </DropdownMenuSubContent>
          </DropdownMenuSub>
          <DropdownMenuSub>
            <DropdownMenuSubTrigger inset>坐标格式</DropdownMenuSubTrigger>
            <DropdownMenuSubContent>
              <DropdownMenuRadioGroup v-model="coordinateFormat">
                <DropdownMenuRadioItem value="dms" @select.prevent
                  >度、分、秒
                </DropdownMenuRadioItem>
                <DropdownMenuRadioItem value="dd" @select.prevent
                  >十进制度
                </DropdownMenuRadioItem>
                <DropdownMenuRadioItem value="MGRS" @select.prevent
                  >MGRS
                </DropdownMenuRadioItem>
              </DropdownMenuRadioGroup>
            </DropdownMenuSubContent>
          </DropdownMenuSub>
        </DropdownMenuSubContent>
      </DropdownMenuSub>
      <DropdownMenuSeparator />
      <DropdownMenuSub>
        <DropdownMenuSubTrigger>工具</DropdownMenuSubTrigger>
        <DropdownMenuSubContent>
          <DropdownMenuItem @select="emit('action', 'browseSymbols')"
            >浏览符号
          </DropdownMenuItem>
        </DropdownMenuSubContent>
      </DropdownMenuSub>
      <DropdownMenuSub>
        <DropdownMenuSubTrigger>帮助</DropdownMenuSubTrigger>
        <DropdownMenuSubContent>
          <DropdownMenuItem as-child
            ><a
              :href="
                route.meta.helpUrl ||
                'https://docs.orbat-mapper.app/guide/about-orbat-mapper'
              "
              target="_blank"
            >
              文档
            </a></DropdownMenuItem
          >
          <DropdownMenuItem @select="emit('uiAction', 'showKeyboardShortcuts')"
            >键盘快捷键
            <DropdownMenuShortcut class="ml-4">?</DropdownMenuShortcut>
          </DropdownMenuItem>
        </DropdownMenuSubContent>
      </DropdownMenuSub>
    </DropdownMenuContent>
  </DropdownMenu>
</template>
