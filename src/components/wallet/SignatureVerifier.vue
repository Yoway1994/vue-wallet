<template>
  <div class="signature-verifier">
    <div class="message-container">
      <h2>待签名信息</h2>
      <div class="message-box">{{ messageToSign }}</div>
    </div>

    <div class="actions">
      <button @click="verifyWallet" :disabled="isVerifying" class="verify-button">
        {{ isVerifying ? '验证中...' : '验证钱包' }}
      </button>
    </div>

    <div v-if="verificationResult" class="result-container">
      <h2>验证结果</h2>
      <div class="result-box" :class="{ success: isVerified, error: !isVerified }">
        <div v-if="isVerified" class="success-message">
          <h3>验证成功!</h3>
          <p><strong>钱包地址:</strong> {{ walletAddress }}</p>
          <p><strong>签名:</strong> {{ signature }}</p>
        </div>
        <div v-else class="error-message">
          <h3>验证失败</h3>
          <p>{{ verificationResult }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, defineEmits } from 'vue'
import { ethers } from 'ethers'

// 定义组件事件
const emit = defineEmits(['verification-success', 'verification-error'])

// 待签名的消息
const messageToSign = ref('我同意连接钱包并验证身份 - ' + new Date().toISOString())
const isVerifying = ref(false)
const verificationResult = ref('')
const isVerified = ref(false)
const walletAddress = ref('')
const signature = ref('')

// 验证钱包签名
async function verifyWallet() {
  isVerifying.value = true
  verificationResult.value = ''
  isVerified.value = false
  walletAddress.value = ''
  signature.value = ''

  try {
    // 检查是否存在 ethereum 提供者
    if (!window.ethereum) {
      throw new Error('未检测到 MetaMask 或其他以太坊钱包，请安装钱包扩展后重试')
    }

    // 连接到以太坊提供者 (ethers v6 语法)
    const provider = new ethers.BrowserProvider(window.ethereum)

    // 请求用户授权连接钱包
    await provider.send('eth_requestAccounts', [])

    // 获取签名者
    const signer = await provider.getSigner()

    // 获取钱包地址
    const address = await signer.getAddress()
    walletAddress.value = address

    // 请求用户签名消息
    const signedMessage = await signer.signMessage(messageToSign.value)
    signature.value = signedMessage

    // 验证签名 (ethers v6 语法)
    const recoveredAddress = ethers.verifyMessage(messageToSign.value, signedMessage)

    // 检查恢复的地址是否与钱包地址匹配
    if (recoveredAddress.toLowerCase() === address.toLowerCase()) {
      isVerified.value = true
      verificationResult.value = '签名验证成功！'

      // 触发成功事件
      emit('verification-success', {
        address,
        signature: signedMessage,
      })
    } else {
      throw new Error('签名验证失败：恢复的地址与钱包地址不匹配')
    }
  } catch (error: Error | unknown) {
    const errorMessage = error instanceof Error ? error.message : '验证过程中发生错误'
    console.error('钱包验证错误:', error)
    verificationResult.value = errorMessage

    // 触发错误事件
    emit('verification-error', errorMessage)
  } finally {
    isVerifying.value = false
  }
}
</script>

<style scoped>
.signature-verifier {
  width: 100%;
}

.message-container {
  margin-bottom: 30px;
}

.message-box {
  padding: 15px;
  background-color: rgb(77, 101, 125);
  border: 1px solid #e9ecef;
  border-radius: 4px;
  word-break: break-all;
}

.actions {
  margin-bottom: 30px;
}

.verify-button {
  padding: 12px 24px;
  background-color: #f0b90b; /* BNB 颜色 */
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.verify-button:hover {
  background-color: #e6ae00;
}

.verify-button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.result-container {
  margin-top: 20px;
}

.result-box {
  padding: 20px;
  border-radius: 4px;
  margin-top: 10px;
}

.success {
  background-color: #d4edda;
  border: 1px solid #c3e6cb;
  color: #155724;
}

.error {
  background-color: #f8d7da;
  border: 1px solid #f5c6cb;
  color: #721c24;
}

.success-message h3,
.error-message h3 {
  margin-top: 0;
}
</style>
