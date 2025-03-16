<template>
  <div>
    <!-- 设备选择 -->
    <div class="device-selector">
      <label for="audioDevice">录音设备：</label>
      <select 
        id="audioDevice" 
        v-model="selectedDevice" 
        @change="handleDeviceChange"
        :disabled="isRecording"
      >
        <option v-for="device in audioDevices" :key="device.deviceId" :value="device.deviceId">
          {{ device.label || `麦克风 ${device.deviceId.slice(0, 8)}...` }}
        </option>
      </select>
    </div>

    <!-- 录音控制 -->
    <button 
      @click="toggleRecording" 
      :class="{ recording: isRecording }"
      :disabled="!isMediaRecorderReady"
    >
      {{ isRecording ? '停止录音' : '开始录音' }}
    </button>

    <!-- 音频播放器 -->
    <audio v-if="audioUrl" controls class="audio-player">
      <source :src="audioUrl" type="audio/wav">
      您的浏览器不支持音频播放
    </audio>

    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const props = defineProps({
  currentIdx: {
    type: Number,
    required: true,
    default: 0
  },
  username: {
    type: String,
    required: true
  }
});

const audioDevices = ref([]);
const selectedDevice = ref('');
let mediaRecorder = null;
let mediaStream = null;
let audioChunks = [];

const isRecording = ref(false);
const audioUrl = ref(null);
const error = ref(null);
const isMediaRecorderReady = ref(false);

// 获取录音设备列表
async function getAudioDevices() {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices();
    audioDevices.value = devices.filter(device => device.kind === 'audioinput');
    
    if (audioDevices.value.length > 0 && !selectedDevice.value) {
      selectedDevice.value = audioDevices.value[0].deviceId;
      await initializeMediaRecorder();
    }
  } catch (err) {
    error.value = '获取录音设备列表失败';
    console.error('获取设备列表失败:', err);
  }
}

async function handleDeviceChange() {
  if (mediaStream) {
    mediaStream.getTracks().forEach(track => track.stop());
  }
  await initializeMediaRecorder();
}

function convertToWav(audioBuffer) {
  const numberOfChannels = 1;
  const sampleRate = 16000; // 使用16kHz采样率
  const bitsPerSample = 16;
  
  const buffer = new ArrayBuffer(44 + audioBuffer.length);
  const view = new DataView(buffer);
  
  writeString(view, 0, 'RIFF');
  view.setUint32(4, 36 + audioBuffer.length, true);
  writeString(view, 8, 'WAVE');
  
  writeString(view, 12, 'fmt ');
  view.setUint32(16, 16, true);
  view.setUint16(20, 1, true);
  view.setUint16(22, numberOfChannels, true);
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, sampleRate * numberOfChannels * (bitsPerSample / 8), true);
  view.setUint16(32, numberOfChannels * (bitsPerSample / 8), true);
  view.setUint16(34, bitsPerSample, true);
  
  writeString(view, 36, 'data');
  view.setUint32(40, audioBuffer.length, true);
  
  const offset = 44;
  for (let i = 0; i < audioBuffer.length; i++) {
    view.setUint8(offset + i, audioBuffer[i]);
  }
  
  return buffer;
}

function writeString(view, offset, string) {
  for (let i = 0; i < string.length; i++) {
    view.setUint8(offset + i, string.charCodeAt(i));
  }
}

// 重采样函数
function resampleAudio(audioBuffer, targetSampleRate) {
  const sourceSampleRate = audioBuffer.sampleRate;
  const sourceLength = audioBuffer.length;
  const targetLength = Math.round(sourceLength * targetSampleRate / sourceSampleRate);
  const outputBuffer = new Float32Array(targetLength);
  
  for (let i = 0; i < targetLength; i++) {
    const sourceIndex = i * sourceSampleRate / targetSampleRate;
    const index1 = Math.floor(sourceIndex);
    const index2 = Math.min(index1 + 1, sourceLength - 1);
    const fraction = sourceIndex - index1;
    
    // 线性插值
    outputBuffer[i] = (1 - fraction) * audioBuffer.getChannelData(0)[index1] +
                      fraction * audioBuffer.getChannelData(0)[index2];
  }
  
  return outputBuffer;
}

// 自动下载功能
function downloadBlob(blob, filename) {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.style.display = 'none';
  a.href = url;
  a.download = `${props.username}_${filename}`;
  document.body.appendChild(a);
  a.click();
  setTimeout(() => {
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }, 100);
}

async function initializeMediaRecorder() {
  try {
    const constraints = {
      audio: {
        deviceId: selectedDevice.value ? { exact: selectedDevice.value } : undefined,
        channelCount: 1,
        sampleRate: 16000, // 使用16kHz采样率
        sampleSize: 16
      }
    };

    mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
    mediaRecorder = new MediaRecorder(mediaStream, {
      mimeType: 'audio/webm'
    });
    
    mediaRecorder.ondataavailable = event => {
      audioChunks.push(event.data);
    };

    mediaRecorder.onstop = async () => {
      try {
        const webmBlob = new Blob(audioChunks, { type: 'audio/webm' });
        const arrayBuffer = await webmBlob.arrayBuffer();
        const audioContext = new AudioContext();
        const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
        
        // 重采样到16kHz
        const resampledData = resampleAudio(audioBuffer, 16000);
        
        // 转换为16位整数
        const samples = new Int16Array(resampledData.length);
        for (let i = 0; i < resampledData.length; i++) {
          samples[i] = Math.max(-1, Math.min(1, resampledData[i])) * 32767;
        }
        
        const wavBuffer = convertToWav(new Uint8Array(samples.buffer));
        const wavBlob = new Blob([wavBuffer], { type: 'audio/wav' });
        
        if (audioUrl.value) {
          URL.revokeObjectURL(audioUrl.value);
        }
        audioUrl.value = URL.createObjectURL(wavBlob);

        // 自动下载
        downloadBlob(wavBlob, `recording_${props.currentIdx}.wav`);
        
        audioChunks = [];
      } catch (err) {
        error.value = '处理录音数据时出错';
        console.error('处理录音数据失败:', err);
      }
    };

    isMediaRecorderReady.value = true;
    error.value = null;
  } catch (err) {
    error.value = '无法访问麦克风';
    console.error('获取媒体设备失败:', err);
    isMediaRecorderReady.value = false;
  }
}

function toggleRecording() {
  if (!isMediaRecorderReady.value) {
    error.value = '录音器未就绪';
    return;
  }

  try {
    if (isRecording.value) {
      stopRecording();
    } else {
      startRecording();
    }
  } catch (err) {
    error.value = '录音操作失败';
    console.error('录音操作失败:', err);
  }
}

function startRecording() {
  if (mediaRecorder.state === 'inactive') {
    audioChunks = [];
    audioUrl.value = null;
    isRecording.value = true;
    mediaRecorder.start();
    error.value = null;
  }
}

function stopRecording() {
  if (mediaRecorder.state === 'recording') {
    isRecording.value = false;
    mediaRecorder.stop();
  }
}

navigator.mediaDevices.addEventListener('devicechange', getAudioDevices);

onMounted(() => {
  getAudioDevices();
});

onUnmounted(() => {
  if (mediaStream) {
    mediaStream.getTracks().forEach(track => track.stop());
  }
  if (audioUrl.value) {
    URL.revokeObjectURL(audioUrl.value);
  }
  navigator.mediaDevices.removeEventListener('devicechange', getAudioDevices);
});
</script>

<style scoped>
.device-selector {
  margin-bottom: 1rem;
}

select {
  padding: 0.5rem;
  margin-left: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  min-width: 200px;
}

select:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
  border: none;
  border-radius: 4px;
  background-color: #4CAF50;
  color: white;
  margin-bottom: 1rem;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.recording {
  background-color: #f44336;
}

.audio-player {
  display: block;
  width: 100%;
  max-width: 500px;
  margin: 1rem 0;
}

.error {
  color: #f44336;
  margin-top: 10px;
}
</style>