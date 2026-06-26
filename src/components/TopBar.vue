<script lang="ts" setup>
import { useMainStore } from '../store';
import Close from './icon/Close.vue'
import Pause from './icon/Pause.vue'
import Play from './icon/Play.vue'
import PlayNext from './icon/PlayNext.vue'
import Load from './icon/Load.vue'
import Bolt from './icon/Bolt.vue'
import Cog from './icon/Cog.vue';

const store = useMainStore()

defineEmits<{
    (e: 'stick-bottom'): void
}>()

const connectionStatusLabel = () => {
    return store.status === 'connected' ? '已连接' : '未连接'
}

const clearAll = () => {
    store.confirm("确定要清空全部日志吗？这只会清空浏览器中的日志，缓冲区日志不会受影响。", store.clearAllRows)
}

</script>
<template>
    <div class="stick-control">
        <button class="stick-button" @click="$emit('stick-bottom')" :class="{ sticked: store.stickedToBottom }">
            <template v-if="!store.stickedToBottom">定位到底部</template>
            <template v-else>已在底部</template>
        </button>
    </div>
    <div class="ctrls">
        <button :disabled="store.receiveStatus.includes('following')"
            @click="store.changeReceiveStatus('following_cursor')" class="ctrl-btn"
            v-tooltip="'从暂停位置继续接收消息'">
            <Play width="19" height="19" />
        </button>
        <button :disabled="store.receiveStatus.includes('following')" @click="store.changeReceiveStatus('following')"
            class="ctrl-btn" v-tooltip="'继续接收消息，并从最新位置开始（空格）'">
            <PlayNext width="19" height="19" />
        </button>
        <button :disabled="store.receiveStatus === 'paused'" @click="store.changeReceiveStatus('paused')"
            class="ctrl-btn" v-tooltip="'暂停接收消息（空格）'">
            <Pause width="19" height="19" />
        </button>
        <button :disabled="store.receiveStatus !== 'paused'" @click="useMainStore().modalShow = 'load-logs'"
            class="ctrl-btn" v-tooltip="'加载日志。连接：' + connectionStatusLabel() + ' 状态：' + store.statusStr">
            <Load width="19" height="19" />
            <Bolt :disabled="store.status === 'not connected'" :fill="store.status === 'connected' ? 'green' : 'red'"
                :size="'19'" />
        </button>
        <button @click="clearAll" class="ctrl-btn" v-tooltip="'清空全部消息'">
            <Close width="19" height="19" />
        </button>
    </div>
    <button class="settings-button" @click="store.settingsDrawer = true" v-tooltip="'设置'">
        <Cog />
    </button>
</template>

<style scoped lang="scss">
.stick-control {
    position: relative;
    margin-right: 8px;
}

.stick-button,
.settings-button {
    height: 34px;
    min-height: 34px;
}

.stick-button {
    min-width: 94px;
    max-width: 140px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    font-size: 12px;

    &.sticked {
        border-color: var(--accent);
        color: var(--accent-strong);
    }
}

.settings-button {
    width: 34px;
    padding: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}
</style>
