<script setup lang="ts">
import { Row } from '../types';
import { Layout } from "../config"
import { ref } from 'vue';
import ArrowUp from './icon/ArrowUp.vue'
import ArrowDown from './icon/ArrowDown.vue'
import Clipboard from './icon/Clipboard.vue'
import { startDraggingDrawer } from '../dragging';
import { useMainStore } from '../store';
import VueJsonPretty from 'vue-json-pretty';
import 'vue-json-pretty/lib/styles.css';

withDefaults(defineProps<{
    row?: Row,
    layout?: Layout
}>(), {})

const showRaw = ref<boolean>(false)

defineEmits<{
    (e: 'close'): void
}>()

const copyToClipboard = (value: string | undefined) => {
    if (!value) {
        return
    }
    navigator.clipboard.writeText(value)
}

</script>

<template>
    <div class="drawer" v-if="row" :style="{ width: useMainStore().layout.settings.drawerColWidth + 'px' }">
        <div class="resize-handle" @mousedown="startDraggingDrawer"></div>
        <div class="inner-drawer">
            <div class="header">
                <div style="margin-right: 10px;">
                    <ArrowUp /> 下一条 /
                    <ArrowDown /> 上一条
                </div>

                <button @click="$emit('close')">关闭 <kbd>Esc</kbd></button>
            </div>
            <div>
                <button @click="useMainStore().filterCorrelated(row.msg)"
                    :disabled="!row.msg.correlation_id && !layout?.settings.correlationIdField">
                    显示关联日志
                </button>
                <button @click="useMainStore().resetCorrelationFilter()" v-if="useMainStore().correlationFilter"
                    style="margin-left: 5px">
                    重置关联筛选
                </button>
            </div>
            <hr />
            <h3>表格列</h3>
            <div v-for="col, k in layout?.columns.filter(c => !c.hidden)">
                <h4 v-tooltip="'点击复制'" style="display: inline;"
                    @click="copyToClipboard(row.cells[k].text || row.cells[k].error)">{{
                        col.name }}
                    <Clipboard :class="'clipboard'" />
                </h4>
                <pre v-if="row.cells[k] && !row.cells[k].isJson">{{ row.cells[k].text || row.cells[k].error }}</pre>
                <pre
                    v-else-if="row.cells[k].text"><VueJsonPretty :theme="'dark'" :data="JSON.parse(row.cells[k].text!)"></VueJsonPretty></pre>

            </div>
            <h3>非表格字段</h3>
            <div v-for="col, k in layout?.columns.filter(c => c.hidden)">
                <h4 v-tooltip="'点击复制'" style="display: inline;"
                    @click="copyToClipboard(row.fields[k].text || row.cells[k].error)">{{
                        col.name }}
                    <Clipboard :class="'clipboard'" />
                </h4>
                <pre v-if="!row.fields[k].isJson">{{ row.fields[k].text }}</pre>
                <pre v-else><VueJsonPretty  :theme="'dark'" :data="row.fields[k].text"></VueJsonPretty></pre>
            </div>
            <hr />
            <button @click="showRaw = !showRaw">显示/隐藏原始消息</button>
            <div v-if="showRaw">
                <div v-if="row.msg.is_json">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(JSON.stringify(row.msg.json_content))">
                        原始消息（JSON）
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><VueJsonPretty  :theme="'dark'" :data="row.msg.json_content"></VueJsonPretty></pre>
                </div>
                <div v-if="!row.msg.is_json" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;" @click="copyToClipboard(row.msg.content)">
                        原始消息
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><code>{{ row.msg.content }}</code></pre>
                </div>
                <div v-if="row.msg.timing" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(JSON.stringify(row.msg.timing))">
                        耗时
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><VueJsonPretty  :theme="'dark'" :data="row.msg.timing"></VueJsonPretty></pre>
                </div>
                <div v-if="row.msg.origin?.port" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.port)">
                        来源端口
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><code>{{ row.msg.origin?.port }}</code></pre>
                </div>
                <div v-if="row.msg.origin?.api_source" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.api_source)">
                        来源 API
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><code>{{ row.msg.origin?.api_source }}</code></pre>
                </div>
                <div v-if="row.msg.origin?.file" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.file)">
                        来源文件名
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre><code>{{ row.msg.origin?.file }}</code></pre>
                </div>
            </div>
            <div style="margin-bottom: 80px;">
                <!-- It's a spacer div, leave it here -->
                &nbsp;
            </div>
        </div>
    </div>
</template>

<style scoped lang="scss">
.drawer {
    right: 0;
    top: 0;
    height: 100%;
    background: var(--hl-bg);
    z-index: 999;
    opacity: 0.99;
    padding: 10px;
    padding-left: 0px;
    padding-top: 0;

    .resize-handle {
        background: white;
        width: 3px;
        opacity: 0.2;
        cursor: ew-resize;
        height: 100%;
        float: left;
        margin-right: 10px;
    }

    h4,
    h3 {
        margin: 2px;
        cursor: pointer;

        .clipboard {
            visibility: hidden;
        }

        &:hover {
            .clipboard {
                visibility: visible !important;
            }
        }
    }

    .inner-drawer {
        padding-top: 20px;
        height: 100%;
        overflow-y: scroll;
        padding-bottom: 20px;


        .header {
            display: flex;
            align-items: center;
            float: right;
        }
    }

    pre {
        margin: 6px 0;
        background: var(--bg);
        padding: 10px;
        white-space: pre-wrap;
    }

}
</style>
