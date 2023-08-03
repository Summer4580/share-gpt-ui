<template>
    <el-row>
        <el-text>模型 </el-text>
        <el-select v-model="model" class="m-2" placeholder="Select">
            <el-option v-for="item in models" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
    </el-row>
    <el-row>
        <el-text>限制 </el-text>
        <el-text></el-text>
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

const content = ref("");
const gptContent = ref([]);
const models = ref([]);
const model = ref("gpt-3.5-turbo")
const  modelMaps = reactive({

});

onMounted(()=>{
    getModels();
})

const chatCompletion = async () => {
    try {
        const response = await axios.post(
            `https://chimeragpt.adventblocks.cc${modelMaps[model.value].endpoints}`,
            {
                model: model.value,
                messages: [
                    { role: 'user', content: content.value }
                ],
                stream: true,
                allow_fallback: true
            },
            {
                headers: {
                    'Content-Type': 'application/json',
                    Authorization: `Bearer J9InRwtewQRsojLqf4HP_zHuS5zOrLZG5qGMvYGIj1o`
                }
            }
        );

        for (const chunk of response.data.choices) {
            const content = chunk.delta.content;
            console.log(content)
            gptContent.push(content)
        }
    } catch (error) {
        console.error('Error:', error);
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