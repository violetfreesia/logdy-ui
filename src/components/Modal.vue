<script setup lang="ts">
defineEmits<{
    (e: 'close'): void,
}>()
</script>

<template>
    <div class="modal">
        <div class="close">
            <button @click="$emit('close')">关闭</button>
        </div>
        <slot></slot>
    </div>
    <div class="overlay"></div>
</template>

<style lang="scss" scoped>
.overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: black;
    opacity: 0.54;
    z-index: 9999;
}

.modal {
    position: fixed;
    display: flex;
    flex-direction: column;
    top: 42px;
    left: 50%;
    transform: translate(-50%, 0);
    border: 1px solid var(--border);
    border-radius: var(--radius-md);

    width: min(900px, calc(100vw - 32px));
    max-height: calc(100% - 100px);
    overflow-y: auto;
    background: var(--surface);
    z-index: 10000;
    padding: 16px;
    box-shadow: var(--shadow);

    .close {
        position: sticky;
        top: -16px;
        z-index: 1;
        margin: -16px -16px 14px;
        padding: 10px 16px;
        border-bottom: 1px solid var(--border);
        background: var(--surface);
        text-align: right;
    }
}
</style>
