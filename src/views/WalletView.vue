<template>
  <div class="wallet-container">
    <h1>BNB 錢包</h1>
    
    <!-- 助記詞生成模塊 -->
    <div class="wallet-module">
      <div class="wallet-module-header">
        <h2>1. 助記詞生成</h2>
        <p class="module-description">生成用於創建和恢復錢包的隨機助記詞</p>
      </div>
      
      <div class="wallet-section">
        <mnemonic-generator @mnemonic-generated="handleMnemonicGenerated" />
      </div>
      
      <div v-if="currentMnemonic" class="wallet-info">
        <div class="info-box mnemonic-info-box">
          <h3>當前助記詞</h3>
          <p>{{ currentMnemonic }}</p>
        </div>
      </div>
    </div>
    
    <!-- HD 錢包生成模塊 -->
    <div class="wallet-module">
      <div class="wallet-module-header">
        <h2>2. HD 錢包生成</h2>
        <p class="module-description">從助記詞創建階層確定性錢包</p>
      </div>
      
      <div class="wallet-section">
        <HDWalletGenerator :mnemonic="currentMnemonic" @wallet-generated="handleWalletGenerated" />
      </div>
      
      <div v-if="hdWallet" class="wallet-info">
        <div class="info-box hdwallet-info-box">
          <h3>HD 錢包信息</h3>
          <p><strong>地址:</strong> {{ hdWallet.address }}</p>
          <p><strong>路徑:</strong> {{ hdWallet.path }}</p>
        </div>
      </div>
    </div>
    
    <!-- 資產餘額查詢模塊 -->
    <div class="wallet-module">
      <div class="wallet-module-header">
        <h2>3. 資產餘額查詢</h2>
        <p class="module-description">查詢錢包地址的原生幣BNB餘額</p>
      </div>
      
      <div class="wallet-section">
        <BalanceChecker :wallet-address="hdWallet?.address || ''" />
      </div>
    </div>
    
    <!-- BNB 轉帳模塊 -->
    <div class="wallet-module">
      <div class="wallet-module-header">
        <h2>4. BNB 轉帳</h2>
        <p class="module-description">從當前錢包發送BNB到其他地址</p>
      </div>
      
      <div class="wallet-section">
        <TokenTransfer :wallet-info="hdWallet" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import MnemonicGenerator from '@/components/wallet/MnemonicGenerator.vue'
import HDWalletGenerator from '@/components/wallet/HDWalletGenerator.vue'
import BalanceChecker from '@/components/wallet/BalanceChecker.vue'
import TokenTransfer from '@/components/wallet/TokenTransfer.vue'

// 助記詞狀態
const currentMnemonic = ref('')

// HD 錢包狀態
interface HDWalletInfo {
  address: string
  publicKey: string
  privateKey: string
  path: string
  index: number
}

const hdWallet = ref<HDWalletInfo | null>(null)



// 處理助記詞生成
function handleMnemonicGenerated(mnemonic: string) {
  currentMnemonic.value = mnemonic
  console.log('已生成新助記詞')
  // 重置 HD 錢包狀態
  hdWallet.value = null
}

// 處理 HD 錢包生成
function handleWalletGenerated(wallet: HDWalletInfo) {
  hdWallet.value = wallet
  console.log('已生成 HD 錢包:', wallet.address)
}
</script>

<style scoped>
.wallet-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 30px;
  font-family: Arial, sans-serif;
  background-color: #ffffff;
  color: #333333;
}

/* 主標題樣式 */
.wallet-container h1 {
  color: #1a1a1a;
  text-align: center;
  margin-bottom: 40px;
  font-size: 32px;
  border-bottom: 2px solid #f0f0f0;
  padding-bottom: 15px;
}

/* 模塊容器樣式 */
.wallet-module {
  margin-bottom: 50px;
  border: 1px solid #e8e8e8;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  background-color: #ffffff;
}

/* 模塊標題區域 */
.wallet-module-header {
  background-color: #f5f5f5;
  padding: 20px 25px;
  border-bottom: 1px solid #e8e8e8;
}

.wallet-module-header h2 {
  margin: 0 0 8px 0;
  color: #1890ff;
  font-size: 22px;
}

.module-description {
  margin: 0;
  color: #666666;
  font-size: 14px;
}

/* 功能區塊樣式 */
.wallet-section {
  padding: 25px;
  background-color: #ffffff;
}

/* 資訊顯示區域 */
.wallet-info {
  padding: 0 25px 25px 25px;
}

.info-box {
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
  margin-top: 15px;
}

.info-box h3 {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 18px;
  font-weight: 600;
}

/* 助記詞信息框 */
.mnemonic-info-box {
  background-color: #f6ffed;
  border: 1px solid #b7eb8f;
  color: #333333;
  word-break: break-all;
}

.mnemonic-info-box h3 {
  color: #52c41a;
}

/* HD 錢包信息框 */
.hdwallet-info-box {
  background-color: #e6f7ff;
  border: 1px solid #91d5ff;
  color: #333333;
  word-break: break-all;
}

.hdwallet-info-box h3 {
  color: #1890ff;
}

/* 信息框中的內容 */
.info-box p {
  margin: 8px 0;
  line-height: 1.6;
}

.info-box strong {
  font-weight: 600;
  margin-right: 5px;
}
</style>
