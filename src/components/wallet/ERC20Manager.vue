<template>
  <div class="erc20-manager-container">
    <h3>{{ title }}</h3>
    
    <div v-if="!walletInfo" class="no-wallet-message">
      <p>請先生成 HD 錢包</p>
    </div>
    
    <div v-else class="erc20-manager-content">
      <!-- 代幣合約輸入區域 -->
      <div class="contract-input-section">
        <div class="network-selector">
          <label for="erc20-network-select">網絡:</label>
          <select id="erc20-network-select" v-model="selectedNetwork" class="input-control">
            <option value="binance-mainnet">幣安智能鏈 (Mainnet)</option>
            <option value="binance-testnet">幣安智能鏈 (Testnet)</option>
          </select>
        </div>
        
        <div class="input-group">
          <label for="contract-address">ERC20 合約地址:</label>
          <div class="input-with-button">
            <input 
              id="contract-address" 
              v-model="contractAddress" 
              type="text" 
              class="input-control"
              placeholder="輸入有效的 ERC20 合約地址"
            />
            <button @click="loadTokenInfo" class="action-btn" :disabled="isLoading || !isValidAddress">
              載入
            </button>
          </div>
        </div>
      </div>
      
      <!-- 代幣信息顯示 -->
      <div v-if="tokenInfo.symbol" class="token-info-display">
        <div class="token-info-header">
          <h4>{{ tokenInfo.name }} ({{ tokenInfo.symbol }})</h4>
          <div class="token-decimals">精度: {{ tokenInfo.decimals }}</div>
        </div>
      </div>
      
      <!-- 標籤選擇器 -->
      <div class="tabs-container">
        <div class="tabs-header">
          <div 
            class="tab-item" 
            :class="{ active: activeTab === 'balance' }"
            @click="activeTab = 'balance'"
          >
            餘額查詢
          </div>
          <div 
            class="tab-item" 
            :class="{ active: activeTab === 'transfer' }"
            @click="activeTab = 'transfer'"
          >
            代幣轉帳
          </div>
        </div>
        
        <!-- 餘額查詢標籤 -->
        <div v-if="activeTab === 'balance'" class="tab-content">
          <div class="balance-query-section">
            <button @click="checkBalance" class="primary-btn" :disabled="isLoading || !tokenInfo.symbol">
              {{ isLoading ? '查詢中...' : '查詢餘額' }}
            </button>
            
            <div v-if="tokenBalance !== null" class="balance-result">
              <h4>代幣餘額</h4>
              <div class="token-balance">{{ formattedBalance }} {{ tokenInfo.symbol }}</div>
            </div>
          </div>
        </div>
        
        <!-- 代幣轉帳標籤 -->
        <div v-if="activeTab === 'transfer'" class="tab-content">
          <div class="token-transfer-section">
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
              <label for="token-amount">金額:</label>
              <input 
                id="token-amount" 
                v-model.number="transferAmount" 
                type="number"
                step="0.000001"
                min="0.000001"
                class="input-control"
                placeholder="輸入轉帳金額"
              />
            </div>
            
            <button @click="transferToken" class="primary-btn" :disabled="isLoading || !isTransferValid">
              {{ isLoading ? '處理中...' : '發送代幣' }}
            </button>
            
            <div v-if="tokenBalance !== null" class="balance-info">
              <p>可用餘額: {{ formattedBalance }} {{ tokenInfo.symbol }}</p>
            </div>
          </div>
        </div>
      </div>
      
      <!-- 狀態顯示區域 -->
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

// ERC20 代幣 ABI (僅包含所需的函數)
const ERC20_ABI = [
  // 查詢餘額
  'function balanceOf(address owner) view returns (uint256)',
  // 轉帳
  'function transfer(address to, uint256 amount) returns (bool)',
  // 代幣信息
  'function name() view returns (string)',
  'function symbol() view returns (string)',
  'function decimals() view returns (uint8)'
]

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
  title: 'ERC20 代幣管理',
  walletInfo: null
})

// 響應式狀態
const selectedNetwork = ref('binance-testnet')
const contractAddress = ref('')
const activeTab = ref('balance')
const isLoading = ref(false)
const loadingMessage = ref('')
const errorMessage = ref('')
const txHash = ref('')

// 代幣信息
const tokenInfo = ref({
  name: '',
  symbol: '',
  decimals: 0
})

// 代幣餘額
const tokenBalance = ref<bigint | null>(null)

// 轉帳相關
const recipientAddress = ref('')
const transferAmount = ref<number | null>(null)

// 計算屬性
const isValidAddress = computed(() => {
  return contractAddress.value && ethers.isAddress(contractAddress.value)
})

const formattedBalance = computed(() => {
  if (tokenBalance.value === null || tokenInfo.value.decimals === 0) return '0'
  
  // 使用 decimals 將代幣餘額從最小單位轉換為標準表示
  return parseFloat(ethers.formatUnits(tokenBalance.value, tokenInfo.value.decimals)).toFixed(6)
})

const isTransferValid = computed(() => {
  return (
    tokenInfo.value.symbol && 
    recipientAddress.value && 
    ethers.isAddress(recipientAddress.value) &&
    transferAmount.value !== null && 
    transferAmount.value > 0 &&
    tokenBalance.value !== null && 
    transferAmount.value <= parseFloat(formattedBalance.value)
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

// 創建合約實例
function createContractInstance(provider: ethers.JsonRpcProvider, withSigner = false) {
  if (!isValidAddress.value) {
    throw new Error('ERC20 合約地址無效')
  }
  
  if (withSigner && !props.walletInfo?.privateKey) {
    throw new Error('需要錢包私鑰進行交易')
  }
  
  const contract = new ethers.Contract(contractAddress.value, ERC20_ABI, provider)
  
  if (withSigner && props.walletInfo?.privateKey) {
    const wallet = new ethers.Wallet(props.walletInfo.privateKey, provider)
    return contract.connect(wallet)
  }
  
  return contract
}

// 重置交易狀態
function resetTransactionState() {
  txHash.value = ''
  errorMessage.value = ''
}

// 載入代幣信息
async function loadTokenInfo() {
  if (!isValidAddress.value) {
    errorMessage.value = 'ERC20 合約地址無效'
    return
  }

  try {
    isLoading.value = true
    loadingMessage.value = '載入代幣信息...'
    errorMessage.value = ''
    
    const networkConfig = getNetworkConfig()
    const provider = new ethers.JsonRpcProvider(networkConfig.rpcUrl)
    
    // 創建合約實例
    const contract = createContractInstance(provider)
    
    // 獲取代幣信息
    const [name, symbol, decimals] = await Promise.all([
      contract.name(),
      contract.symbol(),
      contract.decimals()
    ])
    
    tokenInfo.value = {
      name,
      symbol,
      decimals
    }
    
    // 清除之前的餘額信息
    tokenBalance.value = null
    
  } catch (error) {
    console.error('載入代幣信息失敗:', error)
    errorMessage.value = `載入代幣信息失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
    
    // 重置代幣信息
    tokenInfo.value = {
      name: '',
      symbol: '',
      decimals: 0
    }
  } finally {
    isLoading.value = false
    loadingMessage.value = ''
  }
}

// 查詢代幣餘額
async function checkBalance() {
  if (!props.walletInfo?.address) {
    errorMessage.value = '沒有可用的錢包地址'
    return
  }
  
  if (!tokenInfo.value.symbol) {
    errorMessage.value = '請先載入代幣信息'
    return
  }

  try {
    isLoading.value = true
    loadingMessage.value = '查詢代幣餘額...'
    errorMessage.value = ''
    
    const networkConfig = getNetworkConfig()
    const provider = new ethers.JsonRpcProvider(networkConfig.rpcUrl)
    
    // 創建合約實例
    const contract = createContractInstance(provider)
    
    // 查詢餘額
    const balance = await contract.balanceOf(props.walletInfo.address)
    tokenBalance.value = balance
    
  } catch (error) {
    console.error('查詢代幣餘額失敗:', error)
    errorMessage.value = `查詢餘額失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
    tokenBalance.value = null
  } finally {
    isLoading.value = false
    loadingMessage.value = ''
  }
}

// 轉帳代幣
async function transferToken() {
  if (!props.walletInfo?.privateKey || !props.walletInfo?.address) {
    errorMessage.value = '缺少錢包私鑰或地址'
    return
  }
  
  if (!recipientAddress.value || !ethers.isAddress(recipientAddress.value)) {
    errorMessage.value = '收款地址無效'
    return
  }
  
  if (transferAmount.value === null || transferAmount.value <= 0) {
    errorMessage.value = '金額必須大於0'
    return
  }
  
  if (!tokenInfo.value.symbol) {
    errorMessage.value = '請先載入代幣信息'
    return
  }
  
  if (tokenBalance.value === null) {
    await checkBalance()
    if (tokenBalance.value === null) return // 如果餘額查詢失敗，不繼續執行轉帳
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
    
    // 創建合約實例 (帶簽名者)
    const contract = createContractInstance(provider, true)
    
    // 轉換金額為代幣最小單位
    const amount = ethers.parseUnits(transferAmount.value.toString(), tokenInfo.value.decimals)
    
    // 檢查餘額是否足夠
    if (tokenBalance.value < amount) {
      errorMessage.value = `餘額不足。當前餘額: ${formattedBalance.value} ${tokenInfo.value.symbol}`
      isLoading.value = false
      loadingMessage.value = ''
      return
    }
    
    // 發送交易
    loadingMessage.value = '發送交易...'
    const transaction = await contract.transfer(recipientAddress.value, amount)
    
    // 等待交易確認
    loadingMessage.value = '等待交易確認...'
    await transaction.wait(1) // 等待1個確認
    
    // 交易成功
    txHash.value = transaction.hash
    
    // 更新餘額
    await checkBalance()
    
  } catch (error) {
    console.error('轉帳失敗:', error)
    errorMessage.value = `轉帳失敗: ${error instanceof Error ? error.message : '未知錯誤'}`
  } finally {
    isLoading.value = false
    loadingMessage.value = ''
  }
}
</script>

<style scoped>
.erc20-manager-container {
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

.contract-input-section {
  margin-bottom: 20px;
}

.network-selector {
  margin-bottom: 15px;
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

.input-with-button {
  display: flex;
  gap: 8px;
}

.input-with-button .input-control {
  flex: 1;
}

.input-control {
  padding: 8px 12px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 14px;
  width: 100%;
}

.action-btn {
  background-color: #1890ff;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 0 15px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
}

.action-btn:hover {
  background-color: #40a9ff;
}

.action-btn:disabled {
  background-color: #bfbfbf;
  cursor: not-allowed;
}

.token-info-display {
  margin-bottom: 20px;
  padding: 10px 15px;
  background-color: #f0f0f0;
  border-radius: 4px;
  border-left: 4px solid #1890ff;
}

.token-info-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.token-info-header h4 {
  margin: 0;
  color: #1890ff;
}

.token-decimals {
  font-size: 12px;
  color: #666666;
}

.tabs-container {
  margin-bottom: 20px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  overflow: hidden;
}

.tabs-header {
  display: flex;
  background-color: #f5f5f5;
  border-bottom: 1px solid #d9d9d9;
}

.tab-item {
  padding: 10px 15px;
  cursor: pointer;
  transition: background-color 0.3s;
  flex: 1;
  text-align: center;
  font-weight: 500;
}

.tab-item:hover {
  background-color: #e6f7ff;
}

.tab-item.active {
  background-color: #1890ff;
  color: white;
}

.tab-content {
  padding: 15px;
  background-color: white;
}

.balance-query-section {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.token-transfer-section {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.balance-result {
  padding: 15px;
  background-color: #f6ffed;
  border: 1px solid #b7eb8f;
  border-radius: 4px;
}

.balance-result h4 {
  margin: 0 0 10px 0;
  color: #52c41a;
}

.token-balance {
  font-size: 24px;
  font-weight: 600;
  color: #52c41a;
}

.balance-info {
  padding: 10px;
  background-color: #f5f5f5;
  border-radius: 4px;
  font-size: 14px;
  color: #666666;
}

.primary-btn {
  background-color: #1890ff;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 20px;
  cursor: pointer;
  font-size: 16px;
  transition: background-color 0.3s;
}

.primary-btn:hover {
  background-color: #40a9ff;
}

.primary-btn:disabled {
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
