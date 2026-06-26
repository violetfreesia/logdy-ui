<script setup lang="ts">
import { ref } from 'vue';
import { Layout } from '../config';
import { client } from '../api';

const props = defineProps<{
    layout: Layout
}>()

const emit = defineEmits<{
    (e: 'layout-loaded', layout: Layout): void,
}>()

const copied = ref<boolean>(false)
const saved = ref<string | null>(null)
const savedError = ref<boolean>(false)
const imported = ref<string>("")
const importResult = ref<string>("")

const exportLayout = () => {
    return JSON.stringify(props.layout.toObj(), null, '\t')
}

const copy = () => {
    navigator.clipboard.writeText(exportLayout())
    copied.value = true
}

const save = async() => {
    let res = await client.configSave(exportLayout())
    if(res.status !== 200){
        savedError.value = true
        return
    }
    saved.value = res.json!.location || "unknown"
}

function downloadFile(file: File) {
    // Create a link and set the URL using `createObjectURL`
    const link = document.createElement("a");
    link.style.display = "none";
    link.href = URL.createObjectURL(file);
    link.download = file.name;

    // It needs to be added to the DOM so it can be clicked
    document.body.appendChild(link);
    link.click();

    // To make this work on Firefox we need to wait
    // a little while before removing it.
    setTimeout(() => {
        URL.revokeObjectURL(link.href);
        link.parentNode!.removeChild(link);
    }, 0);
}


const download = () => {
    // Dynamically create a File
    const myFile = new File([exportLayout()], "logdy.json");

    // Download it using our function
    downloadFile(myFile);
}
function importFromFile() {
    let input = document.createElement('input') as HTMLInputElement;
    input.type = 'file';
    input.accept = '.json';
    input.onchange = _ => {
        // you can use this method to get file and perform respective operations
        let file = input.files;
        const reader = new FileReader();
        reader.onload = function () {
            const contents = reader.result;
            importLayout(contents as string)
            // Process the contents of the file
        };
        reader.readAsText(file!.item(0)!)
    };
    input.click();

}

const importFromJson = () => {
    importLayout(imported.value)
}

const importLayout = (data: string) => {
    let l = new Layout('main', { leftColWidth: 300, drawerColWidth: 900, maxMessages: 1000, middlewares: [], entriesOrder: "desc" })
    l.loadFromObj(JSON.parse(data))
    emit('layout-loaded', l)
}

</script>
<template>
    <div class="form-panel import-panel">
        你可以在这里导出和导入 UI 设置。仅导出布局（列、分面）和设置（中间件），不包含日志消息。
        <h2>导出</h2>
        <div class="button-row">
            <button class="btn" @click="copy">复制到剪贴板</button>
            <button class="btn" @click="download">下载</button>
            <button class="btn" @click="save">保存到本地配置文件</button>
        </div>
        <div class="alert alert-info" v-if="copied">已复制</div>
        <div class="alert alert-info" v-if="saved">已保存到：{{saved}}</div>
        <div class="alert alert-danger" v-if="savedError">保存到文件时出错</div>
        <hr />
        <h2>导入</h2>
        <textarea rows="7" class="input import-json" placeholder="请在此粘贴 JSON" v-model="imported"></textarea>

        <div class="button-row">
            <button class="btn" @click="importFromJson" :disabled="imported.length === 0">导入</button>
            <button class="btn" @click="importFromFile">从文件导入</button>
        </div>
        <div class="">{{ importResult }}</div>
    </div>
</template>

<style scoped>
.import-json {
    width: 100%;
    max-height: 240px;
    overflow: auto;
    font-family: var(--code-font);
}
</style>
