<script setup lang="ts">
import { onMounted, ref } from 'vue';
import { useMainStore } from '../store';
import { client } from "../api"

const pass = ref<string>("")
const input = ref<HTMLInputElement>()
const loading = ref<boolean>()
const msg = ref<string>()
const remember = ref<boolean>(true)

const emit = defineEmits<{
    (e: 'success'): void,
}>()

onMounted(() => {
    input.value?.focus()
})

const submit = async () => {
    msg.value = ""
    loading.value = true
    let res = await client.sendGet("check-pass?password=" + pass.value)

    loading.value = false
    if (res.status == 200) {
        useMainStore().setPassword(pass.value.toString(), remember.value)
        emit('success')
        return
    }

    msg.value = "密码错误，请重试"
}

</script>
<template>
    <div class="auth-panel form-panel">
        <div class="auth-title">身份验证</div>
        <div class="auth-copy">访问 Logdy 需要输入密码。<a href="https://logdy.dev">什么是 Logdy？</a></div>
        <div>
            <input class="input password-input" ref="input" v-model="pass" type="password" @keyup.enter="submit" />
        </div>
        <div class="auth-loading" v-if="loading">加载中...</div>
        <div class="form-row">
            <input type="checkbox" v-model="remember" id="ch" /> <label for="ch">记住密码</label>
        </div>
        <div class="err" v-if="msg">
            {{ msg }}
        </div>
        <div class="button-row">
            <button :disabled="pass?.length === 0" class="btn" @click="submit">提交</button>
        </div>
    </div>
</template>

<style scoped lang="scss">
.auth-title {
    font-size: 20px;
    font-weight: 800;
}

.auth-copy,
.auth-loading {
    color: var(--font-muted);
}

.password-input {
    width: min(420px, 100%);
    font-family: var(--code-font);
}

.err {
    padding: 10px;
    border: 1px solid color-mix(in srgb, var(--danger) 55%, transparent);
    border-radius: var(--radius-md);
    background: var(--danger-bg);
    color: var(--danger);
}
</style>
