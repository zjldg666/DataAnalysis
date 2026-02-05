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
              :style="{ color: getNumColor(item.Qty) }"
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

const getNumColor = (val) => {
  // 1. 如果为空，默认黑色
  if (val === null || val === '' || val === undefined) return '#333';
  
  // 2. 转为数字
  const num = Number(val);
  
  // 3. 判断正负
  if (num > 0) return '#007aff'; // 蓝色
  if (num < 0) return '#ff3b30'; // 红色
  return '#333'; // 0 显示黑色
};
</script>

<style lang="scss" scoped>
.container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f2f3f5; /* 统一的灰底 */
}

/* --- 1. 顶部日期栏 (黑底风格) --- */
.header-bar {
  background-color: #1a1a1a;
  padding: 30rpx 40rpx 40rpx;
  display: flex;
  align-items: center;
  justify-content: space-between; /* 左右两端对齐 */
  box-shadow: 0 4rpx 12rpx rgba(0,0,0,0.1);
  z-index: 10;
  border-bottom-left-radius: 30rpx;
  border-bottom-right-radius: 30rpx;
  margin-bottom: 20rpx;

  .date-label {
    font-size: 30rpx;
    font-weight: bold;
    color: #fff; /* 白字 */
  }

  .date-picker-box {
    background-color: rgba(255, 255, 255, 0.2); /* 半透明白底 */
    padding: 10rpx 30rpx;
    border-radius: 30rpx;
    display: flex;
    align-items: center;
    color: #fff;
    font-weight: bold;
    font-size: 30rpx;
    border: 1px solid rgba(255, 255, 255, 0.3);

    .arrow {
      margin-left: 10rpx;
      font-size: 24rpx;
      opacity: 0.8;
      color: #ff9900; /* 橙色箭头点缀 */
    }
  }
}

/* --- 2. 中间滚动区域 --- */
.grid-content {
  flex: 1; 
  padding: 0 20rpx; /* 左右留边距 */
  box-sizing: border-box;
  overflow-y: auto;
  /* 隐藏滚动条 (兼容性写法) */
  &::-webkit-scrollbar { display: none; }
}

.grid-wrapper {
  display: flex;
  flex-wrap: wrap;
  padding-bottom: 180rpx; /* 底部留白给按钮 */
  /* 让卡片嵌入 Header 下方一点点 */
  margin-top: -20rpx; 
}

.grid-item {
  width: 33.33%; 
  padding: 10rpx;
  box-sizing: border-box;
}

/* 正方形卡片优化 */
.square-box {
  background-color: #fff;
  border-radius: 16rpx;
  aspect-ratio: 1 / 1; 
  display: flex;
  flex-direction: column;
  justify-content: space-between; /* 上下分布 */
  padding: 24rpx 16rpx;
  box-sizing: border-box;
  box-shadow: 0 2rpx 8rpx rgba(0,0,0,0.05);
  transition: all 0.2s;
  border: 1rpx solid transparent;

  /* 获取焦点时的微动效 */
  &:focus-within {
    transform: translateY(-4rpx);
    box-shadow: 0 8rpx 20rpx rgba(0,0,0,0.08);
    border-color: #333; /* 黑边框 */
  }

  .type-name {
    font-size: 36rpx; /* 加大名字 */
    font-weight: 900;
    color: #000;
    text-align: center;
    /* 名字方块背景 */
    background-color: #f5f5f5;
    width: 100%;
    padding: 10rpx 0;
    border-radius: 8rpx;
  }

  .qty-input {
    width: 100%;
    height: 80rpx;
    background-color: #fff; /* 纯白背景 */
    text-align: center;
    font-size: 40rpx; /* 数字加大 */
    font-weight: bold;
    color: #333;
    border-bottom: 2rpx solid #eee; /* 只留底边框 */
    
    &:focus {
      border-bottom-color: #000; /* 聚焦黑底边 */
    }
  }
}

.loading-box {
  text-align: center;
  padding: 60rpx;
  color: #999;
  font-size: 26rpx;
}

/* --- 3. 底部按钮区域 --- */
.footer-btn-area {
  position: fixed;
  bottom: 50rpx;
  left: 40rpx; 
  right: 40rpx;
  z-index: 20;
}

.save-btn {
  /* 黑色主按钮 */
  background: #1a1a1a; 
  color: #fff;
  height: 90rpx;
  line-height: 90rpx;
  border-radius: 16rpx; /* 方圆角 */
  font-size: 32rpx;
  font-weight: bold;
  box-shadow: 0 8rpx 24rpx rgba(0, 0, 0, 0.15);
  border: none;
  
  &:active {
    transform: scale(0.98);
    background: #000;
  }

  /* 禁用状态 */
  &.disabled {
    background: #e0e0e0; 
    color: #999;
    box-shadow: none;
  }
}

.input-placeholder {
  color: #ddd;
  font-weight: normal;
  font-size: 30rpx;
}
</style>