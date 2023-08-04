<template>
    <div class="common-layout">
        <el-container>
            <el-header>通用gpt</el-header>
            <el-container>
                <el-aside width="200px"><el-row justify="start">
                        <el-col :span="6">
                            <el-text type="primary">模型 </el-text>
                        </el-col>
                        <el-col :span="14">
                            <el-select v-model="state.model" class="m-2" placeholder="Select">
                                <el-option v-for="(item, index) in state.models" :key="index" :label="item.label"
                                    :value="item.value" />
                            </el-select>
                        </el-col>
                    </el-row>
                    <el-row justify="start">
                        <el-col :span=6>
                            <el-text>限制 </el-text>
                        </el-col>
                        <el-col :span="8" v-for="(item, index) in limits" :key="index">
                            <el-text type="warning">{{ item }}</el-text>
                        </el-col>
                    </el-row>
                    <el-row>
                        <el-col :span="8" :offset="1">
                            <el-tooltip content="设置获取到的apikey" placement="bottom" effect="light">
                                <el-button @click="state.dialogVisible = true" type="success" size="small"
                                    text="true">设置apiKey</el-button>
                            </el-tooltip>
                        </el-col>
                        <el-col :span="8" :offset="1">
                            <el-popconfirm title="确定要清除apiKey吗？">
                                <template #reference>
                                    <el-button @click="removeApiKey" type="danger" size="small"
                                        text="true">清除apiKey</el-button>
                                </template>
                            </el-popconfirm>
                        </el-col>
                    </el-row>
                    <el-row>
                        <el-col :span="8" :offset="1">
                            <el-tooltip content="设置要请求的后端的接口域名" placement="bottom" effect="light">
                                <el-button @click="state.urlDialogVisible = true" type="success" size="small"
                                    text="true">设置url前缀</el-button>
                            </el-tooltip>
                        </el-col>
                        <el-col :span="8" :offset="1">
                            <el-popconfirm title="确定要清除url前缀吗？">
                                <template #reference>
                                    <el-button @click="removeApiUrl" type="danger" size="small"
                                        text="true">清除url前缀</el-button>
                                </template>
                            </el-popconfirm>
                        </el-col>
                    </el-row>
                </el-aside>
                <el-main>
                    <el-card>
                        <el-scrollbar height="60vh">
                            <div ref="chatDialog">
                                <el-row v-for="(message, index) in state.chatHistory" :key="index">
                                    <el-divider />
                                    <el-col :span="2">
                                        <el-text tag="b">{{ message.role }} </el-text>
                                    </el-col>
                                    <el-col :span="20" :offset="1" style="background-color: aliceblue;">
                                        <v-md-preview style="text-align: left;" :text="message.content"></v-md-preview>
                                    </el-col>

                                </el-row>
                            </div>
                            <div class="back-to-bottom" @click="scrollToBottom">
                                <el-button type="primary" :icon="Bottom" circle></el-button>
                            </div>
                        </el-scrollbar>
                        <el-progress :percentage="100" :show-text="false" v-show="state.loading" :indeterminate="true"
                            :duration="5" />
                        <el-divider />
                        <el-row>
                            <el-col :span="2">
                                <el-text tag="b"></el-text>
                            </el-col>
                            <el-col :span="17">
                                <el-input type="textarea" :autosize="{ minRows: 4, maxRows: 15 }" placeholder="请输入您的问题..." v-model="state.content" clearable >
                                </el-input>
                            </el-col>
                            <el-col :span="2">
                                <el-button @click="chatCompletion">发送</el-button>
                            </el-col>
                            <el-col :span="2">
                                <el-popconfirm title="确定要清空聊天记录吗？">
                                    <template #reference>
                                        <el-button @click="removeHistoryChat">清空聊天记录</el-button>
                                    </template>
                                </el-popconfirm>
                            </el-col>
                        </el-row>
                    </el-card>
                </el-main>
            </el-container>
        </el-container>
    </div>

    <el-dialog v-model="state.dialogVisible" title="设置apiKey" width="30%">
        <el-row>
            <el-col>
                <el-input v-model="state.apiKey"></el-input>
            </el-col>
        </el-row>
        <template #footer>
            <span class="dialog-footer">
                <el-button @click="state.dialogVisible = false">关闭</el-button>
                <el-button type="primary" @click="setApiKey">
                    确认
                </el-button>
            </span>
        </template>
    </el-dialog>
    <el-dialog v-model="state.urlDialogVisible" title="设置apiKey" width="30%">
        <el-row>
            <el-col>
                <el-input v-model="state.apiUrl"></el-input>
            </el-col>
        </el-row>
        <template #footer>
            <span class="dialog-footer">
                <el-button @click="state.urlDialogVisible = false">关闭</el-button>
                <el-button type="primary" @click="setApiUrl">
                    确认
                </el-button>
            </span>
        </template>
    </el-dialog>
</template>
<script lang="ts" setup>

import axios from 'axios';
import { onMounted, reactive, computed, ref, nextTick } from 'vue';
import { ElMessage } from 'element-plus';
import { API_URL } from '../../utils/env/constants';
import {
    Bottom
} from '@element-plus/icons-vue';


const state = reactive({
    content: '',
    models: [] as { label: string; value: string }[],
    model: 'gpt-4',
    modelMaps: {} as Record<string, { endpoints: string; limits: string[] }>,
    apiKey: '',
    chatHistory: [] as { role: string; content: string }[],
    dialogVisible: false,
    loading: false,
    urlDialogVisible: false,
    apiUrl: API_URL

});
const chatDialog = ref<HTMLElement | null>(null);

onMounted(() => {
    getModels();
    // 从本地存储中读取聊天记录
    const storedChatHistory = localStorage.getItem('chatHistory');
    if (storedChatHistory) {
        state.chatHistory = JSON.parse(storedChatHistory);
    }
    const storedApiKey = localStorage.getItem('apiKey');
    if (storedApiKey) {
        state.apiKey = storedApiKey;
    }
    const storedApiUrl = localStorage.getItem('apiUrl');
    if (storedApiUrl) {
        state.apiUrl = storedApiUrl;
    }
    scrollToBottom();
})

/**
 * 监听模型是否改变
 */
const limits = computed(() => {
    if (state.modelMaps[state.model]) {
        return state.modelMaps[state.model].limits
    }
}
);

const removeHistoryChat = () => {
    localStorage.removeItem("chatHistory");
}

const chatCompletion = async () => {
    try {
        let returnMsg = '';
        state.loading = true;
        // 在发送消息时，将消息添加到聊天历史中
        state.chatHistory.push({ role: 'user', content: state.content });
        scrollToBottom();
        axios({
            method: 'post',
            url: `${state.apiUrl}${state.modelMaps[state.model].endpoints}`,
            data: {
                model: state.model,
                messages: state.chatHistory,
                stream: true,
                allow_fallback: true
            },
            headers: {
                'Content-Type': 'application/json',
                Authorization: `Bearer ${state.apiKey}`
            },
            responseType: 'stream'
        }).then(async response => {
            // Create a new Blob object
            const blob = new Blob([response.data], { type: 'text/event-stream' });

            // Create a new ReadableStream
            const stream = blob.stream();

            // Create a new TextDecoder
            const decoder = new TextDecoder();

            let chunkTextBuffer = ''; // Buffer to accumulate chunk text

            // Function to process the chunks of data from the stream
            const processChunk = async (chunk: any) => {
                // Convert ArrayBuffer to string
                const chunkText = decoder.decode(chunk.value);

                // Combine the current chunkText with previous buffered text
                chunkTextBuffer += chunkText;

                // Process each line in the chunkTextBuffer
                const lines = chunkTextBuffer.split('\n');
                for (let i = 0; i < lines.length - 1; i++) {
                    const line = lines[i];
                    try {
                        if (line.startsWith('data: ')) {
                            const jsonData = JSON.parse(line.substring(6));
                            if (jsonData.choices && jsonData.choices[0] && jsonData.choices[0].delta && jsonData.choices[0].delta.content) {
                                returnMsg += jsonData.choices[0].delta.content;
                            }
                        }
                    } catch (e) {
                        //returnMsg += '---over---';
                        return;
                    }
                }

                // Update the chunkTextBuffer with any remaining text
                chunkTextBuffer = lines[lines.length - 1];
            };

            // Continuously read chunks from the stream and process them
            const reader = stream.getReader();

            //异步方法
            const readChunks = async () => {
                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;
                    await processChunk({ done, value });
                }

                // 在接收到回复时，将回复添加到聊天历史中
                state.chatHistory.push({ role: 'assistant', content: returnMsg });
                // 将聊天历史保存到本地存储中
                localStorage.setItem('chatHistory', JSON.stringify(state.chatHistory));
                state.loading = false;
                scrollToBottom();
            }
            readChunks();
        }).catch(error => {
            console.log(error)
            ElMessage({
                message: "请求出错,返回数据:" + error.response.data,
                type: 'warning',
                duration: 5000
            })
            state.loading = false;
            scrollToBottom();
        });
    } catch (error) {
        console.log(error)
    }
}


const getModels = async () => {
    try {
        const response = await axios.get(
            `https://chimeragpt.adventblocks.cc/api/v1/models`);

        for (const item of response.data.data) {
            state.models.push({
                label: item.id,
                value: item.id
            })
            state.modelMaps[item.id] = {
                endpoints: item.endpoints[0],
                limits: item.limits
            };
        }
    } catch (error) {
        console.error('Error:', error);
    }

}

const setApiKey = () => {
    localStorage.setItem('apiKey', state.apiKey);
    state.dialogVisible = false
}

const removeApiKey = () => {
    state.apiKey = '';
    localStorage.removeItem('apiKey');
}

const removeApiUrl = () => {
    state.apiUrl = '';
    localStorage.removeItem('apiUrl');
}

const setApiUrl = () => {
    localStorage.setItem('apiUrl', state.apiUrl);
    state.urlDialogVisible = false
}



const scrollToBottom = async () => {
    await nextTick();
    if (chatDialog.value) {
        chatDialog.value.scrollIntoView(false);
    }
};

</script>

<style scoped>
.el-row {
    margin-bottom: 20px;
}

.back-to-bottom {
    right: 20px;
    bottom: 20px;
    position: absolute;
}
</style>