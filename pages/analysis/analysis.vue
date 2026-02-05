<template>
  <view class="container">
    
    <!-- 1. 顶部标题 -->
    <view class="page-header">
      <text class="title">类型统计</text>
      <text class="sub-title">{{ currentTime }}年 </text>
    </view>

    <!-- 2. 统计概览 (六个格子) -->
    <view class="stat-grid" v-if="totalStat">
      <!-- 我们手动定义顺序，方便遍历渲染 -->
      <view 
        class="stat-item" 
        v-for="(key, index) in statKeys" 
        :key="index"
      >
        <view class="stat-card">
          <text class="stat-label">{{ key }}</text>
          <text class="stat-num" :class="getNumClass(key)">{{ totalStat[key] || 0 }}</text>
        </view>
      </view>
    </view>

    <!-- 加载中 -->
    <view v-if="isLoading" class="loading-box">
      <text>数据计算中...</text>
    </view>

    <!-- 3. 列表区域 -->
    <view class="list-container" v-if="listData.length > 0">
      <view class="list-header">
        <text class="th col-1">名称</text>
        <text class="th col-2">次数</text>
        <text class="th col-3">比例</text>
        <text class="th col-4">总数</text>
      </view>

      <view 
        class="list-row" 
        v-for="(item, index) in listData" 
        :key="index"
      >
        <!-- 数据格式是数组: ["LH", "9", "77.8%", "4000"] -->
        <view class="td col-1 bold">{{ item[0] }}</view>
        <view class="td col-2">{{ item[1] }}</view>
        <view class="td col-3 highlight">{{ item[2] }}</view>
        <view class="td col-4" :class="getTotalClass(item[3])">{{ item[3] }}</view>
      </view>
    </view>

  </view>
</template>

<script setup>
import { ref } from 'vue';
import { onLoad } from '@dcloudio/uni-app';

// 状态变量
const currentTime = ref('');
const currentTypeId = ref('');
const listData = ref([]);
const totalStat = ref(null);
const isLoading = ref(false);

// 定义那六个字段的 key，确保顺序
const statKeys = ['1000+', '1000-', '2000+', '2000-', '3000+', '3000-'];

onLoad((options) => {
  if (options.time && options.typeId) {
    currentTime.value = options.time;
    currentTypeId.value = options.typeId;
    getData();
  }
});

const getData = () => {
  isLoading.value = true;
  
  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/GetOtherTotal',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentTime.value,
      typeId: currentTypeId.value
    },
    success: (res) => {
      let data = res.data;
      if (typeof data === 'string') {
        try { data = JSON.parse(data); } catch(e){}
      }

      if (data && !data.isError) {
        // 1. 处理 list
        listData.value = data.list || [];
        
        // 2. 处理 total (接口返回的是 total:[{...}])，取数组第一个对象
        if (data.total && data.total.length > 0) {
          totalStat.value = data.total[0];
        }
      } else {
        uni.showToast({ title: '获取数据失败', icon: 'none' });
      }
    },
    fail: () => {
      uni.showToast({ title: '网络错误', icon: 'none' });
    },
    complete: () => {
      isLoading.value = false;
    }
  });
};

// --- 样式辅助函数 ---

// 根据 key 判断颜色 (带+号的用红色，带-号的用绿色)
const getNumClass = (key) => {
  if (key.includes('+')) return 'red-text';
  if (key.includes('-')) return 'green-text';
  return '';
};

// 列表最后一列 总数 的颜色
const getTotalClass = (valStr) => {
  const num = parseInt(valStr);
  if (num === null || num === undefined) return 'text-gray';
  // 修改点：正数蓝，负数红
  if (num > 0) return 'text-blue';   
  if (num < 0) return 'text-red'; 
  return 'text-black'; 
};
</script>

<style lang="scss" scoped>
.container {
  min-height: 100vh;
  background-color: #f5f7fa;
  padding: 30rpx;
}

/* 顶部标题 */
.page-header {
  margin-bottom: 30rpx;
  .title { font-size: 36rpx; font-weight: bold; color: #333; margin-right: 20rpx; }
  .sub-title { font-size: 24rpx; color: #999; }
}

/* --- 上方六宫格统计 --- */
.stat-grid {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  margin-bottom: 30rpx;
}

.stat-item {
  width: 48%; /* 一行两个，留点缝隙 */
  margin-bottom: 20rpx;
}

.stat-card {
  background-color: #fff;
  border-radius: 12rpx;
  padding: 24rpx;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2rpx 10rpx rgba(0,0,0,0.03);

  .stat-label {
    font-size: 28rpx;
    color: #666;
    font-weight: bold;
  }

  .stat-num {
    font-size: 32rpx;
    font-weight: bold;
  }
}

/* --- 下方列表 --- */
.list-container {
  background-color: #fff;
  border-radius: 16rpx;
  overflow: hidden;
  box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.05);
}

.list-header, .list-row {
  display: flex;
  padding: 24rpx 20rpx;
  align-items: center;
  text-align: center;
}

.list-header {
  background-color: #eef2f9;
  border-bottom: 1rpx solid #e1e5eb;
  .th { font-size: 26rpx; font-weight: bold; color: #333; }
}

.list-row {
  border-bottom: 1rpx solid #f0f0f0;
  &:last-child { border-bottom: none; }
  .td { font-size: 26rpx; color: #666; }
}

/* 列宽分配 (一共100%) */
.col-1 { width: 20%; text-align: left; padding-left: 10rpx; } /* 名称 */
.col-2 { width: 20%; } /* 次数 */
.col-3 { width: 25%; } /* 比例 */
.col-4 { width: 35%; text-align: right; padding-right: 10rpx; } /* 总数 */

/* 字体样式 */
.bold { font-weight: bold; color: #333; }
.highlight { color: #007aff; background-color: rgba(0,122,255,0.05); padding: 4rpx 8rpx; border-radius: 8rpx; }
.red-text { color: #e74c3c; }
.green-text { color: #27ae60; }

.loading-box { text-align: center; padding: 30rpx; color: #999; }
.text-blue { color: #007aff !important; }
.text-red { color: #ff3b30 !important; }
.text-gray { color: #ccc !important; }
.text-black { color: #333 !important; }
</style>