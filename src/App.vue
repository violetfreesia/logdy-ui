<script setup lang="ts">
import { StyleValue, computed, onMounted, ref, watch } from 'vue';
import { Row, Facet, Message, CellHandler, Column, Settings, Middleware, StoredMessage } from "./types"
import { Layout } from "./config"
import { storageLayout, storageLogs } from "./storage"
import Drawer from "./components/Drawer.vue"
import SettingsDrawer from "./components/SettingsDrawer.vue"
import FacetComponent from "./components/Facet.vue"
import Filter from "./components/Filter.vue"
import Modal from "./components/Modal.vue"
import AuthPrompt from "./components/AuthPrompt.vue"
import DemoBar from "./components/DemoBar.vue"
import UpdateBar from "./components/UpdateBar.vue"
import Confirm from "./components/ConfirmModal.vue"
import Import from "./components/Import.vue"
import LoadLogs from "./components/LoadLogs.vue"
import DateModal from "./components/DateModal.vue"
import DoubleLeft from "./components/icon/DoubleLeft.vue"
import DoubleRight from "./components/icon/DoubleRight.vue"
import HideColumnIcon from "./components/HideColumnIcon.vue"
import FilterIcon from "./components/icon/Filter.vue"
import ExportLogs from "./components/ExportLogs.vue"
import ContextMenu from "./components/ContextMenu.vue";
import { useMainStore, InitSettings } from './store';
import { useFilterStore } from "./stores/filter";
import { startDragging, endDragging, startColumnDragging } from './dragging';
import * as demo from "./demo";
import { initKeyEventListeners } from "./key_events"
import loadAnalytics from './analytics';
import { client } from "./api"
import TopBar from "./components/TopBar.vue"
import Close from './components/icon/Close.vue';
import { useContextMenuStore } from './stores/contextMenu';
import { globalEventBus } from './event_bus';
import { useNotificationBarStore } from './stores/notificationBar';
import { hashStringToRgb } from './utils';

const store = useMainStore()
const storeFilter = useFilterStore()

const table = ref<HTMLElement>()

const columns = ref<Column[]>([])
const sampleLineIndex = ref<number>(0)
const traceResolution = ref<number>(20)
const searchbar = ref<string>("")

const changeTraceResolution = (delta: number) => {
  if (traceResolution.value + delta <= 0) {
    return
  }

  traceResolution.value += delta
}

const leftColHidden = ref<boolean>(false)
console.log("Logdy version", __APP_VERSION__)

storageLogs.startClearingUnknowns()

const toggleLeftCol = () => {
  leftColHidden.value = !leftColHidden.value
}

const addToFacet = (f: Facet) => {
  if (!store.facets[f.name]) {
    store.facets[f.name] = {
      items: [],
      toggled: true,
      name: f.name
    }
  }
  let idx = store.facets[f.name].items.findIndex(v => v.label === f.value)
  if (idx < 0) {
    store.facets[f.name].items.push({
      count: 0,
      selected: false,
      label: f.value
    })
    idx = store.facets[f.name].items.length - 1
  }
  store.facets[f.name].items[idx].count++
}

const removeFromFacet = (r: Row) => {
  r.facets.forEach(f => {
    let idx = store.facets[f.name].items.findIndex(v => v.label === f.value)
    store.facets[f.name].items[idx].count--
    if (store.facets[f.name].items[idx].count <= 0) {
      store.facets[f.name].items.splice(idx, 1)
    }
  })
}

const processMiddlewares = (msg: Message, middlewares: Middleware[]): Message | void => {
  for (let i in middlewares) {
    let _msg = middlewares[i].handler!(msg)
    if (_msg == null) {
      return
    }
    msg = _msg
  }

  return msg
}

const tryAddMessage = (msgs: Message[], settings: Settings): Message[] => {
  try {
    let mm = msgs.map(m => processMiddlewares(m, settings.middlewares)).filter(m => m) as Message[]
    store.refeshFilterCorrelated()
    if (msgs) {
      return addMessages(mm)
    }
  } catch (e) {
    // todo: messages that cannot be processed should land in DLQ or some kind of buffer
    console.error("Could not process message", e)
    return []
  }

  return []
}

const addMessages = (msgs: Message[]): Message[] => {
  //filter rows that are already present
  msgs = msgs.filter(msg => {
    // remove messags that are older than the first message in a table
    if (store.rows.length > 0 && msg.ts < store.rows[0].msg.ts) {
      return false
    }
    if (!store.rowsIds[msg.id]) {
      return true
    }
    return false
  })
  let spaceTaken = store.rows.length
  let spaceTotal = store.layout.settings.maxMessages
  let spaceAvail = spaceTotal - spaceTaken // space left
  let spaceNeeded = msgs.length
  // is there a space to fit all items?
  // if not, how much space is needed?
  if (spaceNeeded > 0 && spaceNeeded > spaceAvail) {
    // if space needed > msgs length, then cut msgs and clear space fully
    if (spaceNeeded > spaceTotal) {
      msgs = msgs.splice(msgs.length - spaceTotal)
      removeMessage(spaceTotal)
    } else {
      // if space needed <= msgs length, then free space up to needed point
      removeMessage(spaceNeeded - spaceAvail)
    }
  }

  if (msgs.length === 0) {
    return []
  }

  let toAdd: Row[] = []
  let correlationIdIdx: number = -1
  if(store.layout.settings.correlationIdField){
    correlationIdIdx = store.layout.columns.findIndex(c => c.name === store.layout.settings.correlationIdField)
  }
  msgs.forEach(m => {

    let cells = store.layout.columns.filter(c => !c.hidden).map((l): CellHandler => {
      try {
        let h = l.handler!(m)
        if (l.faceted && h.text !== undefined) {
          h.facets = (h.facets || [])
          h.facets.push({
            name: l.name,
            value: h.text
          })
        }
        h.facets?.forEach(addToFacet)
        return h
      } catch (e) {
        console.log("Error while running handler", { message: m, error: e })
        return { text: "error" }
      }
    })
    let fields = store.layout.columns.filter(c => c.hidden).map((l): CellHandler => {
      try {
        let h = l.handler!(m)
        if (l.faceted && h.text !== undefined) {
          h.facets = (h.facets || [])
          h.facets.push({
            name: l.name,
            value: h.text
          })
        }
        h.facets?.forEach(addToFacet)
        return h
      } catch (e) {
        console.log("Error while running handler", { message: m, error: e })
        return { text: "error" }
      }
    })

    if(store.layout.settings.correlationIdField && !m.correlation_id){
      m.correlation_id = cells[correlationIdIdx].text
    }

    store.rowsIds[m.id] = true
    toAdd.push({
      id: m.id,
      orderKey: m.order_key || 0,
      msg: m,
      cells,
      fields,
      facets: cells.map(c => c.facets || []).flat().concat(fields.map(c => c.facets || []).flat())
    })
  })

  store.rows.push(...toAdd)

  if ((toAdd[0] as any).orderKey) {
    store.rows.sort((a, b) => { return (a.orderKey || 0) >= (b.orderKey || 0) ? 1 : -1 })
  }

  setTimeout(() => {
    if (store.stickedToBottom) {
      stickToBottom()
    }
  }, 10)

  return msgs
}

const shouldStickToBottom = () => {
  return table.value && (table.value.scrollTop + table.value.offsetHeight + 20) >= table.value.scrollHeight
}

const removeMessage = (count: number = 1): string[] => {
  let ids = []
  for (let i = 0; i < count; i++) {
    if (!store.rows[i]) {
      continue
    }
    delete store.rowsIds[store.rows[i].id]
    if (store.rows[i].opened) {
      storeFilter.changeFilter('read', -1)
    } else {
      storeFilter.changeFilter('unread', -1)
    }
    store.rows[i].starred && storeFilter.changeFilter('starred', -1)
    ids.push(store.rows[i].id)
    removeFromFacet(store.rows[i])
    storageLogs.removeFirst()
  }
  store.rows.splice(0, count)

  return ids
}

const loadStorage = () => {
  let msgs = storageLogs.load()
  storeFilter.reset()
  tryAddMessage(msgs.map(sm => {
    return sm.message
  }), store.layout.settings)

  let msgId: Record<string, StoredMessage> = {}
  msgs.forEach(m => {
    msgId[m.message.id] = m
  })

  store.rows.forEach((r, k) => {
    let stored = msgId[r.id]
    if (!stored) {
      return
    }

    store.rows[k].starred = stored.starred
    stored.starred && storeFilter.changeFilter('starred', 1)
    if (stored.opened) storeFilter.changeFilter('read', 1)
    else storeFilter.changeFilter('unread', 1);

    if (stored.message.origin?.file) {
      storeFilter.changeFilter('origin_file_' + stored.message.origin.file, 1)
    }
    else if (stored.message.origin?.port) {
      storeFilter.changeFilter('origin_port_' + stored.message.origin.port, 1)
    } else {
      storeFilter.changeFilter('origin_na', 1)
    }

    store.rows[k].opened = stored.opened

    delete msgId[r.id]
  })

  Object.keys(msgId).forEach(mi => {
    storageLogs.remove(mi)
  })
}

const loadConfig = (load?: Layout) => {

  let layouts = load ? [load] : storageLayout.load()

  if (store.initSettings?.configStr) {
    console.log("Loading Layout from backend config")
    layouts = [JSON.parse(store.initSettings?.configStr)]
  }

  if (layouts[0]) {
    store.layout.loadFromObj(layouts[0])
  } else {
    store.layout.add({
      id: "",
      name: "raw",
      handlerTsCode: `(line: Message): CellHandler => {
          return { text: line.content || "-"}
      }`
    })
  }
}

const loadColumnsFromLayout = () => {
  columns.value = store.layout.columns.filter(col => !col.hidden)
}

const addRawColumn = () => {
  columnEdited({
    name: "raw",
    id: "new",
    width: 500,
    handlerTsCode: `(line: Message): CellHandler => {
    return { text: line.content || "-"}
}`
  })
}

const columnEdited = (col: Column) => {
  if (col.id === 'new') {
    store.layout.add(col)
  } else {
    store.layout.update(col)
  }
  storageLayout.update('main', store.layout as Layout)
  render()
}

const layoutLoaded = (lt: Layout) => {
  store.layout = lt
  storageLayout.update('main', store.layout as Layout)
  render()
}

const columnRemoved = (colId: string) => {
  store.layout.removeColumn(colId)
  storageLayout.update('main', store.layout as Layout)
  render()
}

const settingsUpdate = (settings: Settings) => {
  store.layout.settings = settings
  store.layout.processMiddlewareHandlers()
  storageLayout.update('main', store.layout as Layout)
  render()
}

const render = () => {
  store.rows = []
  store.clearRowIds()
  store.facets = {}
  loadColumnsFromLayout()
  loadStorage()
}

const connectToWs = () => {
  let wsProto = window.location.protocol === 'https:' ? 'wss' : 'ws'
  const endpoint = wsProto + '://' + window.location.host + window.location.pathname + 'ws'
  console.log("Connecting to WS", endpoint)
  const socket = new WebSocket(endpoint + '?password=' + store.getPassword());
  store.status = 'not connected'
  var wasOpened = false

  socket.onopen = () => {
    wasOpened = true
    store.status = 'connected'
    render()
  }

  socket.onclose = () => {
    if (socket.CLOSED && wasOpened) {
      store.status = 'not connected'
      connectToWs()
    }
  }

  socket.onerror = (_: Event) => {
    socket.close()
    setTimeout(() => {
      console.log("Reconnecting to WS")
      connectToWs()
    }, 1000)
  }

  socket.onmessage = (msg: MessageEvent) => {
    let m = JSON.parse(msg.data)

    switch (m.message_type) {
      case "client_joined":
        client.setAuthToken(m.client_id);
        break
      case "client_msg_status":
        if (!store.receiveCounters) {
          store.receiveCounters = {
            MessageCount: 0,
            MessagesToTail: 0,
            LastDeliveredIdx: 0
          }
        }

        store.receiveCounters.MessageCount = (m as any).stats.msg_count
        store.receiveCounters.MessagesToTail = (m as any).client.count_to_tail
        store.receiveCounters.LastDeliveredIdx = (m as any).client.last_delivered_id_idx
        break
      case "log_bulk":
        let msgs = tryAddMessage(m.messages, store.layout.settings);
        storeFilter.changeFilter('unread', msgs.length);
        msgs.forEach(msg => {
          storageLogs.add({ id: msg.id, message: msg }, msg.id)
          if (msg.origin?.file) {
            storeFilter.changeFilter('origin_file_' + msg.origin.file, 1)
          }
          if (msg.origin?.port) {
            storeFilter.changeFilter('origin_port_' + msg.origin.port, 1)
          }
        })
        break
      default:
        console.error(m)
        throw new Error("Unrecognized message")
    }

  }
}

const loadDemoMode = () => {
  store.status = 'connected'
  const numPerSec = 2
  renderDemoMode()

  setInterval(() => {
    if (store.demoStatus === 'stopped') {
      return
    }
    addDemoData()
  }, (1 / numPerSec) * 1000)
}

const renderDemoMode = () => {
  store.layout = demo.getLayout(store.demoContent === 'json')
  store.layout.processMiddlewareHandlers()
  store.clearAllRows()
  render()
}

const addDemoData = (count: number = 1) => {
  let isJson = store.demoContent === 'json'

  while (count--) {
    let rand = Math.random()
    let origin = {
      port: "",
      file: "",
      api_source: ""
    }

    if (rand <= 0.1) {
      storeFilter.changeFilter('origin_na', 1)
    }
    if (rand > 0.1 && rand <= 0.25) {
      origin.file = "foo/bar.log"
      storeFilter.changeFilter('origin_file_' + origin.file, 1)
    }
    if (rand > 0.25 && rand <= 0.5) {
      origin.port = "8999"
      storeFilter.changeFilter('origin_port_' + origin.port, 1)
    }
    if (rand > 0.5 && rand <= 0.75) {
      origin.file = "foo.log"
      storeFilter.changeFilter('origin_file_' + origin.file, 1)
    }
    if (rand > 0.75) {
      origin.port = "8123"
      storeFilter.changeFilter('origin_port_' + origin.port, 1)
    }
    let data = demo.generateData(isJson)
    tryAddMessage([{
      id: new Date().getTime().toString() + Math.random(),
      content: isJson ? JSON.stringify(data) : data as string,
      is_json: isJson,
      log_type: 0,
      json_content: isJson ? data : null,
      origin,
      ts: new Date().getTime(),
    }], store.layout.settings)
    storeFilter.changeFilter('unread', 1);

  }
}

const updateSearchbar = () => {
  store.searchbar = searchbar.value
}

const updateDates = (range: { from: number, to: number }) => {
  store.datepicker = { ...range }
}

globalEventBus.on('searchbar-update', (value: string) => {
  store.searchbar = value
  searchbar.value = value
})

const clearSearchbar = () => {
  store.searchClear()
  searchbar.value = ""
}

watch(() => store.initSettings?.received, (newVal?: boolean) => {
  if (newVal && store.initSettings?.analyticsEnabled) {
    loadAnalytics(false)
  }
})

const hideColumn = (col: Column) => {
  useMainStore().confirm("确定要隐藏这一列吗？之后可以在设置中恢复。", () => {
    col.hidden = true
    columnEdited(col)
  })
}

onMounted(async () => {
  // themeHandler.loadTheme()
  if (store.demoMode) {
    loadDemoMode()
    loadAnalytics(true)
    render()
  } else {
    await initWs()
  }


  initKeyEventListeners()

  table.value?.addEventListener("scroll", () => {
    if (!shouldStickToBottom()) {
      store.stickedToBottom = false
    } else {
      store.stickedToBottom = true
    }
  })
})

const postAuth = () => {
  store.modalShow = ""
  connectToWs()
  if (store.initSettings?.analyticsEnabled) {
    loadAnalytics(false)
  }
  loadConfig()
}

const initWs = async (): Promise<boolean> => {
  let init = await client.sendGet<InitSettings>("status")

  store.initSettings = init.json!

  if (init.json) {
    useNotificationBarStore().processNotification(init.json?.updateVersion)
  }

  let passValid = await client.sendGet("check-pass?password=" + store.getPassword())
  if (store.initSettings.authRequired && passValid.status !== 200) {
    store.modalShow = "auth"
  } else {
    postAuth()
  }

  return true
}

const reorderColumns = (colId: string, diff: number) => {
  store.layout.move(colId, diff)
  render()
}

const stickToBottom = () => {
  table.value!.scrollTop = table.value!.scrollHeight + 30
}

const changeDemoMode = (mode: "json" | "string") => {
  store.demoContent = mode
  renderDemoMode()
}

const sampleLine = computed(() => {
  return store.rows && store.rows[sampleLineIndex.value] && store.rows[sampleLineIndex.value].msg
})

const updateSampleLine = () => {
  sampleLineIndex.value = Math.floor(Math.random() * store.rows.length)
}

</script>

<template>
  <Modal @close="store.modalShow = ''" v-if="store.modalShow">
    <AuthPrompt v-if="store.modalShow == 'auth'" @success="postAuth" />
    <Import v-if="store.modalShow == 'import'" :layout="(store.layout as Layout)" @layout-loaded="layoutLoaded" />
    <ExportLogs v-if="store.modalShow == 'export-logs'" :rows="store.rows" :visible-rows="store.displayRows"
      :layout="(store.layout as Layout)" />
    <LoadLogs v-if="store.modalShow == 'load-logs'" />
  </Modal>
  <ContextMenu v-if="useContextMenuStore().display" />
  <Confirm />

  <SettingsDrawer v-if="store.settingsDrawer" @close="store.settingsDrawer = false" :layout="(store.layout as Layout)"
    @edit="columnEdited" @remove="columnRemoved" @move="reorderColumns" @settings-update="settingsUpdate"
    @update-sample-line="updateSampleLine" :sampleLine="sampleLine" />
  <UpdateBar v-if="useNotificationBarStore().display" />
  <DemoBar v-if="store.demoMode" @start="store.demoStatus = 'started'" @stop="store.demoStatus = 'stopped'"
    @mode="changeDemoMode" @add="addDemoData(100)" />
  <div :class="{ 'demo': store.demoMode, 'update': useNotificationBarStore().display }">
    <div class="top-bar">
      <div class="left"></div>
      <div class="right">
        <input type="text" class="searchbar" id="searchbar-query" v-model="searchbar" @keyup.enter="updateSearchbar"
          placeholder="输入查询后按 Enter（由 breser.dev 支持）" />
        <!-- <button class="btn clear" @click="updateSearchbar" style="display: flex;align-items: center;">
          Search
          <CornerRightDown
            style="margin-left: 5px; border:1px solid white; border-radius: 2px; padding: 2px; height: 10px; width:10px" />
        </button> -->
        <button class="btn clear" @click="clearSearchbar">
          <Close />
        </button>
        <span class="search-error" v-if="store.searchbarValid.length > 0">查询语法无效：{{ store.searchbarValid
        }}。
          <br />
          <a href="https://logdy.dev/docs" target="_blank">查看文档</a>
        </span>
      </div>
      <div class="end">
        <TopBar @stick-bottom="stickToBottom" />
      </div>
    </div>
    <div class="layout" @mouseup="endDragging">
      <div class="left-shell">
        <div class="left-col" :style="{ width: store.layout.settings.leftColWidth + 'px' }"
          :class="{ empty: leftColHidden }">
          <div class="sidebar-toggle">
            <DoubleLeft v-if="!leftColHidden" @click="toggleLeftCol" />
            <DoubleRight v-if="leftColHidden" @click="toggleLeftCol" />
          </div>
          <template v-if="!leftColHidden">
            <DateModal inline @change="updateDates" />
            <div class="counter">
              <span class="counter-total">{{ store.displayRows.length }} / {{ store.rows.length }} 条日志</span>
              <div class="counter-actions">
                <button class="btn-sm" @click="store.modalShow = 'export-logs'">导出消息</button>
                <button class="btn-sm" @click="store.resetAllFiltersAndFacets()">重置全部筛选</button>
              </div>
              <div v-if="store.correlationFilter" class="alert alert-info">
                关联筛选已启用（{{ store.correlationFilter }}）
                <br />
                调整分辨率：
                <button class="btn-sm" @click="changeTraceResolution(1)">-</button>
                <button class="btn-sm" @click="changeTraceResolution(-1)">+</button>
                <br />
                <button class="btn-sm" @click="store.resetCorrelationFilter()">重置关联筛选</button>
              </div>
              <div class="sort-controls">
                <span class="sort-label">分面排序</span>
                <button class="btn-sm" @click="store.facetSort = 'label'"
                  :disabled="store.facetSort === 'label'">标签</button>
                <button class="btn-sm" @click="store.facetSort = 'count'"
                  :disabled="store.facetSort === 'count'">数量</button>
              </div>
            </div>
            <Filter />
            <FacetComponent :facets="store.facets" :sort="store.facetSort" />
          </template>
        </div>
      </div>

      <div>
        <div class="mid-col" :class="{ freeze: leftColHidden }" @mousedown="startDragging"></div>
      </div>
      <div class="right-col" ref="table">
        <div v-if="columns.length === 0" class="empty-state">

          <div v-if="useMainStore().status == 'not connected'">状态：<strong>未连接</strong></div>

          <template v-else>
            尚未定义列，打开 <span class="clickable" @click="store.settingsDrawer = true">设置</span> 后添加列<br />
            或者 <span class="clickable" @click="addRawColumn">立即添加包含原始内容的列</span>。
          </template>
        </div>
        <template v-else>
          <table class="table" cellspacing="0" cellpadding="0">
            <tr>
              <th></th>
              <th v-for="col in columns" :style="{ width: col.width + 'px', cursor: 'auto' }" class="column-name"
                @contextmenu.prevent="useContextMenuStore().show($event, { type: 'column_header', name: col.name })">
                <span>{{ col.name }}</span>
                <FilterIcon v-if="col.faceted" :style="{ opacity: store.isFacetActive(col.name) ? 1 : 0.2 }" />
                <div class="hide-icon" @click="hideColumn(col)">
                  <HideColumnIcon />
                </div>
                <div class="header-border" @mousedown="startColumnDragging(col.id)">
                  &nbsp;</div>
              </th>
              <th v-if="store.correlationFilter">链路
              </th>
            </tr>
            <tr class="row" :class="{ opened: row.opened, open: row.open }" v-for="row in store.displayRows"
              @click="store.openLogDrawer(row)" :style="(row.msg.style as StyleValue || {})">
              <td>
                <span class="mark" :class="{ active: row.starred }" @click.stop="store.toggleRowMark(row)">
                  ⬤
                </span>
              </td>
              <td class="cell" v-for="c, k2 in columns" :style="Object.assign(row.cells[k2].style as StyleValue || {},{
                  backgroundColor: (store.layout.settings.paintCorrelationIdCell && store.layout.settings.correlationIdField == c.name) ? hashStringToRgb(row.cells[k2].text||'', 40): ''
                } as StyleValue)"
                :class="{ 'cell-error': row.cells[k2].error }">
                <div v-if="row.cells[k2].allowHtmlInText" :style="{ width: columns[k2].width + 'px' }"
                  v-html="row.cells[k2].text !== undefined ? row.cells[k2].text : row.cells[k2].error || '&nbsp;'"
                  @contextmenu.prevent="useContextMenuStore().show($event, { type: 'cell', value: row.cells[k2].text, columnId: c.id, error: row.cells[k2].error })">
                </div>
                <div v-else :style="{ width: columns[k2].width + 'px' }"
                  @contextmenu.prevent="useContextMenuStore().show($event, { type: 'cell', value: row.cells[k2].text, columnId: c.id, error: row.cells[k2].error })">
                  {{ row.cells[k2].text !== undefined ? row.cells[k2].text : row.cells[k2].error || "&nbsp;" }}
                </div>

              </td>
              <td class="cell trace-cell" v-if="store.correlationFilter">
                <div v-if="store.tracesRows[row.id] && store.tracesRows[row.id].id === row.id" class="trace-block"
                  v-tooltip="store.tracesRows[row.id].label || ''" :style="{
                    width: store.tracesRows[row.id].width / traceResolution + 'px',
                    marginLeft: store.tracesRows[row.id].offset / traceResolution + 'px',
                    ...store.tracesRows[row.id].style
                  }">
                  {{ store.tracesRows[row.id].label || '&nbsp' }}
                </div>
                <template v-else>-</template>
              </td>
            </tr>
          </table>
        </template>
      </div>
      <div class="drawer-col">
        <Drawer :row="store.drawer.row" :layout="(store.layout as Layout)" @close="store.closeLogDrawer" />
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss"></style>
