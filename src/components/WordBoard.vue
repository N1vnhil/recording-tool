<!-- WordBoard.vue -->
<template>
  <div>
    <!-- 用户名输入 -->
    <div v-if="!username" class="username-input">
      <label for="username">请输入用户名：</label>
      <input 
        type="text" 
        id="username" 
        v-model="usernameInput"
        @keyup.enter="setUsername"
        placeholder="输入用户名后按回车"
      >
      <button 
        @click="setUsername"
        :disabled="!usernameInput.trim()"
        class="confirm-button"
      >
        确认
      </button>
    </div>

    <!-- 主界面 -->
    <div v-else>
      <!-- 显示当前用户 -->
      <div class="current-user">
        当前用户：{{ username }}
        <button class="change-user-button" @click="changeUsername">更换用户</button>
      </div>

      <!-- 加载CSV文件按钮 -->
      <button @click="loadCsv" v-if="!csvData.length" class="start-button">Start</button>
  
      <!-- 当CSV数据加载完成后，显示上一个、下一个按钮以及当前字符 -->
      <div v-if="csvData.length && headers.length" class="navigation-container">
        <div class="navigation-controls">
          <button @click="prevCharacter" :disabled="currentIdx === 0">Previous</button>
          <button @click="nextCharacter" :disabled="currentIdx >= csvData.length - 1">Next</button>
        </div>
        <div class="character-display">
          我念"{{ currentCharacter }}"字
        </div>
        <div v-if="currentIdx < 5" class="tips">
          你可以用前五个字符熟悉录音流程。练习：{{ currentIdx + 1 }} / 5
        </div>
        <div v-else class="tips">
          测试进行中，共需录制270个汉字。
        </div>
        <div class="counter-display">
          已录制: {{ nextCounter }}/50
        </div>
  
        <!-- 传递 currentIdx 和 username 到 Recording 组件 -->
        <Recording 
          ref="recordingRef"
          :filename="getSavedFilename()"
        />
      </div>
    </div>
  </div>
</template>
  
<script setup>
import { ref, computed } from 'vue';
import Papa from 'papaparse';
import Recording from './Recording.vue';

const username = ref('');
const usernameInput = ref('');
const csvData = ref([]); // 存储CSV数据
const headers = ref([]); // 存储CSV头部信息
const currentIdx = ref(0); // 当前遍历到的索引
const recordingRef = ref(null); // 添加对 Recording 组件的引用
const nextCounter = ref(0); // 记录Next点击次数

// 设置用户名
function setUsername() {
  const trimmedUsername = usernameInput.value.trim();
  if (trimmedUsername) {
    username.value = trimmedUsername;
    usernameInput.value = '';
  }
}

// 更换用户
function changeUsername() {
  username.value = '';
  usernameInput.value = '';
  csvData.value = [];
  headers.value = [];
  currentIdx.value = 0;
  nextCounter.value = 0;
}

const currentCharacter = computed(() => {
  const characterIdx = headers.value.indexOf('character');
  return csvData.value[currentIdx.value][characterIdx];
});

const currentRep  = computed(() => {
  const characterIdx = headers.value.indexOf('Rep');
  return csvData.value[currentIdx.value][characterIdx];
});

function getSavedFilename() {
  return username.value + "_" + currentCharacter.value + "_" + currentRep.value + ".wav";
}

function loadCsv() {
  const url = new URL('../../public/20250411_YY.csv', import.meta.url).href;
  fetch(url)
    .then(response => response.text())
    .then(data => {
      Papa.parse(data, {
        complete: results => {
          if (results.data.length > 0) {
            csvData.value = results.data.slice(1); // 跳过标题行
            headers.value = results.data[0]; // 设置标题行
          }
        }
      });
    })
    .catch(error => console.error('Error loading CSV file:', error));
}

function nextCharacter() {
  // 检查是否达到50次
  if (nextCounter.value >= 50) {
    const shouldContinue = confirm('您已经录制了50个字符，是否继续？');
    if (!shouldContinue) {
      return;
    }
    nextCounter.value = 0;
  }

  if (recordingRef.value && recordingRef.value.isRecording) {
    console.log("当前：" + currentIdx.value);
    recordingRef.value.stopAndSaveRecording(getSavedFilename());
  } 
  
  currentIdx.value++;
  nextCounter.value++;

  if (recordingRef.value) {
    recordingRef.value.startRecording();
  }
}

function prevCharacter() {
  // 如果正在录音，先停止并保存
  if (recordingRef.value && recordingRef.value.isRecording) {
    const t = getSavedFilename();
    recordingRef.value.stopAndSaveRecording(t);
  } 

  currentIdx.value--;

  if (recordingRef.value) {
    recordingRef.value.startRecording();
  }
}
</script>

<style scoped>
/* 新增计数器样式 */
.counter-display {
  margin: 0.5rem 0;
  font-size: 1rem;
  color: #666;
}

.username-input {
  margin-bottom: 2rem;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.username-input input {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 16px;
  min-width: 200px;
}

.confirm-button {
  background-color: #2196F3;
}

.current-user {
  margin-bottom: 1rem;
  padding: 0.5rem;
  background-color: #f5f5f5;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.change-user-button {
  background-color: #757575;
  padding: 5px 10px;
  font-size: 14px;
}

.start-button {
  background-color: #4CAF50;
  margin: 1rem 0;
}

.navigation-container {
  margin-top: 1rem;
}

.navigation-controls {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
}

.navigation-controls button {
  background-color: #2196F3;
  padding: 8px 16px;
}

.navigation-controls button:disabled {
  background-color: #ccc;
}

.character-display {
  font-size: xx-large;
  margin: 1rem 0;
  padding: 1rem;
  background-color: #f5f5f5;
  border-radius: 4px;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  border: none;
  border-radius: 4px;
  color: white;
  transition: background-color 0.3s;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

button:hover:not(:disabled) {
  opacity: 0.9;
}

.tips {
  font-size: large;
  color: red;
}
</style>