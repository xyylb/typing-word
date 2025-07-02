<script setup lang="ts">
import { DefaultWord, ShortcutKey, Word, } from "@/types.ts";
import VolumeIcon from "@/components/icon/VolumeIcon.vue";
import { $computed, $ref } from "vue/macros";
import { useSettingStore } from "@/stores/setting.ts";
import { usePlayBeep, usePlayCorrect, usePlayKeyboardAudio, usePlayWordAudio, useTTsPlayAudio } from "@/hooks/sound.ts";
import { emitter, EventKey } from "@/utils/eventBus.ts";
import {cloneDeep, shuffle} from "lodash-es";
import { onUnmounted, watch, onMounted,ref } from "vue";
import Tooltip from "@/components/Tooltip.vue";
import {useBaseStore} from "@/stores/base.ts";

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
  resultMessage.value = wrong = input =  ''
  wordRepeatCount = 0
  waitNext = inputLock = false
  if (settingStore.wordSound) {
    volumeIconRef?.play(400, true)
  }
  generateOptions()
})

onMounted(() => {
  emitter.on(EventKey.resetWord, () => {
    wrong = input = ''
  })

  emitter.on(EventKey.onTyping, onTyping)
})

onUnmounted(() => {
  emitter.off(EventKey.resetWord)
  emitter.off(EventKey.onTyping, onTyping)
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

async function onTyping(inputCorrect: boolean) {
  if (waitNext) {
    return; // 等待用户手动触发下一个单词时，不处理输入
  }

  if (inputLock) {
    return; // 输入被锁定时，不处理输入
  }

  inputLock = true;

  if (inputCorrect) {
    // 处理正确输入
    emit('next');
    playKeyboardAudio();
    wrong = '';

    // 检查单词是否完成
    const isWordRight = (input.length + 1) === props.word.name.length;

    if (isWordRight) {
      playCorrect();

      // 判断是否达到重复次数上限
      const repeatLimit = settingStore.repeatCount === 100
          ? settingStore.repeatCustomCount
          : settingStore.repeatCount;

      if (wordRepeatCount + 1 >= repeatLimit) {
        // 达到重复次数上限，处理下一个单词
        if (settingStore.autoNext) {
          setTimeout(() => emit('next'), settingStore.waitTimeForChangeWord);
        } else {
          waitNext = true;
        }
      } else {
        // 未达到重复次数上限，重复当前单词
        repeat();
      }
    } else {
      // 单词未完成，继续接受输入
      inputLock = false;
    }
  } else {
    // 处理错误输入
    emit('wrong');
    playKeyboardAudio();
    playBeep();
    volumeIconRef?.play();

    wrong = 'X'; // 显示错误标记
    setTimeout(() => {
      wrong = ''; // 500ms后清除错误标记
    }, 500);

    inputLock = false; // 解锁输入，允许继续尝试
  }
}

function del() {
  playKeyboardAudio()

  if (wrong) {
    wrong = ''
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




const isAnswerChecked = ref(false);
const resultMessage = ref('');
const options = ref<Word[]>([]);

const store = useBaseStore();
// 生成备选项
function generateOptions() {
  const correctOption = props.word;
  const allWords = store.currentDict.chapterWords.flat();

  const incorrectOptions: Word[] = [];
  const seenIndexes = new Set<number>();

  while (incorrectOptions.length < 3 && seenIndexes.size < allWords.length) {
    const randomIndex = Math.floor(Math.random() * allWords.length);

    if (seenIndexes.has(randomIndex)) continue;
    seenIndexes.add(randomIndex);

    const randomWord = allWords[randomIndex];

    if (
        randomWord.name !== correctOption.name &&
        randomWord.trans // 确保有翻译，防止空值
    ) {
      // 避免重复添加同名词
      if (!incorrectOptions.some(opt => opt.name === randomWord.name)) {
        incorrectOptions.push(randomWord);
      }
    }
  }

  const allOptions = [...incorrectOptions, correctOption];
  options.value = shuffle(allOptions);
}

// 检查答案
function checkAnswer(option: Word) {
  isAnswerChecked.value = true;
  if (option.name === props.word.name) {
    /*resultMessage.value = '回答正确！';*/
    setTimeout(() => {
      //下一个
      console.log('下一个执行了吗？')
      onTyping(true)
    }, 100);
    isAnswerChecked.value = false;
  } else {
    resultMessage.value = `回答错误，正确答案是：${props.word.trans.join('；')}`;
    //下一题
    onTyping(false)
  }
}
function nextWord(){
  isAnswerChecked.value = false;
  onTyping(true)
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
      <div class="translate-item">
        <span>{{ word.name }}</span> <VolumeIcon ref="volumeIconRef" :simple="true" :cb="() => playWordAudio(word.name)"/>
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
      <div class="phonetic2" v-if="settingStore.wordSoundType === 'us' && word.usphone">[{{ word.usphone }}]</div>
      <div class="phonetic2" v-if="settingStore.wordSoundType === 'uk' && word.ukphone">[{{ word.ukphone }}]</div>
    </div>
    <div class="body-container">
      <ul v-if="!isAnswerChecked">
        <li v-for="(option, index) in options" :key="index" style="font-size: 18px; ">
          <div style="display: flex;">
            <div style="flex: 1;border-bottom: 1px solid #c4c3c3; padding: 10px 0; margin: 5px 20px;cursor: pointer"  @click="checkAnswer(option)"> {{ option.trans.join('；') }}</div></div>
        </li>
      </ul>
    </div>
    <p v-if="resultMessage" style="font-size: 18px;">{{ resultMessage }}</p>
    <button v-if="resultMessage" @click="nextWord" style="font-size: 18px;">下一个</button>
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

  .phonetic2 {
    font-size: 15rem;
    font-family: var(--word-font-family);
  }

  .translate {

    transform: translateY(-50%);
    margin-bottom: 90rem;
    color: var(--color-font-2);

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
  .body-container{
    width: 500px;
    @media (max-width: 600px) {
      width: 100%;
    }
  }
}
</style>
