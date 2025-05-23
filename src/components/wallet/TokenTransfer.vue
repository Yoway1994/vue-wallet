<template>
  <div class="token-transfer-container">
    <h3>{{ title }}</h3>
    
    <div v-if="!walletInfo" class="no-wallet-message">
      <p>請先生成 HD 錢包</p>
    </div>
    
    <div v-else class="token-transfer-content">
      <div class="wallet-info-display">
        <div class="info-item">
          <div class="info-label">發送地址:</div>
          <div class="info-value">{{ walletInfo.address }}</div>
        </div>
      </div>
      
      <div class="network-selector">
        <label for="transfer-network-select">網絡:</label>
        <select id="transfer-network-select" v-model="selectedNetwork" class="input-control">
          <option value="binance-mainnet">幣安智能鏈 (Mainnet)</option>
          <option value="binance-testnet">幣安智能鏈 (Testnet)</option>
        </select>
      </div>
      
      <div class="form-group">
        <div class="input-group">
          <label for="recipient-address">收款地址:</label>
          <input 
            id="recipient-address" 
            v-model="recipientAddress" 
            type="text" 
            class="input-control"
            placeholder="輸入有效的錢包地址"
          />
        </div>
        
        <div class="input-group">
          <label for="amount">金額 (BNB):</label>
          <input 
            id="amount" 
            v-model.number="amount" 
            type="number" 
            step="0.0001"
            min="0.0001"
            class="input-control"
            placeholder="輸入轉帳金額"
          />
        </div>
        
        <div class="button-group">
          <button @click="checkBalance" class="secondary-btn" :disabled="isLoading">
            檢查餘額
          </button>
          <button @click="sendTransaction" class="primary-btn" :disabled="isLoading || !isFormValid">
            {{ isLoading ? '處理中...' : '發送交易' }}
          </button>
        </div>
      </div>
      
      <div v-if="currentBalance !== null" class="balance-display">
        <p>當前餘額: <strong>{{ currentBalance }} BNB</strong></p>
      </div>
      
      <div v-if="isLoading" class="loading-indicator">
        <p>{{ loadingMessage }}</p>
      </div>
      
      <div v-if="errorMessage" class="error-message">
        <p>{{ errorMessage }}</p>
      </div>
      
      <div v-if="txHash" class="success-message">
        <p>交易已發送!</p>
        <div class="tx-hash-container">
          <div class="tx-hash-label">交易哈希:</div>
          <div class="tx-hash-value">{{ txHash }}</div>
          <a :href="txExplorerLink" target="_blank" class="explorer-link">查看交易</a>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, defineProps } from 'vue'
import { ethers } from 'ethers'

// 定義錢包信息接口
interface WalletInfo {
  address: string
  privateKey: string
  path?: string
}

// 定義組件屬性
interface Props {
  title?: string
  walletInfo?: WalletInfo | null
}

const props = withDefaults(defineProps<Props>(), {
  title: 'BNB 轉帳',
  walletInfo: null
})

// 響應式狀態
const selectedNetwork = ref('binance-testnet') // 預設使用測試網絡
const recipientAddress = ref('')
const amount = ref<number | null>(null)
const isLoading = ref(false)
const loadingMessage = ref('')
const errorMessage = ref('')
const txHash = ref('')
const currentBalance = ref<number | null>(null)

// 計算屬性
const isFormValid = computed(() => {
  return (
    recipientAddress.value &&
    ethers.isAddress(recipientAddress.value) &&
    amount.value !== null &&
    amount.value > 0 &&
    (currentBalance.value === null || amount.value <= currentBalance.value)
  )
})

const txExplorerLink = computed(() => {
  if (!txHash.value) return '#'
  
  const baseUrl = selectedNetwork.value === 'binance-mainnet'
    ? 'https://bscscan.com/tx/'
    : 'https://testnet.bscscan.com/tx/'
  
  return baseUrl + txHash.value
})

// 獲取網絡配置
function getNetworkConfig() {
  switch (selectedNetwork.value) {
    case 'binance-mainnet':
      return {
        name: 'Binance Smart Chain',
        rpcUrl: 'https://bsc-dataseed.binance.org/',
        chainId: 56,
        symbol: 'BNB',
        explorerUrl: 'https://bscscan.com'
      }
    case 'binance-testnet':
      return {
        name: 'Binance Smart Chain Testnet',
        rpcUrl: 'https://data-seed-prebsc-1-s1.binance.org:8545/',
        chainId: 97,
        symbol: 'BNB',
        explorerUrl: 'https://testnet.bscscan.com'
      }
    default:
      return {
        name: 'Binance Smart Chain Testnet',
        rpcUrl: 'https://data-seed-prebsc-1-s1.binance.org:8545/',
        chainId: 97,
        symbol: 'BNB',
        explorerUrl: 'https://testnet.bscscan.com'
      }
  }
}

// 重置交易狀態
function resetTransactionState() {
  txHash.value = ''
  errorMessage.value = ''
}

// 檢查餘額
async function checkBalance() {
  if (!props.walletInfo?.address) {
    errorMessage.value = '沒有可用的錢包地址'
    return
  }

  try {
    isLoading.value = true
    loadingMessage.value = '檢查餘額中...'
    errorMessage.value = ''
    
    const networkConfig = getNetworkConfig()
    const provider = new ethers.JsonRpcProvider(networkConfig.rpcUrl)
    
    const balanceWei = await provider.getBalance(props.walletInfo.address)
    currentBalance.value = parseFloat(ethers.formatEther(balanceWei))
    
  } catch (error) {
    console.error('檢查餘額時發生錯誤:', error)
    errorMessage.value = `檢查餘額失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
  } finally {
    isLoading.value = false
    loadingMessage.value = ''
  }
}

// 發送交易
async function sendTransaction() {
  if (!props.walletInfo?.privateKey || !props.walletInfo?.address) {
    errorMessage.value = '缺少錢包私鑰或地址'
    return
  }
  
  if (!recipientAddress.value || !ethers.isAddress(recipientAddress.value)) {
    errorMessage.value = '收款地址無效'
    return
  }
  
  if (amount.value === null || amount.value <= 0) {
    errorMessage.value = '金額必須大於0'
    return
  }
  
  // 檢查地址是否相同
  if (props.walletInfo.address.toLowerCase() === recipientAddress.value.toLowerCase()) {
    errorMessage.value = '不能轉帳給自己的地址'
    return
  }

  resetTransactionState()
  
  try {
    isLoading.value = true
    loadingMessage.value = '準備交易...'
    
    const networkConfig = getNetworkConfig()
    const provider = new ethers.JsonRpcProvider(networkConfig.rpcUrl)
    
    // 首先檢查餘額
    loadingMessage.value = '檢查餘額...'
    const balanceWei = await provider.getBalance(props.walletInfo.address)
    const balance = ethers.formatEther(balanceWei)
    currentBalance.value = parseFloat(balance)
    
    const amountInEther = amount.value.toString()
    const amountInWei = ethers.parseEther(amountInEther)
    
    // 檢查餘額是否足夠
    if (balanceWei < amountInWei) {
      errorMessage.value = `餘額不足。當前餘額: ${balance} BNB`
      isLoading.value = false
      loadingMessage.value = ''
      return
    }
    
    // 創建錢包並連接到提供者
    loadingMessage.value = '創建交易...'
    const wallet = new ethers.Wallet(props.walletInfo.privateKey, provider)
    
    // 獲取當前 gas 價格
    const gasPrice = await provider.getFeeData()
    
    // 構建交易對象
    const tx = {
      to: recipientAddress.value,
      value: amountInWei,
      chainId: networkConfig.chainId,
      maxFeePerGas: gasPrice.maxFeePerGas,
      maxPriorityFeePerGas: gasPrice.maxPriorityFeePerGas,
      gasLimit: 21000, // 標準 ETH 轉帳 gas 限制
      nonce: await provider.getTransactionCount(wallet.address)
    }
    
    // 發送交易
    loadingMessage.value = '發送交易...'
    const transaction = await wallet.sendTransaction(tx)
    
    // 等待交易確認
    loadingMessage.value = '等待交易確認...'
    await transaction.wait(1) // 等待1個確認
    
    // 交易成功
    txHash.value = transaction.hash
    
    // 更新餘額
    const newBalanceWei = await provider.getBalance(props.walletInfo.address)
    currentBalance.value = parseFloat(ethers.formatEther(newBalanceWei))
    
  } catch (error) {
    console.error('交易發送失敗:', error)
    errorMessage.value = `交易失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
  } finally {
    isLoading.value = false
    loadingMessage.value = ''
  }
}
</script>

<style scoped>
.token-transfer-container {
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  background-color: #f9f9f9;
  color: #333333;
}

.no-wallet-message {
  padding: 15px;
  background-color: #fff2f0;
  border: 1px solid #ffccc7;
  border-radius: 4px;
  color: #f5222d;
  margin-bottom: 15px;
}

.wallet-info-display {
  margin-bottom: 20px;
}

.info-item {
  margin-bottom: 10px;
}

.info-label {
  font-weight: 600;
  margin-bottom: 5px;
  color: #333333;
}

.info-value {
  padding: 10px;
  background-color: #f5f5f5;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  word-break: break-all;
  color: #333333;
}

.network-selector, .form-group {
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
  font-size: 14px;
  width: 100%;
}

.button-group {
  display: flex;
  gap: 10px;
  margin-top: 15px;
}

.secondary-btn, .primary-btn {
  border: none;
  border-radius: 4px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
}

.secondary-btn {
  background-color: #f0f0f0;
  color: #333333;
  border: 1px solid #d9d9d9;
  flex: 1;
}

.primary-btn {
  background-color: #1890ff;
  color: white;
  flex: 2;
}

.secondary-btn:hover {
  background-color: #e0e0e0;
}

.primary-btn:hover {
  background-color: #40a9ff;
}

.secondary-btn:disabled, .primary-btn:disabled {
  background-color: #bfbfbf;
  color: #d9d9d9;
  cursor: not-allowed;
}

.balance-display {
  margin-top: 20px;
  padding: 10px;
  background-color: #f6ffed;
  border: 1px solid #b7eb8f;
  border-radius: 4px;
  color: #52c41a;
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
  margin-top: 15px;
}

.success-message {
  padding: 15px;
  background-color: #f6ffed;
  border: 1px solid #b7eb8f;
  border-radius: 4px;
  color: #52c41a;
  margin-top: 15px;
}

.tx-hash-container {
  margin-top: 10px;
}

.tx-hash-label {
  font-weight: 600;
  margin-bottom: 5px;
}

.tx-hash-value {
  padding: 10px;
  background-color: #f5f5f5;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-family: monospace;
  font-size: 14px;
  word-break: break-all;
  margin-bottom: 10px;
}

.explorer-link {
  display: inline-block;
  padding: 8px 16px;
  background-color: #1890ff;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  font-size: 14px;
  transition: background-color 0.3s;
}

.explorer-link:hover {
  background-color: #40a9ff;
}
</style>
