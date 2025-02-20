<template>
  <v-main>
    <SideBar
      :disabled="generating"
      @load="onLoad"
      type="thread"
      ref="sidebar"
    ></SideBar>
    <v-container class="fill-height">
      <v-responsive class="text-center fill-height">
        <slot name="mode-selector" />
        <v-select
          v-model="model"
          :items="filterModels()"
          :hint="model"
          :persistent-hint="true"
          item-title="title"
          item-value="value"
          :disabled="filterModels().length < 1"
          :loading="filterModels().length < 1"
          :readonly="generating"
          @update:model-value="onModelChange()"
        ></v-select>
        <v-textarea
          label="命令 (System)"
          v-model="systemPrompt"
          :disabled="generating"
        ></v-textarea>
        <v-btn
          color="pink"
          prepend-icon="mdi-heart"
          @click="systemPrompt += 'あなたは「ずんだもん」というキャラクターのように話してください。ずんだもんは幼い女の子で、無邪気な性格をしており、口調は強気であり、「〜のだ」「〜なのだ」を語尾につけます。'"
          :disabled="generating"
        >ずんだもん</v-btn>
        <v-text-field
          label="アジ鯖APIキー"
          v-model="azisabaApiKey"
          type="password"
          :readonly="generating"
          @update:model-value="onAzisabaApiKeyUpdate()"
        ></v-text-field>
        <thread-chat-entry
          v-for="(item, index) in current.messages"
          :key="index"
          :content="item"
          :allow-remove="false"
          @remove="() => current.messages.splice(index, 1)"
        />
        <v-textarea
          label="入力 (User)"
          v-model="userPrompt"
          :auto-grow="true"
          :disabled="generating"
        ></v-textarea>
        <v-file-input
          v-model="files"
          :multiple="true"
          label="ファイル"
          :disabled="generating"
        ></v-file-input>
        <v-select
          label="Functions"
          :items="Object.keys(functions)"
          :multiple="true"
          v-model="selectedFunctions"
          :readonly="generating"
        ></v-select>
        <v-btn
          class="float-left"
          color="blue"
          prepend-icon="mdi-delete"
          :disabled="generating || !current.id || current.messages.length === 0"
          @click="deleteThread()"
        >削除</v-btn>
        <v-btn
          class="float-left"
          color="blue"
          prepend-icon="mdi-floppy"
          :disabled="generating || !current.id"
          @click="saveHistory(current)"
        >保存</v-btn>
        <v-btn
          class="float-left"
          color="blue"
          prepend-icon="mdi-refresh"
          :disabled="generating || !current.id"
          @click="updateCurrentThread()"
        >更新</v-btn>
        <v-btn
          style="margin: 10px"
          class="float-right"
          color="blue"
          append-icon="mdi-send"
          :disabled="generating || (userPrompt.length === 0 && current.messages[current.messages.length - 1]?.role !== 'user') || !isSelectedValidModel()"
          @click="generate()"
        >生成</v-btn>
        <v-checkbox
          :label="current.messages.length === 0 ? '自動保存（オン）' : '自動保存'"
          class="float-right"
          v-model="autoSave"
          :disabled="current.messages.length === 0"
        ></v-checkbox>
        <v-checkbox
          label="Code Interpreter"
          class="float-right"
          v-model="codeInterpreter"
          :disabled="generating"
        ></v-checkbox>
        <v-checkbox
          label="Retrieval"
          class="float-right"
          v-model="retrieval"
          :disabled="generating"
        ></v-checkbox>
      </v-responsive>
    </v-container>
  </v-main>
</template>

<script lang="ts" setup>
import SideBar from "@/components/ChatHistorySideBar.vue";
import {ref} from "vue";
import {apiUrl, convertPdfToText, SUMMARIZE_PROMPT} from "@/util/util";
import {deleteHistory, ThreadHistoryEntry, saveHistory, Message, Run} from "@/util/thread_history";
import ThreadChatEntry from "@/components/ThreadChatEntry.vue";
import {NodeHtmlMarkdown} from "node-html-markdown";

const models = ref(new Array<{ title: string, value: string }>())
const model = ref('')
const azisabaApiKey = ref(localStorage.getItem('azisaba_apikey') || '')
const systemPrompt = ref('')
const userPrompt = ref('')
const files = ref<File[]>([])
const current = ref<ThreadHistoryEntry>({ id: '', title: '', messages: [] })
const generating = ref(false)
const autoSave = ref(true)
const codeInterpreter = ref(true)
const retrieval = ref(true)
const sidebar = ref(null)
const functions = {
  fetch: {
    action: async ({url}: { url: string }): Promise<string> => {
      const urlObject = new URL(url)
      const res = await fetch(apiUrl('request'), { method: 'POST', body: url })
      if (urlObject.pathname.endsWith(".pdf")) {
        return await convertPdfToText(await res.arrayBuffer())
      }
      const text = await res.text()
      if (text.startsWith('<!DOCTYPE html>') || text.startsWith('<!DOCTYPE HTML>')) {
        return NodeHtmlMarkdown.translate(text)
      }
      return text
    },
    description: 'Fetch a single web page; returns raw text as result',
    parameters: {
      type: 'object',
      properties: {
        url: { type: 'string', description: 'The URL to fetch' },
      },
      required: ['url'],
    },
  },
  search: {
    action: async ({query}: { query: string }) => {
      const json = await fetch(apiUrl('search'), { method: 'POST', body: query }).then(res => res.json())
      if (json.kind) {
        return (json.items as any[]).map(item => `- [${item.title}](${item.link}): ${item.snippet}`).join('\n')
      } else {
        return 'Search failed (API returned invalid response)'
      }
    },
    description: 'Google Search',
    parameters: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search query' },
      },
      required: ['query'],
    },
  },
  player_info: {
    action: async ({name}: { name: string }) => {
      if (!azisabaApiKey.value) {
        return 'Azisaba API Key is not set in UI'
      }
      return await fetch(`https://api-ktor.azisaba.net/players/by-name/${encodeURI(name)}`, {
        method: 'GET',
        headers: {
          Authorization: 'Bearer ' + azisabaApiKey.value,
        },
      }).then(res => res.text())
    },
    description: 'アジ鯖のプレイヤー情報を取得',
    parameters: {
      type: 'object',
      properties: {
        name: { type: 'string', description: 'Player name' },
      },
      required: ['name'],
    },
  },
  punish_search_player: {
    action: async ({query}: { query: string }) => {
      if (!azisabaApiKey.value) {
        return 'Azisaba API Key is not set in UI'
      }
      return await fetch(`https://api-ktor.azisaba.net/players/by-name/${encodeURI(query)}/punishments`, {
        method: 'GET',
        headers: {
          Authorization: 'Bearer ' + azisabaApiKey.value,
        },
      }).then(res => res.text())
    },
    description: 'アジ鯖の処罰データベースをプレイヤー名から検索',
    parameters: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search query (player name)' },
      },
      required: ['query'],
    },
  },
  punish_search_reason: {
    action: async (args: { [key: string]: any }) => {
      if (!azisabaApiKey.value) {
        return 'Azisaba API Key is not set in UI'
      }
      const url = new URL('https://api-ktor.azisaba.net/punishments/search')
      for (const key of Object.keys(args)) {
        url.searchParams.set(key, String(args[key]))
      }
      return await fetch(url, {
        method: 'GET',
        headers: {
          Authorization: 'Bearer ' + azisabaApiKey.value,
        },
      }).then(res => res.json()).then(res => {
        return res.map((punishment: any) => {
          return {
            ...punishment,
            start_date: new Date(punishment.start).toLocaleString(),
            end_date: punishment.end <= 0 ? 'Permanent' : new Date(punishment.end).toLocaleString()
          }
        })
      }).then(res => JSON.stringify(res))
    },
    description: 'アジ鯖の処罰データベースを処罰理由から検索',
    parameters: {
      type: 'object',
      properties: {
        q: { type: 'string', description: 'Search query (Search by punishment reason)' },
        type: { type: 'string', enum: ['BAN', 'MUTE', 'KICK', 'WARNING', 'CAUTION', 'NOTE'], description: 'Filter by punishment type' },
        revertedOnly: { type: 'boolean', description: 'Filter by reverted punishments only, if true.' },
        server: { type:  'string', description: 'Filter by server name (in english)' },
      },
      required: ['q'],
    },
  }
}
const selectedFunctions = ref(['search', 'fetch'])

const onAzisabaApiKeyUpdate = () => {
  localStorage.setItem('azisaba_apikey', azisabaApiKey.value)
}

const onModelChange = () => {
  localStorage.setItem('assistant-model', model.value)
}

const onLoad = (entry: ThreadHistoryEntry) => {
  current.value = entry
}

const resetCurrent = () => {
  current.value.id = ''
  current.value.title = ''
  current.value.messages = []
}

const deleteThread = () => {
  fetch(apiUrl(`threads/${current.value.id}`), {method: 'DELETE'})
  deleteHistory(current.value.id)
  resetCurrent()
  sidebar.value.update()
}

const updateCurrentThread = async () => {
  return await fetch(apiUrl(`threads/${current.value.id}/messages`))
    .then(res => res.json())
    .then(async res => {
      if (res.data) {
        await updateThreadWithMessages(res.data as Array<Message>)
      }
    })
}

const getRunSteps = (runId: string) =>
  fetch(apiUrl(`threads/${current.value.id}/runs/${runId}/steps`))
    .then(res => res.json())
    .then(res => {
      if (res.data) {
        return res.data
      } else {
        throw new Error('Invalid response: ' + JSON.stringify(res))
      }
    })

const getRun = (runId: string) =>
  fetch(apiUrl(`threads/${current.value.id}/runs/${runId}`))
    .then(res => res.json())
    .then((res: Run) => {
      if (res.status) {
        return res
      } else {
        throw new Error('Invalid response: ' + JSON.stringify(res))
      }
    })

const updateThreadWithMessages = async (messages: Array<Message>) => {
  current.value.messages = current.value.messages.filter((e, i, a) => a.indexOf(a.find(v => e.id === v.id)) === i)
  for (const message of messages.sort((a, b) => a.created_at - b.created_at)) {
    const existingMessage = current.value.messages.find(e => e.id === message.id)
    if (existingMessage) {
      existingMessage.content = message.content
      if (existingMessage.file_ids && existingMessage.file_ids.length > 0 && !existingMessage.files) {
        existingMessage.files =
          (await Promise.all(existingMessage.file_ids.map(id => fetch(apiUrl('files/' + id)).then(res => res.json()))))
            .filter(e => e.filename)
      }
      if (existingMessage.run_id) {
        if (!existingMessage.run) {
          existingMessage.run = await getRun(existingMessage.run_id)
        } else if (!existingMessage.run.steps) {
          existingMessage.run.steps = await getRunSteps(existingMessage.run_id)
        }
      }
    } else {
      if (message.run_id) {
        message.run = await getRun(message.run_id)
        message.run.steps = await getRunSteps(message.run_id)
      }
      if (message.file_ids && message.file_ids.length > 0) {
        message.files =
          (await Promise.all(message.file_ids.map(id => fetch(apiUrl('files/' + id)).then(res => res.json()))))
            .filter(e => e.filename)
      }
      current.value.messages.push(message)
    }
  }
}

const awaitRun = (run: Run) => {
  return new Promise<Run>((resolve, reject) => {
    let paused = false
    const interval = setInterval(() => {
      if (paused) return
      paused = true
      updateCurrentThread()
      fetch(apiUrl(`threads/${run.thread_id}/runs/${run.id}`))
        .then(res => res.json())
        .then(async (res: Run) => {
          if (!res.status) {
            clearInterval(interval)
            return reject(JSON.stringify(res))
          }
          if (res.status === 'requires_action' && res.required_action?.type === 'submit_tool_outputs') {
            const outputs = await Promise.all(res.required_action.submit_tool_outputs.tool_calls.map(async (toolCall) => {
              const args = JSON.parse(toolCall.function.arguments)
              console.log('Calling function ' + toolCall.function.name, args)
              return {
                tool_call_id: toolCall.id,
                output: await functions[toolCall.function.name].action(args)
              }
            }))
            console.log('Submitting tool outputs', outputs)
            await fetch(apiUrl(`threads/${run.thread_id}/runs/${run.id}/submit_tool_outputs`), {
              method: 'POST',
              body: JSON.stringify(outputs)
            }).then(res => {
              if (res.status !== 200) {
                throw new Error('Invalid response: ' + res.statusText)
              }
            })
          } else if (res.status !== 'queued' && res.status !== 'in_progress') {
            clearInterval(interval)
            resolve(res)
          }
        })
        .catch(e => {
          clearInterval(interval)
          reject(e)
        })
        .finally(() => paused = false)
    }, 1000)
  })
}

const buildToolsArray = () => {
  const arr = []
  if (codeInterpreter.value) {
    arr.push({ type: 'code_interpreter' })
  }
  if (retrieval.value) {
    arr.push({ type: 'retrieval' })
  }
  selectedFunctions.value.forEach(name => {
    arr.push({
      type: 'function',
      function: {name,  ...functions[name]}
    })
  })
  return arr
}

const generateFirst = async (): Promise<Run> => {
  const run: Run = await fetch(apiUrl('threads/create_and_run'), {
    method: 'POST',
    body: JSON.stringify({
      model: model.value,
      instructions: systemPrompt.value || null,
      messages: [{ role: 'user', content: userPrompt.value }],
      tools: buildToolsArray(),
      files: await Promise.all(files.value.map(async file => ({
        name: file.name,
        data: Array.from(new Int8Array(await file.arrayBuffer()))
      })))
    })
  }).then(res => res.json())
  if (!run.thread_id) throw new Error(`Received invalid response: ${JSON.stringify(run)}`)
  current.value.id = run.thread_id
  return run
}

const generateMore = async (): Promise<Run> => {
  const message: Message = await fetch(apiUrl(`threads/${current.value.id}/messages`), {
    method: 'POST',
    body: JSON.stringify({
      role: 'user',
      content: userPrompt.value,
    })
  }).then(res => res.json())
  if (!message.content) throw new Error(`Received invalid response: ${JSON.stringify(message)}`)
  current.value.messages.push(message)
  return await fetch(apiUrl(`threads/${current.value.id}/runs`), {
    method: 'POST',
    body: JSON.stringify({
      model: model.value,
      instructions: systemPrompt.value || null,
      tools: buildToolsArray(),
    })
  }).then(res => res.json())
}

const generate = async () => {
  generating.value = true
  try {
    let run: Run
    let consumeUserInput = true
    if (current.value.messages[current.value.messages.length - 1]?.role === 'user') {
      consumeUserInput = false
      run = await fetch(apiUrl(`threads/${current.value.id}/runs`), {
        method: 'POST',
        body: JSON.stringify({
          model: model.value,
          instructions: systemPrompt.value || null,
          tools: buildToolsArray(),
        })
      }).then(res => res.json())
    } else if (userPrompt.value) {
      if (current.value.messages.length === 0) {
        run = await generateFirst()
      } else {
        run = await generateMore();
      }
    } else {
      return
    }
    await awaitRun(run)
    await updateCurrentThread()
    const userPromptBackup = userPrompt.value
    if (consumeUserInput) {
      userPrompt.value = ''
      files.value = []
    }
    if (!current.value.title) {
      await fetch(apiUrl('generate'), {
        method: 'POST',
        body: JSON.stringify({
          model: 'gpt-4',
          content: [
            {role: 'system', content: SUMMARIZE_PROMPT},
            {role: 'user', content: userPromptBackup},
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
      sidebar.value.update()
    }
  } finally {
    generating.value = false
  }
}

const filterModels = (): Array<{ title: string, value: string }> => {
  if (!models.value) return []
  if (retrieval.value) {
    return models.value.filter(e => e.value === 'gpt-4-1106-preview')
  } else {
    return models.value
  }
}

const isSelectedValidModel = () => filterModels().find(e => e.value === model.value)

fetch(apiUrl('models'))
  .then(res => res.json())
  .then(value => Object.keys(value).map(k => ({ value: k, title: value[k] })))
  .then(array => models.value = array)
  .then(() => model.value = 'gpt-4-1106-preview')
  .then(() => {
    const modelOnLocalStorage = localStorage.getItem('assistant-model')
    if (modelOnLocalStorage && models.value.find(e => e.value === modelOnLocalStorage)) {
      model.value = modelOnLocalStorage
    }
  })
</script>
