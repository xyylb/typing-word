<script setup lang="ts">
import { DefaultWord, ShortcutKey, Word } from "@/types.ts";
import VolumeIcon from "@/components/icon/VolumeIcon.vue";
import { $computed, $ref } from "vue/macros";
import { useBaseStore } from "@/stores/base.ts";
import { usePracticeStore } from "@/stores/practice.ts";
import { useSettingStore } from "@/stores/setting.ts";
import { usePlayBeep, usePlayCorrect, usePlayKeyboardAudio, usePlayWordAudio, useTTsPlayAudio } from "@/hooks/sound.ts";
import { emitter, EventKey } from "@/utils/eventBus.ts";
import { cloneDeep } from "lodash-es";
import { onUnmounted, watch, onMounted, ref,computed } from "vue";
import Tooltip from "@/components/Tooltip.vue";

interface IProps {
  word: Word,
}

const props = withDefaults(defineProps<IProps>(), {
  word: () => cloneDeep(DefaultWord),
})

const emit = defineEmits<{
  next: [],
  wrong: []
}>()

let input = $ref('')
let wrong = $ref('')
let showFullWord = $ref(false)
let waitNext = $ref(false)
//输入锁定，因为跳转到下一个单词有延时，如果重复在延时期间内重复输入，导致会跳转N次
let inputLock = false
let wordRepeatCount = 0
const settingStore = useSettingStore()

const playBeep = usePlayBeep()
const playCorrect = usePlayCorrect()
const playKeyboardAudio = usePlayKeyboardAudio()
const playWordAudio = usePlayWordAudio()
const ttsPlayAudio = useTTsPlayAudio()
const volumeIconRef: any = $ref()
const volumeTranslateIconRef: any = $ref()

let displayWord = $computed(() => {
  return props.word.name.slice(input.length + wrong.length)
})

watch(() => props.word, () => {
  wrong = input = ''
  wordRepeatCount = 0
  inputBoxValue.value = '' // 新增重置输入框值
  waitNext = inputLock = false
  if (settingStore.wordSound) {
    volumeIconRef?.play(400, true)
  }
})

onMounted(() => {
  emitter.on(EventKey.resetWord, () => {
    wrong = input = ''
    inputBoxValue.value = '' // 新增重置输入框值
  })

  emitter.on(EventKey.onTyping, onTyping)

  // 初始化屏幕宽度
  updateScreenWidth()
  // 监听窗口大小变化
  window.addEventListener('resize', updateScreenWidth)

})

onUnmounted(() => {
  emitter.off(EventKey.resetWord)
  emitter.off(EventKey.onTyping, onTyping)
  // 移除事件监听防止内存泄漏
  window.removeEventListener('resize', updateScreenWidth)
})

function repeat() {
  setTimeout(() => {
    wrong = input = ''
    wordRepeatCount++
    inputLock = false

    if (settingStore.wordSound) {
      volumeIconRef?.play()
    }
  }, settingStore.waitTimeForChangeWord)
}

async function onTyping(e: KeyboardEvent) {
  // 关键修改：如果是输入框内的事件，直接返回（不做任何处理）
  if (isMobile.value && e.target instanceof HTMLInputElement) {
    return // 移除return e，直接返回
  }

  // 小屏模式下非输入框的键盘事件才禁用
  if (isMobile.value) return

  if (waitNext) {
    if (e.code === 'Space') {
      emit('next')
      waitNext = false
    }
    return
  }
  if (inputLock) return
  inputLock = true
  let letter = e.key
  let isTypingRight = false
  let isWordRight = false
  if (settingStore.ignoreCase) {
    isTypingRight = letter.toLowerCase() === props.word.name[input.length].toLowerCase()
    isWordRight = (input + letter).toLowerCase() === props.word.name.toLowerCase()
  } else {
    isTypingRight = letter === props.word.name[input.length]
    isWordRight = (input + letter) === props.word.name
  }
  if (isTypingRight) {
    input += letter
    wrong = ''
    playKeyboardAudio()
  } else {
    emit('wrong')
    wrong = letter
    playKeyboardAudio()
    playBeep()
    volumeIconRef?.play()
    setTimeout(() => {
      wrong = ''
    }, 500)
  }

  if (isWordRight) {
    playCorrect()
    if (settingStore.repeatCount == 100) {
      if (settingStore.repeatCustomCount <= wordRepeatCount + 1) {
        if (settingStore.autoNext) {
          setTimeout(() => emit('next'), settingStore.waitTimeForChangeWord)
        } else {
          waitNext = true
          inputLock = false
        }
      } else {
        repeat()
      }
    } else {
      if (settingStore.repeatCount <= wordRepeatCount + 1) {
        if (settingStore.autoNext) {
          setTimeout(() => emit('next'), settingStore.waitTimeForChangeWord)
        } else {
          waitNext = true
          inputLock = false
        }
      } else {
        repeat()
      }
    }
  } else {
    inputLock = false
  }
}

function del() {
  playKeyboardAudio()

  if (wrong) {
    wrong = ''
  } else if (isMobile.value) {
    inputBoxValue.value = inputBoxValue.value.slice(0, -1) // 新增输入框删除逻辑
  } else {
    input = input.slice(0, -1)
  }
}

function showWord() {
  if (settingStore.allowWordTip) {
    showFullWord = true
    //如果默写情况下，看了提示也视为输入错误
    emit('wrong')
  }
}

function hideWord() {
  showFullWord = false
}


function play() {
  volumeIconRef?.play()
}

// 新增响应式宽度检测
const screenWidth = ref(window.innerWidth)
const isMobile = computed(() => screenWidth.value < 600 && settingStore.dictation)



// 新增输入框临时值
const inputBoxValue = ref('')

// 窗口宽度变化
const updateScreenWidth = () => {
  //console.log('屏幕宽度变化',window.innerWidth)
  screenWidth.value = window.innerWidth
}

function confirmInput() {
  if (inputLock) return
  inputLock = true

  const userInput = inputBoxValue.value.trim()
  const correctWord = props.word.name

  const isCorrect = settingStore.ignoreCase
      ? userInput.toLowerCase() === correctWord.toLowerCase()
      : userInput === correctWord

  if (isCorrect) {
    playCorrect()
    input = userInput
    wrong = ''
    showFullWord = false

    const repeatLimit = settingStore.repeatCount === 100
        ? settingStore.repeatCustomCount
        : settingStore.repeatCount

    if (wordRepeatCount + 1 >= repeatLimit) {
      if (settingStore.autoNext) {
        setTimeout(() => {
          emit('next')
          inputLock = false
        }, settingStore.waitTimeForChangeWord)
      } else {
        waitNext = true
        inputLock = false
      }
    } else {
      repeat()
    }
  } else {
    // ❌ 错误时显示错误内容
    emit('wrong')
    playKeyboardAudio()
    playBeep()
    volumeIconRef?.play()

    input = ''
    wrong = userInput          // ✅ 显示错的单词
    showFullWord = true
    waitNext = true
    inputLock = false          // ✅ 解锁，允许点击“下一题”
  }

  inputBoxValue.value = ''
}



defineExpose({del, showWord, hideWord, play})
</script>

<template>
  <div class="typing-word">
    <div class="translate"
         :style="{
      fontSize: settingStore.fontSize.wordTranslateFontSize +'rem',
      opacity: settingStore.translate ? 1 : 0
    }"
    >
      <div class="translate-item" v-for="(v,i) in word.trans">
        <span>{{ v }}</span>
        <!--        <div class="volumeIcon">-->
        <!--          <Tooltip-->
        <!--              v-if="i === word.trans.length - 1"-->
        <!--              :title="`发音(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.PlayTranslatePronunciation]})`"-->
        <!--          >-->
        <!--            <VolumeIcon-->
        <!--                ref="volumeTranslateIconRef"-->
        <!--                :simple="true"-->
        <!--                :cb="()=>ttsPlayAudio(word.trans.join(';'))"/>-->
        <!--          </Tooltip>-->
        <!--        </div>-->
      </div>
    </div>
    <div class="word-wrapper">
      <div class="word"
           :class="wrong && 'is-wrong'"
           :style="{fontSize: settingStore.fontSize.wordForeignFontSize +'rem'}"
      >
        <span class="input" v-if="input">{{ input }}</span>
        <span class="wrong" v-if="wrong">{{ wrong }}</span>
        <template v-if="settingStore.dictation">
          <!-- 大屏幕模式：下划线输入 -->
          <template v-if="!isMobile">
    <span class="letter" v-if="!showFullWord"
          @mouseenter="showWord">{{
        displayWord.split('').map(() => settingStore.dictationShowWordLength ? '_' : '&nbsp;').join('')
      }}</span>
            <span class="letter" v-else @mouseleave="hideWord">{{ displayWord }}</span>
          </template>
        </template>
        <span class="letter" v-else>{{ displayWord }}</span>
      </div>
      <Tooltip
          :title="`发音(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.PlayWordPronunciation]})`"
      >
        <VolumeIcon ref="volumeIconRef" :simple="true" :cb="() => playWordAudio(word.name)"/>
      </Tooltip>
    </div>
    <div class="phonetic" v-if="settingStore.wordSoundType === 'us' && word.usphone">[{{ word.usphone }}]</div>
    <div class="phonetic" v-if="settingStore.wordSoundType === 'uk' && word.ukphone">[{{ word.ukphone }}]</div>

    <!-- 小屏幕模式：输入框+确认按钮 -->
    <template v-if="isMobile">
      <div style="text-align: center; ">
        <span class="wrong" style="font-size: 30px" v-if="wrong">答案{{ props.word.name }}</span>
        <input
            v-if="!wrong"
            v-model="inputBoxValue"
            class="dictation-input"
            :placeholder="displayWord.split('').map(() => '_').join('')"
            @keydown.enter="confirmInput"
            style="width: 80%"
            :style="{fontSize: settingStore.fontSize.wordForeignFontSize +'rem'}"
            autofocus
        >
        <div style="margin: 20px">
          <button  v-if="!wrong" class="confirm-btn" @click="confirmInput" style="height: 50px;width: 100px">确认</button> <button v-if="wrong"  style="height: 50px;width: 100px" class="next-btn" @click="() => { emit('next'); waitNext = false }">下一题</button>
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped lang="scss">
@import "@/assets/css/variable";

.typing-word {
  width: 100%;
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  word-break: break-word;

  .phonetic, .translate {
    font-size: 20rem;
    margin-left: -30rem;
    transition: all .3s;
  }

  .phonetic {
    margin-top: 5rem;
    font-family: var(--word-font-family);
  }

  .translate {
    position: absolute;
    transform: translateY(-50%);
    margin-bottom: 90rem;
    color: var(--color-font-2);

    @media (max-width: 600px) {
      position: relative;
    }

    &:hover {
      .volumeIcon {
        opacity: 1;
      }
    }

    .translate-item {
      display: flex;
      align-items: center;
      gap: 10rem;
    }

    .volumeIcon {
      transition: opacity .3s;
      opacity: 0;
    }
  }

  .word-wrapper {
    display: flex;
    align-items: center;
    gap: 10rem;
    color: var(--color-font-1);

    @media (max-width: 600px) {
      gap: 60rem;
      flex-direction: column;
    }


    .word {
      font-size: 48rem;
      line-height: 1;
      font-family: var(--word-font-family);
      letter-spacing: 5rem;

      .input {
        color: rgb(22, 163, 74);
      }

      .wrong {
        color: rgba(red, 0.6);
      }

      &.is-wrong {
        animation: shake 0.82s cubic-bezier(0.36, 0.07, 0.19, 0.97) both;
      }
    }
  }
}
</style>
