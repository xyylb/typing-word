<script setup lang="ts">

import Toolbar from "@/components/toolbar/index.vue"
import {onMounted, onUnmounted, watch} from "vue";
import {usePracticeStore} from "@/stores/practice.ts";
import Footer from "@/pages/practice/Footer.vue";
import {useBaseStore} from "@/stores/base.ts";
import {$ref} from "vue/macros";
import Statistics from "@/pages/practice/Statistics.vue";
import {emitter, EventKey} from "@/utils/eventBus.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {useRuntimeStore} from "@/stores/runtime.ts";
import {MessageBox} from "@/utils/MessageBox.tsx";
import PracticeArticle from "@/pages/practice/practice-article/index.vue";
import PracticeWord from "@/pages/practice/practice-word/index.vue";
import {ShortcutKey} from "@/types.ts";
import DictModal from "@/components/dialog/DictDiglog.vue";
import {useStartKeyboardEventListener} from "@/hooks/event.ts";
import useTheme from "@/hooks/theme.ts";
import RightTopBar from "@/components/RightTopBar.vue";
import Logo from "@/components/Logo.vue";
import ChineseWord from './chinese-word/index.vue'
import WordChinese from './word-chinese/index.vue'

const practiceStore = usePracticeStore()
const store = useBaseStore()
const settingStore = useSettingStore()
const runtimeStore = useRuntimeStore()
const {toggleTheme} = useTheme()
const practiceRef: any = $ref()

watch(practiceStore, () => {
  if (practiceStore.inputWordNumber < 1) {
    return practiceStore.correctRate = -1
  }
  if (practiceStore.wrongWordNumber > practiceStore.inputWordNumber) {
    return practiceStore.correctRate = 0
  }
  practiceStore.correctRate = 100 - Math.trunc(((practiceStore.wrongWordNumber) / (practiceStore.inputWordNumber)) * 100)
})


function test() {
  MessageBox.confirm(
      '2您选择了“本地翻译”，但译文内容却为空白，是否修改为“不需要翻译”并保存?',
      '1提示',
      () => {
        console.log('ok')
      },
      () => {
        console.log('cencal')
      })
}

function write() {
  // console.log('write')
  settingStore.dictation = true
  repeat()
}

//TODO 需要判断是否已忽略
function repeat() {
  // console.log('repeat')
  emitter.emit(EventKey.resetWord)
  practiceRef.getCurrentPractice()
}

function prev() {
  // console.log('next')
  if (store.currentDict.chapterIndex === 0) {
    ElMessage.warning('已经在第一章了~')
  } else {
    store.currentDict.chapterIndex--
    repeat()
  }
}

function toggleShowTranslate() {
  settingStore.translate = !settingStore.translate
}

function toggleDictation() {
  settingStore.dictation = !settingStore.dictation
}

function openSetting() {
  runtimeStore.showSettingModal = true
}

function openDictDetail() {
  emitter.emit(EventKey.openDictModal, 'detail')
}

function toggleConciseMode() {
  settingStore.showToolbar = !settingStore.showToolbar
  settingStore.showPanel = settingStore.showToolbar
}

function togglePanel() {
  settingStore.showPanel = !settingStore.showPanel
}

function jumpSpecifiedChapter(val: number) {
  store.currentDict.chapterIndex = val
  repeat()
}

onMounted(() => {
  emitter.on(EventKey.write, write)
  emitter.on(EventKey.repeat, repeat)
  emitter.on(EventKey.jumpSpecifiedChapter, jumpSpecifiedChapter)

  emitter.on(ShortcutKey.PreviousChapter, prev)
  emitter.on(ShortcutKey.RepeatChapter, repeat)
  emitter.on(ShortcutKey.DictationChapter, write)
  emitter.on(ShortcutKey.ToggleShowTranslate, toggleShowTranslate)
  emitter.on(ShortcutKey.ToggleDictation, toggleDictation)
  emitter.on(ShortcutKey.OpenSetting, openSetting)
  emitter.on(ShortcutKey.OpenDictDetail, openDictDetail)
  emitter.on(ShortcutKey.ToggleTheme, toggleTheme)
  emitter.on(ShortcutKey.ToggleConciseMode, toggleConciseMode)
  emitter.on(ShortcutKey.TogglePanel, togglePanel)
})

onUnmounted(() => {
  emitter.off(EventKey.write, write)
  emitter.off(EventKey.repeat, repeat)
  emitter.off(EventKey.jumpSpecifiedChapter, jumpSpecifiedChapter)

  emitter.off(ShortcutKey.PreviousChapter, prev)
  emitter.off(ShortcutKey.RepeatChapter, repeat)
  emitter.off(ShortcutKey.DictationChapter, write)
  emitter.off(ShortcutKey.ToggleShowTranslate, toggleShowTranslate)
  emitter.off(ShortcutKey.ToggleDictation, toggleDictation)
  emitter.off(ShortcutKey.OpenSetting, openSetting)
  emitter.off(ShortcutKey.OpenDictDetail, openDictDetail)
  emitter.off(ShortcutKey.ToggleTheme, toggleTheme)
  emitter.off(ShortcutKey.ToggleConciseMode, toggleConciseMode)
  emitter.off(ShortcutKey.TogglePanel, togglePanel)
})

useStartKeyboardEventListener()

</script>
<template>
  <div class="practice-wrapper">
    <Logo/>
    <Toolbar/>
    <!--    <BaseButton @click="test">test</BaseButton>-->
    <template v-if="settingStore.currentMode=='dictation'">
      <PracticeArticle ref="practiceRef" v-if="store.isArticle"/>
      <PracticeWord ref="practiceRef" v-else/>
    </template>
    <!--这里-->
      <ChineseWord ref="practiceRef" v-if="settingStore.currentMode=='chineseSelectionWord'"></ChineseWord>
    <WordChinese ref="practiceRef" v-if="settingStore.currentMode=='wordSelectionChinese'"></WordChinese>
    <Footer/>
  </div>
  <RightTopBar/>
  <DictModal/>
  <Statistics/>
</template>

<style scoped lang="scss">
.practice-wrapper {
  font-size: 13rem;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
  //padding-right: var(--practice-wrapper-padding-right);
  transform: translateX(var(--practice-wrapper-translateX));

  @media (max-width: 600px) {
    transform: none;
  }

}

</style>
