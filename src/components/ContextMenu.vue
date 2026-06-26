<script setup lang="ts">
import { useContextMenuStore } from '../stores/contextMenu';
</script>

<template>
    <div>
        <div class="overlay" @click="useContextMenuStore().hide()"></div>

        <div class="context-menu"
            :style="{ top: useContextMenuStore().y + 'px', left: useContextMenuStore().x + 'px' }">
            <div v-for="action in useContextMenuStore().actions" @click="action.fn()"
                :class="{ disabled: action.disabled && action.disabled() }">
                {{ action.label }}
            </div>
        </div>
    </div>
</template>

<style scoped>
.context-menu {
    position: absolute;
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: var(--radius-md);
    box-shadow: var(--shadow);
    padding: 4px;
    min-width: 150px;
    z-index: 101;
    font-size: 12px;
    font-family: var(--code-font);

    .disabled {
        opacity: 0.5;
        pointer-events: none;
    }
}

.context-menu div {
    cursor: pointer;
    padding: 6px 8px;
    border-radius: var(--radius-sm);
}

.context-menu div:hover {
    background-color: var(--surface-hover);
}

.overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 100;
}

.overlay::before {
    content: '';
    position: absolute;
    width: 100%;
    height: 100%;
}
</style>
