<template>
  <view class="container">
    
    <view class="page-title">
      <text>{{ currentTime }}年度数据明细表</text>
      <text class="sub-text">← 左右滑动查看更多类型 →</text>
    </view>

    <!-- 表格区域 -->
    <view class="table-wrapper" v-if="tableData.length > 0">
      <scroll-view scroll-x class="scroll-view" :scroll-left="scrollLeft">
        
        <!-- 1. 表头 -->
        <view class="header-row">
          <view 
            class="th" 
            v-for="(col, index) in columns" 
            :key="index"
            :class="{ 
              'col-date': col === 'Date', 
              'col-total': col === 'TOTAL' 
            }"
          >
            {{ getColName(col) }}
          </view>
        </view>

        <!-- 2. 数据行 -->
        <view 
          class="data-row" 
          v-for="(row, rIndex) in tableData" 
          :key="rIndex"
          :class="{ 'stripe': rIndex % 2 === 1 }"
        >
          <view 
            class="td" 
            v-for="(col, cIndex) in columns" 
            :key="cIndex"
            :class="{ 
              'col-date': col === 'Date', 
              'col-total': col === 'TOTAL',
              'text-blue': isNumber(row[col]) && row[col] > 0,
              'text-red': isNumber(row[col]) && row[col] < 0
            }"
          >
            {{ formatValue(row[col]) }}
          </view>
        </view>

      </scroll-view>
    </view>

    <view v-if="isLoading" class="loading-box">
      <text>加载数据中...</text>
    </view>
    <view v-if="!isLoading && tableData.length === 0" class="empty-box">
      <text>暂无数据</text>
    </view>

  </view>
</template>

<script setup>
import { ref } from 'vue';
import { onLoad } from '@dcloudio/uni-app';

const currentTime = ref('');
const tableData = ref([]);
const columns = ref([]); 
const isLoading = ref(false);
const scrollLeft = ref(0); 

onLoad((options) => {
  if (options.time) {
    currentTime.value = options.time;
    getData();
  }
});

const getData = () => {
  isLoading.value = true;
  tableData.value = [];

  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/GetDetailByYear',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: { time: currentTime.value },
    success: (res) => {
      let data = res.data;
      if (typeof data === 'string') {
        try { data = JSON.parse(data); } catch(e){}
      }
      if (data && !data.isError && data.dtMain) {
        processData(data.dtMain);
      } else {
        uni.showToast({ title: '暂无数据', icon: 'none' });
      }
    },
    fail: () => { uni.showToast({ title: '请求失败', icon: 'none' }); },
    complete: () => { isLoading.value = false; }
  });
};

const processData = (list) => {
  if (list.length === 0) return;
  const allKeys = Object.keys(list[0]);
  const dynamicKeys = allKeys.filter(key => key !== 'DateID' && key !== 'Date' && key !== 'TOTAL');
  
  // 保证顺序：Date -> TOTAL -> 其他
  columns.value = ['Date', 'TOTAL', ...dynamicKeys];
  tableData.value = list;
  scrollLeft.value = 0; 
};

const getColName = (key) => {
  if (key === 'Date') return '日期';
  if (key === 'TOTAL') return '小计';
  return key;
};

const formatValue = (val) => {
  if (val === null || val === undefined) return '-';
  return val;
};

const isNumber = (val) => {
  return typeof val === 'number';
};
</script>

<style lang="scss" scoped>
/* 定义列宽变量，方便统一修改 */
$date-width: 180rpx;
$total-width: 140rpx;
$normal-width: 140rpx;

.container {
  min-height: 100vh;
  background-color: $bg-color-page; /* 使用公共背景色变量 #f5f7fa */
  padding: 30rpx 0; /* 保持上下有间距，但左右铺满 */
}

.page-title {
  padding: 0 30rpx 30rpx 30rpx; 
  text-align: center; 
  @include flex-col; /* 使用公共Mixin */
  
  text:first-child { 
    font-size: 34rpx; 
    font-weight: bold; 
    color: $text-color-main; /* 使用公共深色字 #333 */
  }
  
  .sub-text { 
    font-size: 22rpx; 
    color: $text-color-sub; /* 使用公共灰色字 #999 */
    margin-top: 10rpx; 
  }
}

.table-wrapper {
  background-color: #fff;
  border-top: 1rpx solid #eee;
  border-bottom: 1rpx solid #eee;
}

.scroll-view { 
  width: 100%; 
  white-space: nowrap; 
}

.header-row, .data-row {
  display: flex;
  min-width: 100%;
  width: max-content;
}

/* 表头吸顶 (上下滚动时固定头部) */
.header-row {
  background-color: #eef2f9; /* 保持表格特有的淡蓝灰表头 */
  position: sticky; 
  top: 0;
  z-index: 50; /* 层级最高 */
  border-bottom: 1rpx solid #e1e5eb;
  
  .th { 
    font-weight: bold; 
    color: $text-color-main; 
    font-size: 26rpx; 
  }
}

.data-row {
  border-bottom: 1rpx solid #f0f0f0;
  
  &.stripe { 
    background-color: #fafafa; 
  }
  
  .td { 
    font-size: 26rpx; 
    color: $text-color-main; 
  }
}

/* 通用单元格 */
.th, .td {
  display: inline-block;
  width: $normal-width;
  padding: 24rpx 10rpx;
  text-align: center;
  box-sizing: border-box;
  vertical-align: middle;
}

/* --- 核心修改：两列双冻结逻辑 --- */

/* 1. 日期列 (第一列) */
.col-date {
  position: sticky;
  left: 0; 
  width: $date-width; 
  z-index: 20; 
  border-right: 1rpx solid #eee;
}

/* 2. 小计列 (第二列) */
.col-total {
  position: sticky;
  left: $date-width; /* 核心：固定位置是第一列的宽度 */
  width: $total-width;
  z-index: 20;
  border-right: 1rpx dashed #ccc; /* 明显的分界线 */
  font-weight: bold;
}

/* 3. 背景色修正 (防止透明叠加) */
/* 表头部分 */
.header-row .col-date,
.header-row .col-total {
  background-color: #eef2f9; /* 必须和表头背景一致 */
  z-index: 60; /* 表头的固定列层级要最高 */
}

/* 数据行部分 */
.data-row:not(.stripe) .col-date,
.data-row:not(.stripe) .col-total {
  background-color: #fff;
}
.data-row.stripe .col-date,
.data-row.stripe .col-total {
  background-color: #fafafa;
}

/* --- 数字颜色修改 --- */
/* 正数：蓝色 */
.text-blue { 
  color: $text-color-blue !important; 
  font-weight: bold; 
}

/* 负数：红色 */
.text-red { 
  color: $text-color-red !important; 
  font-weight: bold; 
}

.loading-box, .empty-box { 
  text-align: center; 
  padding: 100rpx; 
  color: $text-color-sub; 
}
</style>