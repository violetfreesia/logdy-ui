<script setup lang="ts">
import { computed, onMounted, ref } from 'vue';
import { client } from "../api"
import { useMainStore } from '../store';
import noUiSlider from 'nouislider';
import 'nouislider/dist/nouislider.css';
import { Message } from '../types';
import moment from 'moment';
import Info from "./icon/Info.vue"
import { formatThousands } from "../utils"

const offset = ref<number>(0)
const store = useMainStore()
const offsets = ref<[number, number]>([0, 0])
const lines = ref<Message[]>([])

const peek = async () => {
    let peeked = await client.peek(offsets.value[0], offsets.value[1])
    lines.value = peeked.json as Message[]
}

//@ts-expect-error
var slider;
const updateSliders = () => {
    //@ts-expect-error
    slider.noUiSlider.set([offset.value]);
}

const showSlider = computed(() => {
    return store.receiveCounters.MessageCount - store.layout.settings.maxMessages > 0
})

onMounted(() => {
    slider = document.getElementById('slider-square');
    let mm = store.layout.settings.maxMessages - 1
    let count = store.receiveCounters.MessageCount - mm

    if (!showSlider.value) {
        return
    }

    noUiSlider.create(slider!, {
        start: [0],
        tooltips: [
            {
                to: function (value: number) {
                    let max = value + mm
                    if (max > count) {
                        let over = value + mm - store.receiveCounters.MessageCount
                        max = count + mm + over
                    }
                    return formatThousands((Math.round(value))) + " - " + formatThousands(Math.round(max))
                }
            }
        ],
        range: {
            'min': 1,
            'max': count <= 0 ? mm + count : count
        },
        format: {
            from: function (value: string) {
                return parseInt(value)
            },
            to: function (value) {
                return Math.round(value)
            }
        }
    });

    // @ts-ignore
    slider!.noUiSlider.on('set', (e: number[]) => {
        offset.value = e[0]
        let offsetEnd = offset.value + store.layout.settings.maxMessages
        offsets.value = [offset.value - 1, offsetEnd - 2]
        peek()
    })
})

const load = async () => {
    store.clearAllRows()
    await client.load(offsets.value[0], store.layout.settings.maxMessages)
}

</script>

<template>
    <div>
        <h3>从缓冲区加载日志
            <Info v-tooltip="`所有日志消息都会由 Logdy 进程缓冲并保存在内存中，但 UI 中只能保留其中一部分
            （由“浏览器中存储的最大日志数”设置决定）。
            你可以在此页面选择一批消息，从缓冲区加载到 UI 中。`" />
        </h3>
        <p><strong>缓冲区状态</strong>：{{ store.statusStr }}</p>
        <div>
            从偏移位置加载：
            <input type="number" class="input" v-model="offset" @change="updateSliders" />
            <Info v-tooltip="`最多可加载 ${store.layout.settings.maxMessages} 条消息（取决于当前设置）`" />
            <br />

            <div class="slider-styled" id="slider-square" style="margin-top: 60px" v-if="showSlider"></div>
            <div v-else>
                <p>
                    <strong>
                        缓冲区中共有 {{ store.receiveCounters.MessageCount }} 条日志，可以全部加载。
                    </strong>
                </p>
            </div>
        </div>
        <br />
        <button class="btn" @click="load">加载消息</button>
        <template v-if="lines.length > 0">
            <hr />
            <small>指定偏移位置的日志消息预览</small>

            <div v-for=" line, k  in  lines ">
                <hr style="opacity: 0.3;" />
                消息 #{{ formatThousands(offsets[k] + 1) }}，接收时间：{{ moment(line.ts).format("DD/MM/YY HH:mm:ss")
                }}
                <pre v-if="!line.json_content">{{ line.content }}</pre>
                <pre v-if="line.json_content">{{ line.json_content }}</pre>
            </div>
        </template>
    </div>
</template>

<style>
.v-popper__inner {
    max-width: 360px !important;
}

#slider-square {
    width: 90%;
    margin: 0 auto;
}

pre {
    margin: 6px 0;
    background: #1E1E1E;
    padding: 10px;
    white-space: pre-wrap;
    word-wrap: break-word;
}
</style>
