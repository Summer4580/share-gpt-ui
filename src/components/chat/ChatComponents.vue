<template>
    <el-row>
        <el-text>模型 </el-text>
        <el-select v-model="model" class="m-2" placeholder="Select">
            <el-option v-for="item in models" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
    </el-row>
    <el-row>
        <el-text>限制 </el-text>
    </el-row>
    <el-row>
        <el-text tag="b">内容</el-text>
        <el-input v-model="content">
        </el-input>
        <el-button @click="chatCompletion">发送</el-button>
    </el-row>
    <el-row>
        <el-text>{{ gptContent }}</el-text>
    </el-row>
</template>
<script lang="ts" setup>

import axios from 'axios';
import { ref,onMounted, reactive } from 'vue';
import { ElMessage } from 'element-plus'

const content = ref("");
const gptContent = ref('');
const models = ref([]);
const model = ref("gpt-3.5-turbo")
const  modelMaps = reactive({});

onMounted(()=>{
    getModels();
    
})

const chatCompletion = async () => {
    try {
        axios({
            method: 'post',
            url: `https://chimeragpt.adventblocks.cc${modelMaps[model.value].endpoints}`,
            data: {
                model: model.value,
                messages: [
                    { role: 'user', content: content.value }
                ],
                stream: true,
                allow_fallback: true
            },
            headers:  {
                    'Content-Type': 'application/json',
                    Authorization: `Bearer J9InRwtewQRsojLqf4HP_zHuS5zOrLZG5qGMvYGIj1o`
                },
            responseType: 'stream'
        }).then(async response => {
            console.log(response.data)
            // Create a new Blob object
            const blob = new Blob([response.data], { type: 'text/event-stream' });

            // Create a new ReadableStream
            const stream = blob.stream();

            // Create a new TextDecoder
            const decoder = new TextDecoder();

            // Function to process the chunks of data from the stream
            const processChunk = async (chunk) => {
            // Convert ArrayBuffer to string
            const chunkText = decoder.decode(chunk.value);

            // Process each line in the chunkText
            const lines = chunkText.split('\n');
            for (const line of lines) {
                if (line.startsWith('data: ')) {
                const jsonData = JSON.parse(line.substring(6));
                if (jsonData.choices && jsonData.choices[0] && jsonData.choices[0].delta && jsonData.choices[0].delta.content) {
                    gptContent.value += jsonData.choices[0].delta.content;
                }
                } else if (line === '[DONE]') {
                    // Data reception is complete
                    gptContent.value += 'Data reception complete';
                }
            }
            };

            // Continuously read chunks from the stream and process them
            const reader = stream.getReader();
            while (true) {
            const { done, value } = await reader.read();
            if (done) break;
            await processChunk({ done, value });
            }
        }).catch(error =>{
            ElMessage({
                message: error.response.data,
                type: 'warning',
                duration: 5000
            })
        }); 
    } catch (error) {
        console.log(error)
    }
}


const getModels = async ()=>{
    try {
        const response = await axios.get(
            `https://chimeragpt.adventblocks.cc/api/v1/models`);

        for (const item of response.data.data) {
            models.value.push({
                label: item.id,
                value: item.id
            })
            modelMaps[item.id]= {};
            modelMaps[item.id].endpoints = item.endpoints[0];
            modelMaps[item.id].limits = item.limits;
        }
        console.log(modelMaps)
    } catch (error) {
        console.error('Error:', error);
    }

}

</script>