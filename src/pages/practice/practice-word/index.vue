<script setup lang="ts">

import TypingWord from "@/pages/practice/practice-word/TypingWord.vue";
import {$ref} from "vue/macros";
import {cloneDeep} from "lodash-es";
import {useBaseStore} from "@/stores/base.ts";
import {onMounted, onUnmounted} from "vue";
import {useRuntimeStore} from "@/stores/runtime.ts";
import {ShortcutKey, Word} from "@/types.ts";
import {emitter, EventKey} from "@/utils/eventBus.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {syncMyDictList} from "@/hooks/dict.ts";

const store = useBaseStore()
const runtimeStore = useRuntimeStore()
const settingStore = useSettingStore()

let wordData = $ref({
  words: [],
  index: -1
})

function getCurrentPractice() {
  if (store.chapter.length) {
    wordData.words = store.chapter
    wordData.index = 0

    store.chapter.map((w: Word) => {
      if (!w.trans.length) {
        let res = runtimeStore.translateWordList.find(a => a.name === w.name)
        if (res) w = Object.assign(w, res)
      }
    })

    wordData.words = cloneDeep(store.chapter)
    emitter.emit(EventKey.resetWord)
  }
}

function sort(list: Word[]) {
  store.currentDict.chapterWords[store.currentDict.chapterIndex] = wordData.words = list
  wordData.index = 0
  syncMyDictList(store.currentDict)
}

function next() {
  if (store.currentDict.chapterIndex >= store.currentDict.chapterWords.length - 1) {
    store.currentDict.chapterIndex = 0
  } else store.currentDict.chapterIndex++

  getCurrentPractice()
}

onMounted(() => {
  emitter.on(EventKey.changeDict, getCurrentPractice)
  emitter.on(EventKey.next, next)
  emitter.on(ShortcutKey.NextChapter, next)

  updateScreenWidth();
  // 移除事件监听防止内存泄漏
  window.addEventListener('resize', updateScreenWidth)
  getCurrentPractice()
})

// 窗口宽度变化
const updateScreenWidth = () => {
  const isMobile  = window.innerWidth<600
}


onUnmounted(() => {
  emitter.off(EventKey.changeDict, getCurrentPractice)
  emitter.off(EventKey.next, next)
  emitter.off(ShortcutKey.NextChapter, next)

  // 移除事件监听防止内存泄漏
  window.removeEventListener('resize', updateScreenWidth)
})

defineExpose({getCurrentPractice})

</script>

<template>
  <div class="practice">
    <TypingWord
        @sort="sort"
        v-model:words="wordData.words"
        :index="wordData.index"/>
  </div>
</template>

<style scoped lang="scss">
.practice {
  //height: 100%;
  flex: 1;
  @media (max-width: 600px) {
    width: 100%;
  }
}
</style>
