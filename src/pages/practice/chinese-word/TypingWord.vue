<script setup lang="ts">
import { onMounted, onUnmounted, watch } from "vue"
import { $computed, $ref } from "vue/macros"
import { useBaseStore } from "@/stores/base.ts"
import { DefaultDisplayStatistics, DictType, ShortcutKey, Sort, Word } from "../../../types.ts";
import { emitter, EventKey } from "@/utils/eventBus.ts"
import { cloneDeep, reverse, shuffle } from "lodash-es"
import { usePracticeStore } from "@/stores/practice.ts"
import { useSettingStore } from "@/stores/setting.ts";
import { useOnKeyboardEventListener, useWindowClick } from "@/hooks/event.ts";
import { Icon } from "@iconify/vue";
import Tooltip from "@/components/Tooltip.vue";
import Options from "@/pages/practice/Options.vue";
import Typing from "./Typing.vue";
import Panel from "@/pages/practice/Panel.vue";
import IconWrapper from "@/components/IconWrapper.vue";
import { useRuntimeStore } from "@/stores/runtime.ts";
import { syncMyDictList, useWordOptions } from "@/hooks/dict.ts";
import BaseIcon from "@/components/BaseIcon.vue";
import WordList from "@/components/list/WordList.vue";
import Empty from "@/components/Empty.vue";
import MiniDialog from "@/components/dialog/MiniDialog.vue";
import BaseButton from "@/components/BaseButton.vue";
import { reactive } from 'vue'

interface IProps {
  words: Word[],
  index: number,
}

const props = withDefaults(defineProps<IProps>(), {
  words: [],
  index: -1
})

const emit = defineEmits<{
  'update:words': [val: Word[]],
  sort: [val: Word[]]
}>()

const typingRef: any = $ref()
const store = useBaseStore()
const runtimeStore = useRuntimeStore()
const practiceStore = usePracticeStore()
const settingStore = useSettingStore()

const {
  isWordCollect,
  toggleWordCollect,
  isWordSimple,
  toggleWordSimple
} = useWordOptions()

let data = $ref({
  index: props.index,
  words: props.words,
  wrongWords: [],
})

let stat = cloneDeep(DefaultDisplayStatistics)
let showSortOption = $ref(false)
useWindowClick(() => showSortOption = false)

watch(() => props.words, () => {
  data.words = props.words
  data.index = props.index
  data.wrongWords = []

  practiceStore.wrongWords = []
  practiceStore.repeatNumber = 0
  practiceStore.startDate = Date.now()
  practiceStore.correctRate = -1
  practiceStore.inputWordNumber = 0
  practiceStore.wrongWordNumber = 0
  stat = cloneDeep(DefaultDisplayStatistics)

}, {immediate: true})

watch(data, () => {
  practiceStore.total = data.words.length
  practiceStore.index = data.index
})

const word = $computed(() => {
  return data.words[data.index] ?? {
    trans: [],
    name: '',
    usphone: '',
    ukphone: '',
  }
})

const prevWord: Word = $computed(() => {
  return data.words?.[data.index - 1] ?? undefined
})

const nextWord: Word = $computed(() => {
  return data.words?.[data.index + 1] ?? undefined
})

function next(isTyping: boolean = true) {
  if (data.index === data.words.length - 1) {
    //复制当前错词，因为第一遍错词是最多的，后续的练习都是从错词中练习
    if (stat.total === -1) {
      let now = Date.now()
      stat = {
        startDate: practiceStore.startDate,
        endDate: now,
        spend: now - practiceStore.startDate,
        total: props.words.length,
        correctRate: -1,
        inputWordNumber: practiceStore.inputWordNumber,
        wrongWordNumber: data.wrongWords.length,
        wrongWords: data.wrongWords,
      }
      stat.correctRate = 100 - Math.trunc(((stat.wrongWordNumber) / (stat.total)) * 100)
    }

    if (data.wrongWords.length) {
      console.log('当前背完了，但还有错词')
      data.words = cloneDeep(data.wrongWords)

      practiceStore.total = data.words.length
      practiceStore.index = data.index = 0
      practiceStore.inputWordNumber = 0
      practiceStore.wrongWordNumber = 0
      practiceStore.repeatNumber++
      data.wrongWords = []
    } else {
      console.log('这章节完了')
      isTyping && practiceStore.inputWordNumber++

      let now = Date.now()
      stat.endDate = now
      stat.spend = now - stat.startDate

      emitter.emit(EventKey.openStatModal, stat)
    }
  } else {
    data.index++
    isTyping && practiceStore.inputWordNumber++
    console.log('这个词完了')
    if (store.skipWordNames.includes(word.name.toLowerCase())) {
      next()
    }
  }
}

function wordWrong() {
  if (!store.wrong.originWords.find((v: Word) => v.name.toLowerCase() === word.name.toLowerCase())) {
    store.wrong.originWords.push(word)
  }
  if (!data.wrongWords.find((v: Word) => v.name.toLowerCase() === word.name.toLowerCase())) {
    data.wrongWords.push(word)
    practiceStore.wrongWordNumber++
  }
}

function onKeyUp(e: KeyboardEvent) {
  typingRef.hideWord()
}

async function onKeyDown(e: KeyboardEvent) {
  // console.log('e', e)
  switch (e.key) {
    case 'Backspace':
      typingRef.del()
      break
  }
}

useOnKeyboardEventListener(onKeyDown, onKeyUp)

//TODO 略过忽略的单词上
function prev() {
  if (data.index === 0) {
    ElMessage.warning('已经是第一个了~')
  } else {
    data.index--
  }
}

function skip(e: KeyboardEvent) {
  next(false)
  // e.preventDefault()
}

function show(e: KeyboardEvent) {
  typingRef.showWord()
}

function collect(e: KeyboardEvent) {
  toggleWordCollect(word)
}

function toggleWordSimpleWrapper() {
  if (!isWordSimple(word)) {
    toggleWordSimple(word)
    //延迟一下，不知道为什么不延迟会导致当前条目不自动定位到列表中间
    setTimeout(() => next(false))
  } else {
    toggleWordSimple(word)
  }
}

function play() {
  typingRef.play()
}

function sort(type: Sort) {
  if (type === Sort.reverse) {
    ElMessage.success('已翻转排序')
    emit('sort', reverse(cloneDeep(data.words)))
  }
  if (type === Sort.random) {
    ElMessage.success('已随机排序')
    emit('sort', shuffle(data.words))
  }
}

onMounted(() => {
  emitter.on(ShortcutKey.ShowWord, show)
  emitter.on(ShortcutKey.Previous, prev)
  emitter.on(ShortcutKey.Next, skip)
  emitter.on(ShortcutKey.ToggleCollect, collect)
  emitter.on(ShortcutKey.ToggleSimple, toggleWordSimpleWrapper)
  emitter.on(ShortcutKey.PlayWordPronunciation, play)
})

onUnmounted(() => {
  emitter.off(ShortcutKey.ShowWord, show)
  emitter.off(ShortcutKey.Previous, prev)
  emitter.off(ShortcutKey.Next, skip)
  emitter.off(ShortcutKey.ToggleCollect, collect)
  emitter.off(ShortcutKey.ToggleSimple, toggleWordSimpleWrapper)
  emitter.off(ShortcutKey.PlayWordPronunciation, play)
})

let showFrame:any = reactive({
  domain:null,
  isShow: null
})
const handleShowFrame=(domain:any)=>{
  showFrame.domain = domain
  showFrame.isShow = 'baidu'

  fetch("https://fanyi.baidu.com/ait/text/translate", {
    "headers": {
      "accept": "text/event-stream",
      "accept-language": "zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6",
      "acs-token": "1741579597284_1741610498194_JCqdgddd3wyu/e6cB//HOYKU9BiFeG7Mj+tyYgL+kYyOcHyfwYivVhTnwbgSI/14lMGR1hODaMlQ7dBa7BtMPjl+sab76pl/vLn6b0/TzlcJTpiujIfvG6HmdGI76OjHQ2U4A8aI4VrPkYMi0+VeXkod4Z7qowxPnf8rmZNh00/eLLd1G9dO/uF9O0GvPn2bnhpKNc4opk2J0XxSZK58EPKT/EBbR/BO+HLfHTynuESucMqamc9N+piQTccnzF/VgkwmIBbwN3y8/3u7tTHLoDfAf8gShUZZikA3DQ1R0LihRpkQfOkcQtoaRfiM15VXF6r2FQDh4FuM4vS3/KVIusblvV4eC7tfVLF2ZeNHKveUiBvYViHEJA3uNoH85yCSqq4VC7g+/yMHCSjzifqAcy14VYNqGwNSZet0YRYdbsGJ/R3winTljJ6Zv9Lvn5cBuYTSCebMR9Ccie4nNJ/iFtPIynYeeQyiryZet1hQDTNBZsDTpgsnnGfOZaE7XnMs",
      "content-type": "application/json",
      "sec-ch-ua": "\"Not(A:Brand\";v=\"99\", \"Microsoft Edge\";v=\"133\", \"Chromium\";v=\"133\"",
      "sec-ch-ua-mobile": "?0",
      "sec-ch-ua-platform": "\"Windows\"",
      "sec-fetch-dest": "empty",
      "sec-fetch-mode": "cors",
      "sec-fetch-site": "same-origin",
      "cookie": "BAIDUID=3A94214A6576184EDFAEBB126BDC3C91:FG=1; PSTM=1722385540; BIDUPSID=2CE524E5BFA836A3E283D45A8E0BF10E; MAWEBCUID=web_exdrmCEeGriThxQdMFIKSxTNdExxytkGeHWgKtuhqlbGtootNh; H_WISE_SIDS_BFESS=61027_62127_62167_62256_62272_62135_62325_62337_62329_62366_62421_62422; BDUSS=zlEazFOWHd0bERpeTlDalM2WXZQSlJ-RFFpMVFPb2pSOVlmS3ZwVjhwMEkxflZuRVFBQUFBJCQAAAAAAAAAAAEAAAC1uEQdz-DN~NPava26~jM2MTIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAhKzmcISs5nb; BDUSS_BFESS=zlEazFOWHd0bERpeTlDalM2WXZQSlJ-RFFpMVFPb2pSOVlmS3ZwVjhwMEkxflZuRVFBQUFBJCQAAAAAAAAAAAEAAAC1uEQdz-DN~NPava26~jM2MTIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAhKzmcISs5nb; H_PS_PSSID=61027_62135_62325_62337_62329_62366_62421_62422_62473_62483; H_WISE_SIDS=61027_62135_62325_62337_62329_62366_62421_62422_62473_62483; BAIDUID_BFESS=3A94214A6576184EDFAEBB126BDC3C91:FG=1; BA_HECTOR=aga42hag2lah2k0k20258lag88tfha1jstc3323; ZFY=OFoh:BJh004ELuanTamNrmXfPX3El:AJv3zG:Aw1MGyYt0:C; BDRCVFR[feWj1Vr5u3D]=I67x6TjHwwYf0; PSINO=2; delPer=0; BDSFRCVID=1q0OJexroGWIIGQJzZmUo9xhIgKpJRJTDYrEOwXPsp3LGJLVveC9EG0PtEhTCouhEiP_ogKKymOTHukF_2uxOjjg8UtVJeC6EG0Ptf8g0M5; BDSFRCVID_BFESS=1q0OJexroGWIIGQJzZmUo9xhIgKpJRJTDYrEOwXPsp3LGJLVveC9EG0PtEhTCouhEiP_ogKKymOTHukF_2uxOjjg8UtVJeC6EG0Ptf8g0M5; H_BDCLCKID_SF=JR4t_D0-tI83H48k-4QEbbQH-UnLq5cGLgOZ04n-ah3ROqTj0f60Wx_zhNrOaUcMW5RZbqcm3UTKsq76Wh35K5tTQP6rLJ5-0b54KKJxbp5sShOv5t7pMq8mhUJiBbcLBan7LRvIXKohJh7FM4tW3J0ZyxomtfQxtNRJ0DnjtpChbCKxjjDbD6OyeU5eetjK2CntsJOOaCvaqxJOy4oWK4413PrA0hoZMHLf-JAXMnnzqM78yU8-3M04K4o9-hvT-54e2p3FBUQ-eq8wQft20b03QbCDbx333C5TMb7jWhvBDq72yboOQlRX5q79atTMfNTJ-qcH0KQpsIJM5-DWbT8IjHCtJTLqJbujoKvt-5rDHJTg5DTjhPrMLfkOWMT-MTryKM3YWl3GO4cLjUooeJkJyfJibUTNWHnRh4oNB-3iV-OxDUvnyxAZ5xomqxQxtNRJKRnm0b7hjh5Y0hnobUPU2fc9LUvt22cdot5yBbc8eIna5hjkbfJBQttjQn3hfIkj2CKLJCthMC04D5-3-RJH-xQ0KnLXKKOLVhjsfp7ketn4hUJRyfF10t6EXjb4ab5OBpTjW-JkVlc2QhrKQf4WWb3ebTJr32Qr-tQH0KTpsIJM5Mos3Mte5f74alOdaKvia-TEBMb1fJvDBT5h2M4qMxtOLR3pWDTm_q5TtUJMeCnTDMFhe6QQjHDfJj8efKresJoq2RbhKROvhj4aKj8gyxoObtRxQ25AKMjFMfcVSJAw2-7zbUPQyR5ZLU3k-eT9LMnx--t58h3_Xhj-bfJ-QttjQn342NnmW-5S3RTm_n7TyURBhf47yhoW0q4Hb6b9BJcjfU5MSlcNLTjpQT8r5MDOK5OuJRQ2QJ8BtDKMMIoP; H_BDCLCKID_SF_BFESS=JR4t_D0-tI83H48k-4QEbbQH-UnLq5cGLgOZ04n-ah3ROqTj0f60Wx_zhNrOaUcMW5RZbqcm3UTKsq76Wh35K5tTQP6rLJ5-0b54KKJxbp5sShOv5t7pMq8mhUJiBbcLBan7LRvIXKohJh7FM4tW3J0ZyxomtfQxtNRJ0DnjtpChbCKxjjDbD6OyeU5eetjK2CntsJOOaCvaqxJOy4oWK4413PrA0hoZMHLf-JAXMnnzqM78yU8-3M04K4o9-hvT-54e2p3FBUQ-eq8wQft20b03QbCDbx333C5TMb7jWhvBDq72yboOQlRX5q79atTMfNTJ-qcH0KQpsIJM5-DWbT8IjHCtJTLqJbujoKvt-5rDHJTg5DTjhPrMLfkOWMT-MTryKM3YWl3GO4cLjUooeJkJyfJibUTNWHnRh4oNB-3iV-OxDUvnyxAZ5xomqxQxtNRJKRnm0b7hjh5Y0hnobUPU2fc9LUvt22cdot5yBbc8eIna5hjkbfJBQttjQn3hfIkj2CKLJCthMC04D5-3-RJH-xQ0KnLXKKOLVhjsfp7ketn4hUJRyfF10t6EXjb4ab5OBpTjW-JkVlc2QhrKQf4WWb3ebTJr32Qr-tQH0KTpsIJM5Mos3Mte5f74alOdaKvia-TEBMb1fJvDBT5h2M4qMxtOLR3pWDTm_q5TtUJMeCnTDMFhe6QQjHDfJj8efKresJoq2RbhKROvhj4aKj8gyxoObtRxQ25AKMjFMfcVSJAw2-7zbUPQyR5ZLU3k-eT9LMnx--t58h3_Xhj-bfJ-QttjQn342NnmW-5S3RTm_n7TyURBhf47yhoW0q4Hb6b9BJcjfU5MSlcNLTjpQT8r5MDOK5OuJRQ2QJ8BtDKMMIoP; BCLID=11682019886874875799; BCLID_BFESS=11682019886874875799; BDORZ=FFFB88E999055A3F8A630C64834BD6D0; MCITY=-353%3A; RT=\"z=1&dm=baidu.com&si=c4c5064f-eefb-4223-9020-be8851f33dfa&ss=m831wkta&sl=1&tt=20m&bcn=https%3A%2F%2Ffclog.baidu.com%2Flog%2Fweirwood%3Ftype%3Dperf\"; ab_sr=1.0.1_MTVjYjU4MWQwZDFkYWY4YzIyNWIxMmVkYmY2MWJmOTFkNWFlYzMxOGI1M2Q1MGQ1N2VkMmZjZGQ1MzQwMjIxZGU0ZDNkMmU0ODU0OTRkYTMyMTJiYTg1MmY1N2Q0NGE3NjgxNDVhMzNkNzRkNTljYmU5Mjg1YzJkNGM4ODVjZmJlNjFhZTgwODA3OTg0YjgzZDMwOTBjMDk4OTNjZjVjMQ==",
      "Referer": "https://fanyi.baidu.com/mtpe-individual/multimodal?query=standard&lang=en2zh",
      "Referrer-Policy": "strict-origin-when-cross-origin"
    },
    "body": "{\"query\":\"standard\",\"from\":\"en\",\"to\":\"zh\",\"reference\":\"\",\"corpusIds\":[],\"needPhonetic\":false,\"domain\":\"common\",\"milliTimestamp\":1741610498259}",
    "method": "POST"
  });
}


</script>

<template>
  <div class="practice-word">
<!--    <div style=" position: fixed;z-index: 9999;   top: 0;
    left: 0;
    right: 0;
    bottom: 0;
}">
      <iframe v-if="showFrame.isShow==='baidu'" src="https://fanyi.baidu.com/mtpe-individual/multimodal?query=toc&lang=en2zh" style="width: 100%;height: 100%;">

      </iframe>
    </div>-->

    <div class="near-word" v-if="settingStore.showNearWord">
      <div class="prev"
           @click="prev"
           v-if="prevWord">
        <Icon class="arrow" icon="bi:arrow-left" width="22"/>
        <Tooltip
            :title="`上一个(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.Previous]})`"
        >
          <div class="word">{{ prevWord.name }}</div>
        </Tooltip>
      </div>
      <div class="next"
           @click="next(false)"
           v-if="nextWord">
        <Tooltip
            :title="`下一个(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.Next]})`"
        >
          <div class="word" :class="settingStore.dictation && 'text-shadow'">{{ nextWord.name }}</div>
        </Tooltip>
        <Icon class="arrow" icon="bi:arrow-right" width="22"/>
      </div>
    </div>
    <Typing
        v-loading="!store.load"
        ref="typingRef"
        :word="word"
        @wrong="wordWrong"
        @next="next"
    />
    <div class="options-wrapper">
<!--      <Options
          :is-simple="isWordSimple(word)"
          @toggle-simple="toggleWordSimpleWrapper"
          :is-collect="isWordCollect(word)"
          @toggle-collect="toggleWordCollect(word)"
          @skip="next(false)"
      />-->
    </div>

    <Teleport to="body">
      <div class="word-panel-wrapper">
        <Panel>
          <template v-slot="{active}">
            <div class="panel-page-item"
                 v-loading="!store.load"
            >
              <div class="list-header">
                <div class="left">
                  <div class="title">
                    {{ store.chapterName }}
                  </div>
                  <BaseIcon title="切换词典"
                            @click="emitter.emit(EventKey.openDictModal,'list')"
                            icon="carbon:change-catalog"/>
                  <div style="position:relative;"
                       @click.stop="null">
                    <BaseIcon
                        title="改变顺序"
                        icon="icon-park-outline:sort-two"
                        @click="showSortOption = !showSortOption"
                    />
                    <MiniDialog
                        v-model="showSortOption"
                        style="width: 130rem;"
                    >
                      <div class="mini-row-title">
                        列表循环设置
                      </div>
                      <div class="mini-row">
                        <BaseButton size="small" @click="sort(Sort.reverse)">翻转</BaseButton>
                        <BaseButton size="small" @click="sort(Sort.random)">随机</BaseButton>
                      </div>
                    </MiniDialog>
                  </div>
                  <BaseIcon icon="bi:arrow-right"
                            @click="emitter.emit(EventKey.next)"
                            :title="`下一章(快捷键：${settingStore.shortcutKeyMap[ShortcutKey.NextChapter]})`"
                            v-if="store.currentDict.chapterIndex < store.currentDict.chapterWords.length - 1"/>
                </div>
                <div class="right">
                  {{ data.words.length }}个单词
                </div>
              </div>
              <WordList
                  v-if="data.words.length"
                  :is-active="active"
                  :static="false"
                  :show-word="!settingStore.dictation"
                  :show-translate="settingStore.translate"
                  :list="data.words"
                  :activeIndex="data.index"
                  @click="(val:any) => data.index = val.index"
              >
                <template v-slot:suffix="{item,index}">
                  <BaseIcon
                      v-if="!isWordCollect(item)"
                      class="collect"
                      @click="toggleWordCollect(item)"
                      title="收藏" icon="ph:star"/>
                  <BaseIcon
                      v-else
                      class="fill"
                      @click="toggleWordCollect(item)"
                      title="取消收藏" icon="ph:star-fill"/>
                  <BaseIcon
                      v-if="!isWordSimple(item)"
                      class="easy"
                      @click="toggleWordSimple(item)"
                      title="标记为简单词"
                      icon="material-symbols:check-circle-outline-rounded"/>
                  <BaseIcon
                      v-else
                      class="fill"
                      @click="toggleWordSimple(item)"
                      title="取消标记简单词"
                      icon="material-symbols:check-circle-rounded"/>
                </template>
              </WordList>
              <Empty v-else/>
            </div>
          </template>
        </Panel>
      </div>
    </Teleport>


  </div>
</template>

<style scoped lang="scss">
@import "@/assets/css/variable";


.practice-word {
  height: 100%;
  flex: 1;
  display: flex;
  //display: none;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  font-size: 14rem;
  color: gray;
  gap: 6rem;
  position: relative;
  width: var(--toolbar-width);
  @media (max-width: 600px) {
    width: 100%;
  }

  .near-word {
    position: absolute;
    top: 0;
    width: 100%;

    & > div {
      width: 45%;
      align-items: center;

      .arrow {
        min-width: 22rem;
        min-height: 22rem;
      }
    }

    .word {
      font-size: 24rem;
      margin-bottom: 4rem;
      font-family: var(--word-font-family);
    }

    .prev {
      cursor: pointer;
      display: flex;
      float: left;
      gap: 10rem;
    }

    .next {
      cursor: pointer;
      display: flex;
      justify-content: flex-end;
      gap: 10rem;
      float: right;
    }
  }

  .options-wrapper {
    position: absolute;
    //bottom: 0;
    margin-left: -30rem;
    margin-top: 120rem;
  }
}

.word-panel-wrapper {
  position: fixed;
  left: 0;
  top: 10rem;
  z-index: 1;
  margin-left: var(--panel-margin-left);
  height: calc(100% - 20rem);
  @media (max-width: 600px) {
    display: none;
  }
}

</style>
