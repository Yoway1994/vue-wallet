<template>
  <div class="wallet-container">
    <h1>BNB 钱包</h1>

    <div class="wallet-section">
      <h2>钱包签名验证</h2>
      <signature-verifier
        @verification-success="handleVerificationSuccess"
        @verification-error="handleVerificationError"
      />
    </div>

    <div v-if="walletVerified" class="wallet-info">
      <div class="info-box">
        <h3>已验证钱包</h3>
        <p><strong>地址:</strong> {{ walletAddress }}</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import SignatureVerifier from '@/components/wallet/SignatureVerifier.vue'

// 钱包状态
const walletVerified = ref(false)
const walletAddress = ref('')
const walletSignature = ref('')

// 定义验证成功返回的数据类型
interface VerificationResult {
  address: string
  signature: string
}

// 处理验证成功
function handleVerificationSuccess(data: VerificationResult) {
  walletVerified.value = true
  walletAddress.value = data.address
  walletSignature.value = data.signature
  console.log('钱包验证成功', data)
}

// 处理验证错误
function handleVerificationError(errorMessage: string) {
  walletVerified.value = false
  walletAddress.value = ''
  walletSignature.value = ''
  console.error('钱包验证失败:', errorMessage)
}
</script>

<style scoped>
.wallet-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  background-color: rgb(130, 52, 52);
}

.wallet-section {
  margin-bottom: 30px;
  border: 1px solid #eaeaea;
  border-radius: 8px;
  padding: 20px;
  background-color: rgb(117, 60, 60);
}

.wallet-info {
  margin-top: 30px;
}

.info-box {
  padding: 20px;
  background-color: #e6f7ff;
  border: 1px solid #91d5ff;
  border-radius: 8px;
  color: #0050b3;
}

.info-box h3 {
  margin-top: 0;
  color: #0050b3;
}
</style>
