<template>
  <view class="container">
    
    <!-- 1. 顶部日期选择 -->
    <view class="header-bar">
      <view class="date-label">当前日期：</view>
      <picker mode="date" :value="currentDate" @change="onDateChange">
        <view class="date-picker-box">
          <text>{{ currentDate }}</text>
          <text class="arrow">▼</text>
        </view>
      </picker>
    </view>

    <!-- 2. 中间网格输入区域 -->
    <scroll-view scroll-y class="grid-content">
      <view class="grid-wrapper" v-if="dataList.length > 0">
        <view class="grid-item" v-for="(item, index) in dataList" :key="index">
          <view class="square-box">
            <!-- 类型名称 -->
            <text class="type-name">{{ item.Type }}</text>
            
            <!-- 输入框 -->
            <input 
              class="qty-input" 
              type="number" 
              v-model="item.Qty" 
              placeholder="0" 
              placeholder-class="input-placeholder"
            />
          </view>
        </view>
      </view>

      <!-- 加载中 -->
      <view v-if="isLoading" class="loading-box">
        <text>加载数据中...</text>
      </view>
    </scroll-view>

    <!-- 3. 底部悬浮保存按钮 -->
   
    <view class="footer-btn-area">
      <!-- 
         1. :disabled="!hasInput" -> 没有输入时禁用点击
         2. :class="{ 'disabled': !hasInput }" -> 应用灰色样式
         3. 文字动态显示
      -->
      <button 
        class="save-btn" 
        :class="{ 'disabled': !hasInput }" 
        :disabled="!hasInput" 
        @click="handleSave"
      >
        {{ hasInput ? '保存数据' : '请先录入数据' }}
      </button>
    </view>

  </view>
</template>

<script setup>
import { ref, onMounted,computed  } from 'vue';

// 状态变量
const currentDate = ref('');
const dataList = ref([]);
const dateId = ref('0'); // 接口返回的 DateID，保存时可能需要用到
const isLoading = ref(false);

onMounted(() => {
  // 1. 初始化为今天
  const now = new Date();
  const year = now.getFullYear();
  const month = (now.getMonth() + 1).toString().padStart(2, '0');
  const day = now.getDate().toString().padStart(2, '0');
  currentDate.value = `${year}-${month}-${day}`;

  // 2. 获取数据
  getDataByDay();
});

// 判断是否有有效输入
// 只要数组中有一项的 Qty 不为空(null/undefined/空字符串)，就视为有输入
const hasInput = computed(() => {
  if (!dataList.value || dataList.value.length === 0) return false;
  return dataList.value.some(item => {
    return item.Qty !== null && item.Qty !== '' && String(item.Qty).trim() !== '';
  });
});

// 获取数据接口
const getDataByDay = () => {
  isLoading.value = true;
  dataList.value = []; // 清空旧数据防止闪烁

  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/GetDetailByDay',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentDate.value
    },
    success: (res) => {
      let data = res.data;
      if (typeof data === 'string') {
        try { data = JSON.parse(data); } catch(e){}
      }

      if (data && !data.isError && data.result) {
        dataList.value = data.result;
        dateId.value = data.DateID || '0';
      } else {
        uni.showToast({ title: '获取数据失败', icon: 'none' });
      }
    },
    fail: () => {
      uni.showToast({ title: '网络请求失败', icon: 'none' });
    },
    complete: () => {
      isLoading.value = false;
    }
  });
};

// 日期改变回调
const onDateChange = (e) => {
  currentDate.value = e.detail.value;
  getDataByDay(); // 重新请求
};

// 保存按钮 (目前只是打印数据，后续你提供保存接口后可替换)
const handleSave = () => {
  // 防御性代码：如果按钮是灰色禁用的，不允许点击
  if (!hasInput.value) return;

  uni.showLoading({ title: '保存中...', mask: true });

  // 1. 构建 list 参数
  // 接口要求 Qty 如果没填就是空字符串 ""，填了就是字符串格式的数字
  const listParam = dataList.value.map(item => {
    // 处理 Qty
    let finalQty = "";
    if (item.Qty !== null && item.Qty !== undefined && String(item.Qty).trim() !== '') {
      finalQty = String(item.Qty);
    }

    return {
      TypeID: item.TypeID,
      Qty: finalQty,
      Type: item.Type,
      ID: item.ID
    };
  });

  // 2. 发起请求
  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/UpdateTypeList',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentDate.value,
      dateID: dateId.value,
      list: listParam
    },
    success: (res) => {
      let data = res.data;
      if (typeof data === 'string') {
        try { data = JSON.parse(data); } catch(e){}
      }

      // 3. 判断结果
      if (data && data.isError === false) {
        uni.showToast({ title: '保存成功', icon: 'success' });
        
        // 延迟返回上一页，体验更好
        setTimeout(() => {
          uni.navigateBack();
        }, 1500);
      } else {
        uni.showToast({ title: '保存失败', icon: 'none' });
      }
    },
    fail: (err) => {
      console.error(err);
      uni.showToast({ title: '网络请求错误', icon: 'none' });
    },
    complete: () => {
      uni.hideLoading();
    }
  });
};

</script>

<style lang="scss" scoped>
.container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f7fa;
}

/* 1. 顶部日期栏 */
.header-bar {
  background-color: #fff;
  padding: 30rpx 40rpx;
  display: flex;
  align-items: center;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.05);
  z-index: 10;

  .date-label {
    font-size: 30rpx;
    font-weight: bold;
    color: #333;
    margin-right: 20rpx;
  }

  .date-picker-box {
    background-color: #eef2f9;
    padding: 10rpx 30rpx;
    border-radius: 30rpx;
    display: flex;
    align-items: center;
    color: #007aff;
    font-weight: bold;
    font-size: 30rpx;

    .arrow {
      margin-left: 10rpx;
      font-size: 24rpx;
      opacity: 0.6;
    }
  }
}

/* 2. 中间滚动区域 */
.grid-content {
  flex: 1; /* 占满剩余空间 */
  padding: 20rpx;
  box-sizing: border-box;
  overflow-y: auto;
}

.grid-wrapper {
  display: flex;
  flex-wrap: wrap;
  padding-bottom: 150rpx; /* 防止内容被底部按钮遮挡 */
}

.grid-item {
  width: 33.33%; /* 一行三个 */
  padding: 10rpx;
  box-sizing: border-box;
}

/* 正方形卡片 */
.square-box {
  background-color: #fff;
  border-radius: 16rpx;
  /* 利用 aspect-ratio 实现正方形 (兼容性较好)，或者用 padding-bottom 技巧 */
  aspect-ratio: 1 / 1; 
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.03);
  border: 1rpx solid transparent;
  transition: all 0.2s;

  &:active {
    transform: scale(0.98);
    border-color: #007aff;
  }

  .type-name {
    font-size: 32rpx;
    font-weight: bold;
    color: #333;
    margin-bottom: 20rpx;
  }

  .qty-input {
    width: 80%;
    height: 70rpx;
    background-color: #f5f7fa;
    border-radius: 8rpx;
    text-align: center;
    font-size: 32rpx;
    color: #333;
    border: 1rpx solid #eee;

    &:focus {
      border-color: #007aff;
      background-color: #fff;
    }
  }
}

.loading-box {
  text-align: center;
  padding: 50rpx;
  color: #999;
}

/* 3. 底部按钮区域 */
.footer-btn-area {
  position: fixed;
  bottom: 40rpx;
  right: 40rpx;
  left: 40rpx; /* 如果想居中或者全宽 */
  z-index: 20;
}

/* 原来的 .save-btn 样式保持不变 */
.save-btn {
  background: linear-gradient(135deg, #007aff, #0056b3);
  color: #fff;
  border-radius: 50rpx;
  font-size: 32rpx;
  font-weight: bold;
  box-shadow: 0 6rpx 20rpx rgba(0, 122, 255, 0.4);
  border: none;
  transition: all 0.3s; /* 加个过渡动画更顺滑 */
  
  &:active {
    opacity: 0.9;
    transform: translateY(2rpx);
  }

  /* --- 新增：禁用状态样式 --- */
  &.disabled {
    background: #ccc; /* 灰色背景 */
    color: #fff;
    box-shadow: none; /* 去掉阴影 */
    opacity: 1;
    pointer-events: none; /* 禁止点击 */
  }
}

/* input placeholder 样式 */
.input-placeholder {
  color: #ccc;
  font-size: 28rpx;
}
</style>