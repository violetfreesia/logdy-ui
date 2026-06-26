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
                <div class="nav-hint">
                    <ArrowUp /> 下一条 /
                    <ArrowDown /> 上一条
                </div>

                <button @click="$emit('close')">关闭 <kbd>Esc</kbd></button>
            </div>
            <div class="drawer-actions">
                <button @click="useMainStore().filterCorrelated(row.msg)"
                    :disabled="!row.msg.correlation_id && !layout?.settings.correlationIdField">
                    显示关联日志
                </button>
                <button @click="useMainStore().resetCorrelationFilter()" v-if="useMainStore().correlationFilter"
                    class="btn-danger">
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
                <pre class="code-panel" v-if="row.cells[k] && !row.cells[k].isJson">{{ row.cells[k].text || row.cells[k].error }}</pre>
                <pre class="code-panel"
                    v-else-if="row.cells[k].text"><VueJsonPretty :theme="'dark'" :data="JSON.parse(row.cells[k].text!)"></VueJsonPretty></pre>

            </div>
            <h3>非表格字段</h3>
            <div v-for="col, k in layout?.columns.filter(c => c.hidden)">
                <h4 v-tooltip="'点击复制'" style="display: inline;"
                    @click="copyToClipboard(row.fields[k].text || row.cells[k].error)">{{
                        col.name }}
                    <Clipboard :class="'clipboard'" />
                </h4>
                <pre class="code-panel" v-if="!row.fields[k].isJson">{{ row.fields[k].text }}</pre>
                <pre class="code-panel" v-else><VueJsonPretty  :theme="'dark'" :data="row.fields[k].text"></VueJsonPretty></pre>
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
                    <pre class="code-panel"><VueJsonPretty  :theme="'dark'" :data="row.msg.json_content"></VueJsonPretty></pre>
                </div>
                <div v-if="!row.msg.is_json" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;" @click="copyToClipboard(row.msg.content)">
                        原始消息
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre class="code-panel"><code>{{ row.msg.content }}</code></pre>
                </div>
                <div v-if="row.msg.timing" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(JSON.stringify(row.msg.timing))">
                        耗时
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre class="code-panel"><VueJsonPretty  :theme="'dark'" :data="row.msg.timing"></VueJsonPretty></pre>
                </div>
                <div v-if="row.msg.origin?.port" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.port)">
                        来源端口
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre class="code-panel"><code>{{ row.msg.origin?.port }}</code></pre>
                </div>
                <div v-if="row.msg.origin?.api_source" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.api_source)">
                        来源 API
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre class="code-panel"><code>{{ row.msg.origin?.api_source }}</code></pre>
                </div>
                <div v-if="row.msg.origin?.file" class="raw">
                    <h4 v-tooltip="'点击复制'" style="display: inline;"
                        @click="copyToClipboard(row.msg.origin?.file)">
                        来源文件名
                        <Clipboard :class="'clipboard'" />
                    </h4>
                    <pre class="code-panel"><code>{{ row.msg.origin?.file }}</code></pre>
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
    height: 100%;
    max-width: min(900px, 48vw);
    min-width: 360px;
    background: var(--surface);
    border-left: 1px solid var(--border);
    z-index: 90;
    padding: 0;

    .resize-handle {
        background: var(--border);
        width: 4px;
        cursor: ew-resize;
        height: 100%;
        float: left;
        margin-right: 0;

        &:hover {
            background: var(--accent);
        }
    }

    h4,
    h3 {
        margin: 12px 0 6px;
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
        padding: 14px;
        height: 100%;
        overflow-y: auto;
        padding-bottom: 72px;


        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            position: sticky;
            top: 0;
            z-index: 2;
            margin: -14px -14px 12px;
            padding: 10px 14px;
            border-bottom: 1px solid var(--border);
            background: var(--surface);

            .nav-hint {
                display: flex;
                align-items: center;
                gap: 4px;
                color: var(--font-muted);
                font-size: 12px;
            }
        }
    }

    .drawer-actions {
        display: flex;
        gap: 8px;
        flex-wrap: wrap;
    }

}
</style>
