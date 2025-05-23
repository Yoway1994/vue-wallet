<template>
  <div class="balance-checker-container">
    <h3>{{ title }}</h3>
    
    <div v-if="!walletAddress" class="no-address-message">
      <p>請先生成錢包地址</p>
    </div>
    
    <div v-else class="balance-checker-content">
      <div class="balance-controls">
        <div class="address-display">
          <div class="key-label">錢包地址:</div>
          <div class="address-value">{{ walletAddress }}</div>
        </div>
        
        <div class="network-selector">
          <label for="network-select">區塊鏈網絡:</label>
          <select id="network-select" v-model="selectedNetwork" class="input-control">
            <option value="binance-mainnet">幣安智能鏈 (Mainnet)</option>
            <option value="binance-testnet">幣安智能鏈 (Testnet)</option>
          </select>
        </div>
        
        <button @click="checkBalance" class="check-btn" :disabled="isLoading">
          {{ isLoading ? '查詢中...' : '查詢餘額' }}
        </button>
      </div>

      <div v-if="isLoading" class="loading-indicator">
        <p>正在查詢餘額...</p>
      </div>
      
      <div v-else-if="errorMessage" class="error-message">
        <p>{{ errorMessage }}</p>
      </div>
      
      <div v-else-if="balance !== null" class="balance-info">
        <div class="balance-item">
          <div class="balance-label">BNB 餘額:</div>
          <div class="balance-value">{{ formattedBalance }} BNB</div>
        </div>
        <div class="balance-item">
          <div class="balance-label">最後更新:</div>
          <div class="balance-timestamp">{{ lastUpdated }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, defineProps } from 'vue'
import { ethers } from 'ethers'

// 定義組件屬性
interface Props {
  title?: string
  walletAddress?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: '資產餘額查詢',
  walletAddress: ''
})

// 響應式狀態
const selectedNetwork = ref('binance-mainnet')
const balance = ref<number | null>(null)
const isLoading = ref(false)
const errorMessage = ref('')
const lastUpdated = ref('')

// 計算格式化的餘額
const formattedBalance = computed(() => {
  if (balance.value === null) return '0'
  return parseFloat(balance.value.toFixed(8))
})

// 獲取網絡配置
function getNetworkConfig() {
  switch (selectedNetwork.value) {
    case 'binance-mainnet':
      return {
        name: 'Binance Smart Chain',
        rpcUrl: 'https://bsc-dataseed.binance.org/',
        symbol: 'BNB'
      }
    case 'binance-testnet':
      return {
        name: 'Binance Smart Chain Testnet',
        rpcUrl: 'https://data-seed-prebsc-1-s1.binance.org:8545/',
        symbol: 'BNB'
      }
    default:
      return {
        name: 'Binance Smart Chain',
        rpcUrl: 'https://bsc-dataseed.binance.org/',
        symbol: 'BNB'
      }
  }
}

// 設置時間戳
function updateTimestamp() {
  const now = new Date()
  lastUpdated.value = now.toLocaleString()
}

// 查詢餘額
async function checkBalance() {
  if (!props.walletAddress) {
    errorMessage.value = '需要錢包地址才能查詢餘額'
    return
  }

  try {
    isLoading.value = true
    errorMessage.value = ''
    
    // 獲取網絡配置
    const networkConfig = getNetworkConfig()
    
    // 使用 ethers.js 創建提供者
    const provider = new ethers.JsonRpcProvider(networkConfig.rpcUrl)
    
    // 查詢餘額
    const balanceWei = await provider.getBalance(props.walletAddress)
    
    // 從 wei 轉換為 ether 單位
    balance.value = parseFloat(ethers.formatEther(balanceWei))
    
    // 更新時間戳
    updateTimestamp()
    
  } catch (error) {
    console.error('查詢餘額時發生錯誤:', error)
    errorMessage.value = `查詢餘額失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
    balance.value = null
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
.balance-checker-container {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  background-color: #f9f9f9;
  color: #333333;
}

.no-address-message {
  padding: 15px;
  background-color: #fff2f0;
  border: 1px solid #ffccc7;
  border-radius: 4px;
  color: #f5222d;
  margin-bottom: 15px;
}

.balance-controls {
  margin-bottom: 20px;
}

.address-display {
  margin-bottom: 15px;
}

.key-label {
  font-weight: 600;
  margin-bottom: 5px;
  color: #333333;
}

.address-value {
  padding: 10px;
  background-color: #f5f5f5;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  word-break: break-all;
  color: #333333;
}

.network-selector {
  margin-bottom: 15px;
  display: flex;
  flex-direction: column;
}

.network-selector label {
  margin-bottom: 5px;
  font-weight: 500;
}

.input-control {
  padding: 8px 12px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 14px;
  width: 100%;
}

.check-btn {
  background-color: #1890ff;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
  width: 100%;
}

.check-btn:hover {
  background-color: #40a9ff;
}

.check-btn:disabled {
  background-color: #bfbfbf;
  cursor: not-allowed;
}

.loading-indicator {
  text-align: center;
  padding: 15px;
  color: #1890ff;
}

.error-message {
  padding: 15px;
  background-color: #fff2f0;
  border: 1px solid #ffccc7;
  border-radius: 4px;
  color: #f5222d;
}

.balance-info {
  padding: 15px;
  background-color: #f6ffed;
  border: 1px solid #b7eb8f;
  border-radius: 4px;
}

.balance-item {
  margin-bottom: 10px;
}

.balance-label {
  font-weight: 600;
  margin-bottom: 5px;
}

.balance-value {
  font-size: 24px;
  color: #52c41a;
  font-weight: 600;
}

.balance-timestamp {
  font-size: 12px;
  color: #8c8c8c;
}
</style>
