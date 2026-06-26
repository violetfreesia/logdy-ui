<script setup lang="ts">
import { useMainStore } from '../store';

defineEmits<{
    (e: 'start'): void,
    (e: 'stop'): void,
    (e: 'add'): void,
    (e: 'mode', mode: "json" | "string"): void,
}>()
</script>

<template>
    <div class="demo-bar">
        <span>演示模式</span>
        <button @click="$emit('start')" :disabled="useMainStore().demoStatus === 'started'">启动</button>
        <button @click="$emit('stop')" :disabled="useMainStore().demoStatus === 'stopped'">停止</button>
        <button @click="$emit('add')">新增 100 条消息</button>
        <button @click="$emit('mode', 'json')" :disabled="useMainStore().demoContent === 'json'">生成 JSON</button>
        <button @click="$emit('mode', 'string')" :disabled="useMainStore().demoContent === 'string'">生成
            字符串</button>
    </div>
</template>

<style lang="scss">
.demo {
    .top-bar {
        margin-top: var(--banner-height);
    }

    .layout {
        height: calc(100vh - var(--topbar-height) - var(--banner-height)) !important;
    }
}

.demo-bar {
    position: fixed;
    display: flex;
    align-items: center;
    justify-content: center;
    top: 0;
    z-index: 120;
    gap: 8px;

    width: 100%;
    font-weight: 600;

    height: var(--banner-height);
    background: var(--surface-raised);
    border-bottom: 1px solid var(--border);
    box-shadow: var(--shadow);

    span {
        margin-right: 8px;
        color: var(--font-muted);
    }

    button {
        font-size: 12px;
    }
}
</style>
