<script setup lang="ts">
import { useMainStore } from "../store";
import Modal from "./Modal.vue"
</script>

<template>
    <Modal v-if="useMainStore().confirmShow === true" @close="useMainStore().confirmProcess(false)">
        <div class="container">
            <p>{{ useMainStore().confirmMsg || `确定要继续吗？` }}</p>

            <div>
                <button @click="useMainStore().confirmProcess(true)">确认</button>
                <button @click="useMainStore().confirmProcess(false)">取消</button>
            </div>
        </div>
    </Modal>
</template>

<style lang="scss">
.container {
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;
    justify-content: center;

    button {
        margin: 0 4px;
    }

    p {
        max-width: 620px;
        color: var(--font);
    }
}
</style>
