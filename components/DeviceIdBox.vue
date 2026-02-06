<template>
  <view class="device-card">
    <!-- 左侧信息区 -->
    <view class="content">
      <text class="label">设备识别码 </text>
      
      <view class="value-row">
        <!-- 加载中状态 -->
        <view v-if="deviceIdLoading" class="loading-state">
          <text>获取中...</text>
        </view>
        
        <!-- 显示 ID -->
        <text v-else class="device-id-text">{{ deviceId }}</text>
      </view>
    </view>

    <!-- 右侧复制按钮 -->
    <view class="copy-btn" :class="{ disabled: deviceIdLoading }" @click="handleCopy">
      <text>复制</text>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue';

// 定义事件
const emit = defineEmits(['loaded']); // 新增

const deviceId = ref('');
const deviceIdLoading = ref(false);
const hasAttemptedToFetchId = ref(false);

onMounted(() => {
  getDeviceId();
});

const getDeviceId = () => {
  if (deviceId.value || hasAttemptedToFetchId.value) return;
  
  hasAttemptedToFetchId.value = true;
  deviceIdLoading.value = true;

  uni.getSystemInfo({
    success: (systemInfo) => {
      if (systemInfo.deviceId) {
        deviceId.value = systemInfo.deviceId;
      } else {
        deviceId.value = 'device_' + Date.now().toString(36).slice(-8);
      }
      deviceIdLoading.value = false;
      
      // 【关键修改】获取成功后，通知父组件
      emit('loaded', deviceId.value);
    },
    fail: (err) => {
      deviceId.value = 'unknown_' + Date.now().toString(36).slice(-8);
      deviceIdLoading.value = false;
      
      // 失败也要通知，不然父组件一直等
      emit('loaded', deviceId.value);
    }
  });
};

// 2. 复制功能
const handleCopy = () => {
  if (deviceIdLoading.value || !deviceId.value) return;

  uni.setClipboardData({
    data: deviceId.value,
    success: () => {
      // uni.setClipboardData 默认会弹一个 toast，这里可以根据需要自定义
      // uni.showToast({ title: '已复制到剪贴板', icon: 'success' });
    },
    fail: () => {
      uni.showToast({ title: '复制失败', icon: 'none' });
    }
  });
};
</script>

<style lang="scss" scoped>
.device-card {
  background-color: #fff;
  border-radius: 16rpx;
  padding: 24rpx 30rpx;
  display: flex;
  align-items: center;
  justify-content: space-between;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
  margin: 20rpx 0; /* 上下留点间距 */
  border: 1rpx solid #eee;
}

.content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden; /* 防止长文本撑开 */
  margin-right: 20rpx;
}

.label {
  font-size: 24rpx;
  color: #999;
  margin-bottom: 8rpx;
}

.value-row {
  display: flex;
  align-items: center;
  min-height: 40rpx;
}

.device-id-text {
  font-size: 30rpx;
  font-weight: bold;
  color: #333;
  font-family: monospace; /* 等宽字体显示 ID 更专业 */
  word-break: break-all; /* 允许换行 */
}

.loading-state {
  font-size: 26rpx;
  color: #ccc;
}

.copy-btn {
  background-color: #f0f7ff;
  color: #007aff;
  font-size: 26rpx;
  font-weight: bold;
  padding: 12rpx 24rpx;
  border-radius: 30rpx;
  transition: all 0.2s;
  flex-shrink: 0; /* 防止被挤压 */

  &:active {
    background-color: #dcedff;
    transform: scale(0.95);
  }

  &.disabled {
    opacity: 0.5;
    pointer-events: none;
    background-color: #f5f5f5;
    color: #ccc;
  }
}
</style>