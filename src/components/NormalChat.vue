<template>
  <v-main>
    <SideBar
      :disabled="generating"
      @load="onLoad"
      type="normal"
      ref="sidebar"
    ></SideBar>
    <v-container class="fill-height">
      <v-responsive class="text-center fill-height">
        <slot name="mode-selector" />
        <v-select
          label="モデル"
          v-model="model"
          :items="models"
          :hint="model"
          :persistent-hint="true"
          item-title="title"
          item-value="value"
          :disabled="models.length <= 1"
          :loading="models.length <= 1"
          :readonly="generating"
          @update:model-value="onModelChange()"
        ></v-select>
        <v-textarea
          v-if="current.messages.length === 0"
          label="命令 (System)"
          v-model="systemPrompt"
          :disabled="generating"
        ></v-textarea>
        <v-btn
          v-if="current.messages.length === 0"
          color="pink"
          prepend-icon="mdi-heart"
          @click="systemPrompt += 'あなたは「ずんだもん」というキャラクターのように話してください。ずんだもんは幼い女の子で、無邪気な性格をしており、口調は強気であり、「〜のだ」「〜なのだ」を語尾につけます。'"
          :disabled="generating"
        >ずんだもん</v-btn>
        <chat-entry
          v-for="(item, index) in current.messages"
          :key="index"
          :content="item"
          @remove="() => current.messages.splice(index, 1)"
        />
        <v-textarea
          label="入力 (User)"
          v-model="userPrompt"
          :auto-grow="true"
          :disabled="generating"
        ></v-textarea>
        <v-file-input
          v-if="model.includes('vision')"
          v-model="images"
          :multiple="true"
          accept="image/png, image/jpeg, image/gif, image/webp"
          label="画像"
          :disabled="generating"
        ></v-file-input>
        <v-btn
          class="float-left"
          color="blue"
          prepend-icon="mdi-delete"
          :disabled="generating || !current.id || current.messages.length === 0"
          @click="deleteHistory(current.id); resetCurrent(); sidebar.update()"
        >削除</v-btn>
        <v-btn
          class="float-left"
          color="blue"
          prepend-icon="mdi-floppy"
          :disabled="generating || !current.id"
          @click="saveHistory(current)"
        >保存</v-btn>
        <v-btn
          style="margin: 10px"
          class="float-right"
          color="blue"
          append-icon="mdi-send"
          :disabled="generating || userPrompt.length === 0"
          @click="generate()"
        >生成</v-btn>
        <v-checkbox
          :label="current.messages.length === 0 ? '自動保存（オン）' : '自動保存'"
          class="float-right"
          v-model="autoSave"
          :disabled="current.messages.length === 0"
        ></v-checkbox>
      </v-responsive>
    </v-container>
  </v-main>
</template>

<script lang="ts" setup>
import SideBar from "@/components/ChatHistorySideBar.vue";
import {ref} from "vue";
import {apiUrl, fileToBase64DataUrl, SUMMARIZE_PROMPT} from "@/util/util";
import {deleteHistory, HistoryEntry, JsonContent, saveHistory} from "@/util/history";
import ChatEntry from "@/components/ChatEntry.vue";

const decoder = new TextDecoder()
const models = ref([{ title: 'Loading...', value: 'dummy' }])
const model = ref('')
const systemPrompt = ref('')
const userPrompt = ref('')
const images = ref<File[]>([])
const current = ref<HistoryEntry>({ id: '', title: '', messages: [] })
const generating = ref(false)
const autoSave = ref(true)
const sidebar = ref(null)

const onModelChange = () => {
  localStorage.setItem('model', model.value)
}

const onLoad = (entry: HistoryEntry) => {
  current.value = entry
}

const resetCurrent = () => {
  current.value.id = ''
  current.value.title = ''
  current.value.messages = []
}

const generate = async () => {
  generating.value = true
  try {
    if (current.value.messages.length === 0) {
      current.value.id = crypto.randomUUID()
      current.value.title = ''
      if (systemPrompt.value.length > 0) {
        current.value.messages.push({ role: 'system', content: systemPrompt.value })
      }
    }
    const userContent: Array<JsonContent> = [{ type: 'text', text: userPrompt.value }]
    for (const file of images.value) {
      userContent.push({ type: 'image_url', image_url: await fileToBase64DataUrl(file) })
    }
    current.value.messages.push({
      role: 'user',
      content: userContent,
    })
    await fetch(apiUrl('generate'), {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: model.value,
        content: current.value.messages,
      }),
    }).then(async (res) => {
      let currentContent = ''
      current.value.messages.push({role: 'assistant', content: currentContent})
      for await (const chunk of res.body) {
        const string = decoder.decode(chunk)
        currentContent += string
        current.value.messages[current.value.messages.length - 1].content = currentContent
      }
      const userPromptBackup = userPrompt.value
      userPrompt.value = ''
      images.value = []
      if (!current.value.title) {
        await fetch(apiUrl('generate'), {
          method: 'POST',
          body: JSON.stringify({
            model: 'gpt-4',
            content: [
              { role: 'system', content: SUMMARIZE_PROMPT },
              { role: 'user', content: userPromptBackup },
            ]
          })
        }).then(res => res.text()).then(summary => {
          if ((summary.startsWith('"') && summary.endsWith('"')) || (summary.startsWith('「') && summary.endsWith('」'))) {
            summary = summary.substring(1, summary.length - 1)
          }
          current.value.title = summary
          saveHistory(current.value)
          sidebar.value.update()
        })
      } else if (autoSave.value) {
        saveHistory(current.value)
      }
    }).catch(e => console.error(e.stack || e))
  } finally {
    generating.value = false
  }
}

fetch(apiUrl('models'))
  .then(res => res.json())
  .then(value => Object.keys(value).map(k => ({ value: k, title: value[k] })))
  .then(array => models.value = array)
  .then(() => model.value = models.value[0].value)
  .then(() => {
    const modelOnLocalStorage = localStorage.getItem('model')
    if (modelOnLocalStorage && models.value.find(e => e.value === modelOnLocalStorage)) {
      model.value = modelOnLocalStorage
    }
  })
</script>
