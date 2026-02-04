<template>
  <view class="container">
    
    <!-- 1. 头部区域 -->
    <!-- 只要 totalData 有值就显示 (我们在JS里保证了它就算接口空也会有初始值) -->
    <view class="header" v-if="totalData">
      <!-- 第一排：年份选择 -->
      <view class="header-row-top">
        <!-- 
           修改点在这里：
           1. start="2000" 固定起始年份
           2. :end="currentLimitYear" 动态绑定结束年份（当前年）
           3. fields="year" 只选择年份
        -->
        <picker 
          mode="selector" 
          :range="yearList" 
          :value="yearIndex" 
          @change="onYearChange"
        >
          <view class="year-selector">
            <text class="year-text">年份：{{ currentYear }}</text>
            <text class="arrow-icon">▼</text>
          </view>
        </picker>
      </view>

      <!-- 第二排：加总数和总次数 -->
      <view class="header-row-bottom">
        <view class="stat-item">
          <text class="label">加总数</text>
          <text class="value">{{ totalData.xiaoji }}</text>
        </view>
		<view class="stat-item clickable-item" @click="goYearTable">
          <text class="label">总次数 (点击查看明细)</text> <!-- 稍微改下文字提示用户 -->
          <text class="value">{{ totalData.zhengshu }}</text>
          <text class="arrow">></text> <!-- 加个小箭头 -->
        </view>
      </view>

      <!-- 右下角按钮 -->
      <view class="record-btn" @click="handleRecord">
        <text>录入数据</text>
      </view>
    </view>

    <!-- 加载中占位 -->
    <view v-else class="loading-placeholder">
      <text>加载中...</text>
    </view>

    <!-- 2. 下方列表区域 -->
    <view class="grid-container" v-if="listData.length > 0">
      <view class="grid-item" v-for="(item, index) in listData" :key="index" @click="goDetail(item)">
        <view class="item-content">
          <!-- 左侧正方形名字 -->
          <view class="name-square">
            <text>{{ item.name }}</text>
          </view>
          
          <!-- 右侧数据详情 -->
          <view class="info-list">
            <text>场数:{{ item.changshu }}</text>
            <text>正数:{{ item.zhengshu }}</text>
            <text>小计:{{ item.xiaoji }}</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 空状态提示 -->
    <view v-if="totalData && listData.length === 0" class="empty-state">
      <text>该年份暂无明细数据</text>
    </view>

  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue';

// --- 状态变量 ---
const currentYear = ref('');      // 当前选中的年份（用于请求接口）
const currentLimitYear = ref(''); // 当前实际年份（用于限制 picker 的最大值）
const yearList = ref([]);         //存放 2000-当前年 的数组
const yearIndex = ref(0);         //Picker 选中的索引
const totalData = ref(null); 
const listData = ref([]);    

// --- 生命周期 ---
onMounted(() => {
  const date = new Date();
  const nowYear = date.getFullYear(); // 获取数字类型的当前年，例如 2026
  
  // 1. 生成年份数组 (从当前年 倒序排 到 2000年)
  // 结果例如：['2026', '2025', ... '2000']
  let years = [];
  for (let i = nowYear; i >= 2000; i--) {
    years.push(i.toString());
  }
  yearList.value = years;

  // 2. 默认选中第一项 (即当前年)
  yearIndex.value = 0;
  currentYear.value = years[0];

  // 3. 调用接口
  getData();
});

// --- 方法 ---

const getData = () => {
  uni.showLoading({ title: '加载中...', mask: true });
  
  // 你的接口地址（修正了双斜杠）
  const url = 'http://13.94.38.44:8080/TypeList/GetTotalByYear';

  uni.request({
    url: url,
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentYear.value // 使用当前选中的年份
    },
    success: (res) => {
      let responseData = res.data;
      if (typeof responseData === 'string') {
        try { responseData = JSON.parse(responseData); } catch (e) {}
      }

      if (responseData && !responseData.isError) {
        processData(responseData.result || []);
      } else {
        processData([]); // 错误也显示空页面
      }
    },
    fail: (err) => {
      console.error(err);
      uni.showToast({ title: '网络请求失败', icon: 'none' });
      processData([]); // 失败也显示空页面
    },
    complete: () => {
      uni.hideLoading();
    }
  });
};

// 数据处理逻辑
const processData = (list) => {
  let newList = [];
  let newTotal = null;

  if (list && list.length > 0) {
    list.forEach(item => {
      if (item.name === 'TOTAL') {
        newTotal = item;
      } else {
        newList.push(item);
      }
    });
  }

  // 如果没有 TOTAL 数据，手动生成 0 数据
  if (!newTotal) {
    newTotal = { name: 'TOTAL', xiaoji: 0, zhengshu: 0, changshu: 0 };
  }

  listData.value = newList;
  totalData.value = newTotal;
};

// 年份选择回调
const onYearChange = (e) => {
  const index = e.detail.value; // 获取选中的索引，比如 0, 1, 2...
  const selectedYear = yearList.value[index]; // 从数组里拿到年份字符串

  // 如果年份变了，才重新请求
  if (selectedYear !== currentYear.value) {
    yearIndex.value = index;      // 更新索引
    currentYear.value = selectedYear; // 更新显示的年份
    getData(); // 发送请求
  }
};

const goDetail = (item) => {
  // 携带 name 和 time (当前选中的年份)
  uni.navigateTo({
    url: `/pages/detail/detail?name=${item.name}&time=${currentYear.value}`
  });
};

// 录入数据按钮点击
const handleRecord = () => {
  // 跳转到录入页面
  uni.navigateTo({
    url: '/pages/input/input'
  });
};

// 跳转到年度明细表
const goYearTable = () => {
  uni.navigateTo({
    url: `/pages/yearTable/yearTable?time=${currentYear.value}`
  });
};
</script>

<style lang="scss" scoped>

.container {
  min-height: 100vh;
  background-color: #f5f5f5;
  padding-bottom: 30rpx;
}

.header {
  width: 100%;
  background: linear-gradient(135deg, #4a90e2, #007aff);
  color: #fff;
  padding: 30rpx 40rpx 50rpx 40rpx;
  position: relative;
  box-sizing: border-box;
  margin-bottom: 20rpx;
  border-bottom-left-radius: 30rpx;
  border-bottom-right-radius: 30rpx;
  box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.1);

  .header-row-top {
    margin-bottom: 30rpx;
    font-size: 32rpx;
    font-weight: bold;
    
    .year-selector {
      display: inline-flex;
      align-items: center;
      padding: 10rpx 20rpx;
      background-color: rgba(255, 255, 255, 0.2);
      border-radius: 30rpx;
      
      .arrow-icon {
        margin-left: 10rpx;
        font-size: 24rpx;
        transform: scaleY(0.8);
      }
    }
  }

  .header-row-bottom {
    display: flex;
    justify-content: space-between;
    padding-right: 180rpx;
    
    .stat-item {
      display: flex;
      flex-direction: column;
      
      .label {
        font-size: 24rpx;
        opacity: 0.9;
        margin-bottom: 8rpx;
      }
      
      .value {
        font-size: 40rpx;
        font-weight: bold;
      }
	  &.clickable-item {
	      position: relative;
	      padding-right: 20rpx;
	      
	      &:active {
	        opacity: 0.8;
	        transform: scale(0.98);
	      }
	  
	      .arrow {
	        position: absolute;
	        right: -20rpx;
	        top: 50%;
	        transform: translateY(-50%);
	        font-size: 24rpx;
	        opacity: 0.6;
	      }
	    }
    }
  }

  .record-btn {
    position: absolute;
    right: 30rpx;
    bottom: 20rpx;
    background-color: #fff;
    color: #007aff;
    padding: 12rpx 24rpx;
    border-radius: 10rpx;
    font-size: 26rpx;
    font-weight: bold;
    box-shadow: 0 2rpx 6rpx rgba(0,0,0,0.2);
    
    &:active {
      opacity: 0.8;
      transform: scale(0.98);
    }
  }
}

.loading-placeholder, .empty-state {
  padding: 100rpx 0;
  text-align: center;
  color: #999;
  font-size: 28rpx;
}

.grid-container {
  display: flex;
  flex-wrap: wrap;
  padding: 0 10rpx;
}

.grid-item {
  width: 33.33%;
  box-sizing: border-box;
  padding: 10rpx 10rpx 30rpx 10rpx; 

  .item-content {
    background-color: #fff;
    border-radius: 12rpx;
    padding: 15rpx 10rpx;
    display: flex;
    align-items: center;
    box-shadow: 0 2rpx 5rpx rgba(0,0,0,0.05);
    height: 100%; 

    .name-square {
      width: 70rpx;
      height: 70rpx;
      background-color: #eef2f9;
      color: #333;
      font-weight: bold;
      font-size: 24rpx;
      border-radius: 8rpx;
      display: flex;
      align-items: center;
      justify-content: center;
      margin-right: 10rpx;
      flex-shrink: 0; 
      word-break: break-all;
      text-align: center;
      line-height: 1.1;
    }

    .info-list {
      display: flex;
      flex-direction: column;
      justify-content: center;
      
      text {
        font-size: 20rpx;
        color: #666;
        margin-bottom: 2rpx;
        white-space: nowrap;
        
        &:last-child {
          margin-bottom: 0;
          font-weight: bold;
          color: #333;
        }
      }
    }
  }
}
</style>