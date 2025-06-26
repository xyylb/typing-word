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
  settingStore.currentMode = 'dictation'
}


</script>

<template>
  <div class="setting" @click.stop="null">
    <Tooltip
        :title="`开关释义显示(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.ToggleShowTranslate]})`"
    >
      <IconWrapper>
        <Icon v-if="settingStore.translate" icon="fad:logo-audacity"
              @click="toggle"
        />
        <Icon v-else icon="fad:logo-audacity"
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
