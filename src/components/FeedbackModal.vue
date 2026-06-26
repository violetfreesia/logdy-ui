<script setup lang="ts">
import { onMounted, ref } from 'vue';

const email = ref<string>("")
const content = ref<string>("")
const loading = ref<boolean>(false)
const success = ref<boolean>(false)

onMounted(() => {

    //@ts-expect-error
    document.JSONP = (function (document) {
        var requests = 0,
            callbacks = {};

        return {
            /**
             * makes a JSONP request
             *
             * @param {String} src
             * @param {Object} data
             * @param {Function} callback
             */
            get: function (src: string, data: any, callback: any) {
                // check if data was passed
                if (!arguments[2]) {
                    callback = arguments[1];
                    data = {};
                }

                // determine if there already are params
                src += (src.indexOf('?') + 1 ? '&' : '?');

                var head = document.getElementsByTagName('head')[0],
                    script = document.createElement('script'),
                    params = [],
                    param;

                // increment the requests
                requests++;

                // create external callback name
                // data.callback = 'JSONP.callbacks.request_' + requestId;
                data.callback = 'cb'

                // set callback function
                // callbacks['request_' + requestId] = function (data) {
                //@ts-expect-error
                callbacks['cb'] = function (data: any) {
                    // clean up
                    head.removeChild(script);
                    //@ts-expect-error
                    delete callbacks['cb'];

                    // fire callback
                    callback(data);
                };

                // traverse data
                for (param in data) {
                    params.push(param + '=' + encodeURIComponent(data[param]));
                }

                // generate params
                src += params.join('&');

                // set script attributes
                script.type = 'text/javascript';
                script.src = src;

                // add to the DOM
                head.appendChild(script);
            },

            /**
             * keeps a public reference of the callbacks object
             */
            callbacks: callbacks
        };
    })(document);

})

const submit = async () => {

    loading.value = true

    await new Promise(resolve => {
        //@ts-expect-error
        document.JSONP.get('https://eoutbn4ig0dwes4.m.pipedream.net', { email: email.value, content: content.value }, function (data) {
            resolve(data)
        });
    })

    loading.value = false
    success.value = true
    email.value = ''
    content.value = ''
}
</script>

<template>
    <div class="form-panel feedback-panel">

        你可以填写下面的表单留下反馈或报告错误。
        消息会直接提交给 Logdy 团队。
        也可以考虑创建一个 <a href="https://github.com/logdyhq/logdy-core/issues/new" target="_blank">GitHub Issue</a>。

        <label class="form-label">邮箱（可选）：</label>
        <input type="text" class="input" name="email" v-model="email">

        <textarea v-model="content" placeholder="在这里填写反馈或错误描述" rows="5" class="input"
            ></textarea>

        <div class="button-row">
            <button @click="submit" :disabled="loading || content.length === 0">提交</button>
        </div>

        <div class="alert alert-success" v-if="success">消息已发送！</div>
    </div>
</template>

<style scoped lang="scss">
.feedback-panel textarea {
    width: 100%;
    max-height: 220px;
    overflow: auto;
}
</style>
