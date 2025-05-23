<template>
  <div class="mnemonic-container">
    <h3>{{ title }}</h3>
    <div class="mnemonic-display" v-if="mnemonic">
      <div class="mnemonic-words">
        <span v-for="(word, index) in mnemonicArray" :key="index" class="mnemonic-word">
          {{ word }}
        </span>
      </div>
      <div class="mnemonic-info">
        <p>這組助記詞可用於恢復您的錢包，請妥善保管</p>
      </div>
    </div>
    <div class="mnemonic-actions">
      <button @click="generateMnemonic" class="generate-btn">
        {{ mnemonic ? '重新產生助記詞' : '產生新助記詞' }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import { ethers } from 'ethers'

// 定義組件屬性
interface Props {
  title?: string
  wordCount?: 12 | 15 | 18 | 21 | 24
}

const props = withDefaults(defineProps<Props>(), {
  title: '助記詞產生器',
  wordCount: 12
})

// 定義組件要傳出的事件
const emit = defineEmits<{
  (e: 'mnemonic-generated', mnemonic: string): void
}>()

// 響應式狀態
const mnemonic = ref('')

// 將助記詞轉換為數組以便顯示
const mnemonicArray = computed(() => {
  return mnemonic.value ? mnemonic.value.split(' ') : []
})

// 產生新助記詞
async function generateMnemonic() {
  try {
    // 根據選擇的字數設置熵的大小
    // 12 words = 16 bytes, 15 words = 20 bytes, 18 words = 24 bytes, 21 words = 28 bytes, 24 words = 32 bytes
    const entropyBytes = props.wordCount * 4 / 3
    
    // 使用 ethers.js 創建隨機熵，然後生成助記詞
    const entropy = ethers.randomBytes(entropyBytes)
    const generatedMnemonic = ethers.Mnemonic.fromEntropy(entropy)
    mnemonic.value = generatedMnemonic.phrase
    
    // 向父組件發送事件
    emit('mnemonic-generated', mnemonic.value)
  } catch (error) {
    console.error('產生助記詞時發生錯誤:', error)
  }
}
</script>

<style scoped>
.mnemonic-container {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  background-color: #f9f9f9;
  color: #333333;
}

.mnemonic-display {
  margin: 20px 0;
}

.mnemonic-words {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-bottom: 15px;
}

.mnemonic-word {
  background-color: #ffffff;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  padding: 8px 12px;
  font-family: monospace;
  font-size: 14px;
  color: #000000;
  font-weight: 500;
}

.mnemonic-info {
  font-size: 14px;
  color: #ff4d4f;
  margin-top: 10px;
}

.mnemonic-actions {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.generate-btn {
  background-color: #1890ff;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
}

.generate-btn:hover {
  background-color: #40a9ff;
}
</style>
