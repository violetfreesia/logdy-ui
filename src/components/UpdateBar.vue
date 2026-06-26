<script setup lang="ts">
import { useNotificationBarStore } from '../stores/notificationBar';

const store = useNotificationBarStore()

</script>

<template>
    <div class="update-bar" v-if="store.updateResponse">
        <span>发现新版本 Logdy v{{ store.updateResponse.current_version }}！{{
            store.updateResponse.excerpt }} <a :href="store.updateResponse.blog_link"
                v-if="store.updateResponse.blog_link" target="_blank">了解更多</a></span>
        <button @click="store.hide">隐藏</button>
        <button @click="store.remindLater">稍后提醒</button>
    </div>
</template>

<style lang="scss">
.update {
    .top-bar {
        margin-top: var(--banner-height);
    }

    .layout {
        height: calc(100vh - var(--topbar-height) - var(--banner-height)) !important;
    }
}

.update-bar {
    position: fixed;
    display: flex;
    align-items: center;
    justify-content: center;
    top: 0;
    z-index: 120;
    gap: 8px;

    width: 100%;

    height: var(--banner-height);
    font-size: 13px;
    background: var(--surface-raised);
    border-bottom: 1px solid var(--border);
    box-shadow: var(--shadow);

    span {
        margin-right: 8px;
        color: var(--font-muted);
    }

    button {
        min-height: 26px;
        font-size: 11px;
    }
}
</style>
