<script setup lang="ts">
import { ref } from 'vue';
import { Row } from '../types';
import { Layout } from '../config';

const props = defineProps<{
    rows: Row[],
    visibleRows: Row[],
    layout: Layout
}>()

const onlyVisible = ref<boolean>(true)
const format = ref<"json" | "csv">("json")
const content = ref<"columns" | "full">("full")

function convertToCSV(arr: any[]) {
    const array = [Object.keys(arr[0])].concat(arr)

    return array.map(it => {
        return Object.values(it).map(s => `"${s.replace(/"/g, '\"')}"`).toString()
    })
}

const exportFile = () => {
    let rows = onlyVisible.value ? props.visibleRows : props.rows
    let contents = rows.map(r => {
        let obj: any = {}
        props.layout.columns.filter(c => !c.hidden).forEach((c, k) => {
            obj[c.name] = r.cells[k].text
        })

        if (content.value === 'full') {
            props.layout.columns.filter(c => c.hidden).forEach((c, k) => {
                obj[c.name] = r.fields[k].text
            })
        }
        return obj
    })
    let text = format.value === 'json' ? contents.map(c => JSON.stringify(c)) : convertToCSV(contents)
    const myFile = new File([text.join("\n")], "logdy-messages." + format.value);

    downloadFile(myFile)
}

function downloadFile(file: File) {
    const link = document.createElement("a");
    link.style.display = "none";
    link.href = URL.createObjectURL(file);
    link.download = file.name;

    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
        URL.revokeObjectURL(link.href);
        link.parentNode!.removeChild(link);
    }, 0);
}

</script>

<template>
    <div class="form-panel export-panel">
    <div>
        <input type="checkbox" v-model="onlyVisible" id="only-visible" /> <label for="only-visible">仅导出当前可见内容（已过滤）</label><br />
        <small>取消勾选后将导出全部消息</small>
    </div>
    <p>格式</p>
    <div>
        <input type="radio" v-model="format" value="json" id="format-json" /> <label for="format-json">JSON</label><br />
        <input type="radio" v-model="format" value="csv" id="format-csv" /> <label for="format-csv">CSV</label>
    </div>
    <p>内容</p>
    <div>
        <input type="radio" v-model="content" value="columns" id="content-columns" /> <label for="content-columns">仅列</label><br />
        <input type="radio" v-model="content" value="full" id="content-full" /> <label for="content-full">完整行</label>
    </div>
    <div class="button-row">
        <button class="btn" @click="exportFile">导出</button>
    </div>
    </div>
</template>

<style scoped lang="scss">
.export-panel p {
    margin: 4px 0;
    color: var(--font-muted);
    font-weight: 700;
}

small {
    color: var(--font-muted);
}
</style>
