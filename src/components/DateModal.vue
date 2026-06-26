<script setup lang="ts">
import moment, { DurationInputArg2 } from 'moment';
import { onMounted, ref } from 'vue';
import { useMainStore } from '../store';

const props = withDefaults(defineProps<{
    inline?: boolean
}>(), {
    inline: false
})

const emit = defineEmits<{
    (e: 'close'): void,
    (e: 'change', range: { from: number, to: number }): void
}>()

type Period = "5_m" | "15_m" | "30_m" | "1_h" | "4_h" | "1_d"

const changePeriod = (period?: Period) => {
    if (!period) {
        emit("change", {
            from: 0,
            to: 0
        })
        if (!props.inline) {
            emit("close")
        }
    } else {
        let splitted = period.split("_")
        from.value = moment().subtract(parseInt(splitted[0]), splitted[1] as DurationInputArg2).format().slice(0, 19)
    }
}

const from = ref<string | undefined>()
const to = ref<string | undefined>()

onMounted(() => {
    const store = useMainStore()

    if (store.datepicker.from) {
        from.value = moment(store.datepicker.from * 1000).format().slice(0, 19)
    }
    if (store.datepicker.to) {
        to.value = moment(store.datepicker.to * 1000).format().slice(0, 19)
    }
})

const setToNow = (r: "from" | "to") => {
    switch (r) {
        case 'from':
            from.value = moment().format().slice(0, 19)
            break
        case 'to':
            to.value = moment().format().slice(0, 19)

    }
}

const submitDates = (close?: boolean) => {
    emit("change", {
        from: moment(from.value).unix(),
        to: to.value ? moment(to.value).unix() : 0
    })
    if (close) {
        emit("close")
    }
}

</script>
<template>
    <div :class="props.inline ? 'date-panel' : 'modal'">
        <div class="date-picker">
            <span class="form-label">开始时间：</span>
            <div>
                <input type="datetime-local" step="1" class="date-from" v-model="from" />
                <button class="btn-sm" @click="setToNow('from')">现在</button>
            </div>
            <div class="quick-ranges">
                <button class="btn-sm" @click="changePeriod('5_m')">过去 5 分钟</button>
                <button class="btn-sm" @click="changePeriod('15_m')">过去 15 分钟</button>
                <button class="btn-sm" @click="changePeriod('30_m')">过去 30 分钟</button>
                <button class="btn-sm" @click="changePeriod('1_h')">过去 1 小时</button>
                <button class="btn-sm" @click="changePeriod('4_h')">过去 4 小时</button>
                <button class="btn-sm" @click="changePeriod('1_d')">过去 1 天</button>
            </div>
            <hr style="width:100%" />
            <span class="form-label">结束时间：</span>
            <div>
                <input type="datetime-local" step="1" v-model="to" />
                <button class="btn-sm" @click="setToNow('to')">现在</button>
            </div>
            <hr style="width:100%" />
            <div class="button-row">
                <button class="btn-sm" @click="submitDates()">提交</button>
                <button class="btn-sm" @click="submitDates(true)">提交并关闭</button>
                <button class="btn-sm" @click="changePeriod()">清除</button>
            </div>
        </div>
    </div>
    <div v-if="!props.inline" class="overlay" @click="$emit('close')"></div>
</template>

<style lang="scss" scoped>
.overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 9999;
}

.modal,
.date-panel {
    position: absolute;
    display: flex;
    flex-direction: column;
    border: 1px solid var(--border);
    border-radius: var(--radius-md);
    overflow-y: auto;
    background: var(--surface);
    padding: 14px;
    box-shadow: var(--shadow);

    .date-picker {
        display: flex;
        flex-direction: column;
        gap: 8px;

        > div {
            display: flex;
            gap: 6px;
            align-items: center;
            flex-wrap: wrap;
        }

        .date-from {
            margin-bottom: 0;
        }

        input {
            min-width: 0;
            flex: 1 1 190px;
        }

        .quick-ranges {
            align-items: flex-start;
        }

        span.picker {
            cursor: pointer;
            padding: 3px;

            &:hover {
                background: rgba(255, 255, 255, 0.1);
            }
        }
    }
}

.modal {
    transform: translate(-50%, 0);
    width: 310px;
    max-height: calc(100% - 100px);
    z-index: 10000;
}

.date-panel {
    position: static;
    width: 100%;
    box-shadow: none;
}
</style>
