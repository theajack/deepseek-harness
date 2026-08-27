<!--
 * @Author: theajack theajack@qq.com
 * @Date: 2026-08-26 00:33:28
 * @LastEditors: theajack theajack@qq.com
 * @LastEditTime: 2026-08-27 08:46:00
 * @FilePath: \deepseek-harness\apps\desktop\src\components\layout\TitleBar.vue
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";
import { Copy, Minus, Square, X } from "lucide-vue-next";
import { t } from "../../i18n";

const isMac = ref(false);
const isMaximized = ref(false);

onMounted(async () => {
  // UA 嗅探用于前端条件渲染
  isMac.value = /Mac|iPod|iPhone|iPad/.test(navigator.platform);

  if (!isMac.value) {
    const { getCurrentWindow } = await import("@tauri-apps/api/window");
    const w = getCurrentWindow();
    isMaximized.value = await w.isMaximized();
    // 窗口尺寸变化（含最大化/还原）时同步状态
    let unlisten: (() => void) | undefined;
    void w.onResized(async () => {
      isMaximized.value = await w.isMaximized();
    }).then((fn) => {
      unlisten = fn;
    });
    onUnmounted(() => unlisten?.());
  }
});

async function startDrag() {
  const { getCurrentWindow } = await import("@tauri-apps/api/window");
  await getCurrentWindow().startDragging();
}

async function win(action: "minimize" | "toggleMaximize" | "close") {
  const { getCurrentWindow } = await import("@tauri-apps/api/window");
  const w = getCurrentWindow();
  if (action === "minimize") await w.minimize();
  else if (action === "toggleMaximize") await w.toggleMaximize();
  else await w.close();
}
</script>

<template>
  <!-- 透明可拖拽顶栏 -->
  <div
    class="fixed h-3 top-0 right-0 left-0 z-30 flex cursor-move select-none items-center bg-transparent"
    data-tauri-drag-region
    @mousedown="startDrag"
  >
    <!-- 左侧留给 Mac 红绿灯（系统原生渲染） -->
    <div class="w-15 shrink-0" :data-tauri-drag-region="true" />
    <!-- 标题占位（透明可拖拽） -->
    <div class="flex-1" :data-tauri-drag-region="true" />
  </div>

  <!-- Windows 窗口控制按钮：fixed 于右上角，独立于 ChatHeader，样式对齐 header 图标 -->
  <div v-if="!isMac" class="fixed top-[10px] right-0 z-40 flex items-center gap-1 pr-3">
    <button
      class="flex h-8 w-8 items-center justify-center rounded-lg text-mid transition-colors hover:bg-ink-3 hover:text-hi"
      :title="t('titlebar.minimize')"
      @click="win('minimize')"
    >
      <Minus :size="16" :stroke-width="2" />
    </button>
    <button
      class="flex h-8 w-8 items-center justify-center rounded-lg text-mid transition-colors hover:bg-ink-3 hover:text-hi"
      :title="t(isMaximized ? 'titlebar.restore' : 'titlebar.maximize')"
      @click="win('toggleMaximize')"
    >
      <Copy v-if="isMaximized" :size="16" :stroke-width="2" />
      <Square v-else :size="16" :stroke-width="2" />
    </button>
    <button
      class="flex h-8 w-8 items-center justify-center rounded-lg text-mid transition-colors hover:bg-danger hover:text-white"
      :title="t('titlebar.close')"
      @click="win('close')"
    >
      <X :size="16" :stroke-width="2" />
    </button>
  </div>
</template>
