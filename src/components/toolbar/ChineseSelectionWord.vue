<script setup lang="ts">

import {Icon} from "@iconify/vue";
import IconWrapper from "@/components/IconWrapper.vue";
import Tooltip from "@/components/Tooltip.vue";
import {useBaseStore} from "@/stores/base.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {ShortcutKey} from "@/types.ts";

const settingStore = useSettingStore()


function toggle() {
  // if (!show) emitter.emit(EventKey.closeOther)
  settingStore.currentMode = 'chineseSelectionWord'
}


</script>

<template>
  <div class="setting" @click.stop="null">
    <Tooltip
        :title="`根据中文选择单词(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.ToggleShowTranslate]})`"
    >
      <IconWrapper>
        <Icon v-if="settingStore.translate" icon="fluent:select-all-on-24-regular"
              @click="toggle"
        />
        <Icon v-else icon="fluent:select-all-on-24-regular"
              @click="toggle"
        />
      </IconWrapper>
    </Tooltip>
  </div>
</template>

<style scoped lang="scss">
@import "@/assets/css/style.scss";

.setting {
  position: relative;
}

.footer {
  margin-top: 10rem;
  display: flex;
  justify-content: flex-end;
  gap: 10rem;
}
</style>
