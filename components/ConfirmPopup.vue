<template>
  <view v-if="visible" class="confirm-modal">
    <view class="modal-content">
      <view class="modal-header">
        <text class="modal-title">{{ title }}</text>
      </view>
      
      <view class="modal-body">
        <slot></slot>
      </view>
      
      <view class="modal-footer">
        <button class="cancel-btn" @click="onCancel">{{ cancelText }}</button>
        <button 
          class="confirm-btn" 
          :disabled="confirmDisabled" 
          @click="onConfirm"
        >
          {{ confirmText }}
        </button>
      </view>
    </view>
  </view>
</template>

<script setup>
// 定义属性
const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  },
  title: {
    type: String,
    default: '确认操作'
  },
  confirmText: {
    type: String,
    default: '确定'
  },
  cancelText: {
    type: String,
    default: '取消'
  },
  confirmDisabled: {
    type: Boolean,
    default: false
  }
});

// 定义事件
const emit = defineEmits(['update:visible', 'confirm', 'cancel']);

const onCancel = () => {
  emit('update:visible', false); // 支持 v-model:visible
  emit('cancel');
};

const onConfirm = () => {
  if (props.confirmDisabled) return;
  // 注意：这里不自动关闭，交由父组件逻辑决定是否关闭（例如再次校验失败就不关）
  emit('confirm');
};
</script>

<style scoped>
/* 这里直接复用之前写好的样式 */
.confirm-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.modal-content {
  width: 80%;
  background-color: white;
  border-radius: 20rpx;
  padding: 30rpx;
  box-shadow: 0 10rpx 30rpx rgba(0, 0, 0, 0.2);
}

.modal-header {
  margin-bottom: 20rpx;
}

.modal-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #333;
  text-align: center;
  display: block;
}

.modal-body {
  margin-bottom: 30rpx;
}

/* 这里的样式为了让 slot 里的 text 默认好看一点，也可以在父组件覆盖 */
:deep(.modal-text) {
  display: block;
  font-size: 28rpx;
  color: #666;
  margin-bottom: 10rpx;
  word-break: break-all;
}

.modal-footer {
  display: flex;
  justify-content: space-between;
}

.cancel-btn {
  flex: 1;
  height: 70rpx;
  background-color: #f0f0f0;
  color: #333;
  font-size: 28rpx;
  border-radius: 10rpx;
  margin-right: 15rpx;
  border: none;
  line-height: 70rpx;
}

.confirm-btn {
  flex: 1;
  height: 70rpx;
  background-color: #007aff;
  color: white;
  font-size: 28rpx;
  border-radius: 10rpx;
  margin-left: 15rpx;
  border: none;
  line-height: 70rpx;
  opacity: 1;
}

.confirm-btn:disabled {
  opacity: 0.6;
}
</style>