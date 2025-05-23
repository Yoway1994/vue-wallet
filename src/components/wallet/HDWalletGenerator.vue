<template>
  <div class="hdwallet-container">
    <h3>{{ title }}</h3>
    
    <div v-if="!mnemonic" class="no-mnemonic-message">
      <p>請先產生助記詞</p>
    </div>
    
    <div v-else class="hdwallet-content">
      <div class="hdwallet-controls">
        <div class="input-group">
          <label for="derivation-path">衍生路徑:</label>
          <input 
            id="derivation-path" 
            v-model="derivationPath" 
            type="text" 
            class="input-control"
          />
        </div>
        <div class="input-group">
          <label for="account-index">賬戶索引:</label>
          <input 
            id="account-index" 
            v-model.number="accountIndex" 
            type="number" 
            min="0" 
            class="input-control"
          />
        </div>
        <button @click="generateWallet" class="generate-btn">
          產生錢包
        </button>
      </div>

      <div v-if="wallet" class="hdwallet-info">
        <div class="key-info-group">
          <div class="key-label">公鑰 (Public Key):</div>
          <div class="key-value">{{ wallet.publicKey }}</div>
        </div>
        <div class="key-info-group">
          <div class="key-label">私鑰 (Private Key):</div>
          <div class="key-value private-key">
            <span v-if="showPrivateKey">{{ wallet.privateKey }}</span>
            <span v-else>*************************</span>
            <button @click="togglePrivateKey" class="toggle-btn">
              {{ showPrivateKey ? '隱藏' : '顯示' }}
            </button>
          </div>
        </div>
        <div class="key-info-group">
          <div class="key-label">錢包地址 (Address):</div>
          <div class="key-value">{{ wallet.address }}</div>
        </div>
        <div class="key-info-group">
          <div class="key-label">衍生路徑 (Derivation Path):</div>
          <div class="key-value">{{ wallet.path }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, defineProps, defineEmits } from 'vue'
import { ethers } from 'ethers'

// 定義組件屬性
interface Props {
  title?: string
  mnemonic?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: 'HD 錢包產生器',
  mnemonic: ''
})

// 定義組件要傳出的事件
const emit = defineEmits<{
  (e: 'wallet-generated', wallet: WalletInfo): void
}>()

interface WalletInfo {
  address: string
  publicKey: string
  privateKey: string
  path: string
  index: number
}

// 響應式狀態
const derivationPath = ref("m/44'/60'/0'/0")
const accountIndex = ref(0)
const wallet = ref<WalletInfo | null>(null)
const showPrivateKey = ref(false)

// 監聽助記詞變化，重置錢包信息
watch(() => props.mnemonic, () => {
  wallet.value = null
  showPrivateKey.value = false
})

// 切換顯示/隱藏私鑰
function togglePrivateKey() {
  showPrivateKey.value = !showPrivateKey.value
}

// 產生 HD 錢包
async function generateWallet() {
  if (!props.mnemonic) {
    console.error('需要助記詞才能產生 HD 錢包')
    return
  }

  try {
    // 從助記詞創建 HD 節點
    const hdNode = ethers.HDNodeWallet.fromPhrase(
      props.mnemonic,
      undefined, // 密碼，暫不使用
      derivationPath.value // 衍生路徑
    )
    
    // 從 HD 節點衍生指定索引的錢包
    const derivedWallet = hdNode.deriveChild(accountIndex.value)
    
    // 設置錢包信息
    const walletInfo: WalletInfo = {
      address: derivedWallet.address,
      publicKey: derivedWallet.publicKey,
      privateKey: derivedWallet.privateKey,
      path: `${derivationPath.value}/${accountIndex.value}`,
      index: accountIndex.value
    }
    
    wallet.value = walletInfo
    
    // 向父組件發送事件
    emit('wallet-generated', walletInfo)
    
  } catch (error) {
    console.error('產生 HD 錢包時發生錯誤:', error)
  }
}
</script>

<style scoped>
.hdwallet-container {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  background-color: #f9f9f9;
  color: #333333;
}

.no-mnemonic-message {
  padding: 15px;
  background-color: #fff2f0;
  border: 1px solid #ffccc7;
  border-radius: 4px;
  color: #f5222d;
  margin-bottom: 15px;
}

.hdwallet-controls {
  margin-bottom: 20px;
}

.input-group {
  margin-bottom: 15px;
  display: flex;
  flex-direction: column;
}

.input-group label {
  margin-bottom: 5px;
  font-weight: 500;
}

.input-control {
  padding: 8px 12px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  width: 100%;
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

.hdwallet-info {
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  padding: 15px;
  background-color: white;
}

.key-info-group {
  margin-bottom: 15px;
}

.key-label {
  font-weight: 600;
  margin-bottom: 5px;
  color: #333333;
}

.key-value {
  padding: 10px;
  background-color: #f5f5f5;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  word-break: break-all;
  color: #333333;
}

.private-key {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.toggle-btn {
  background-color: #f0f0f0;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  padding: 4px 8px;
  cursor: pointer;
  font-size: 12px;
  margin-left: 10px;
  flex-shrink: 0;
}
</style>
