<template>
  <view class="container">
    
    <!-- 1. 顶部标题 (修复点：去掉 uni-icons，换成普通箭头符号) -->
    <view class="header-btn" @click="goToAnalysis">
      <view class="header-left">
        <text class="main-text">大数据分析 (点击查看统计)</text>
        <text class="sub-text">当前类型ID: {{ currentTypeId || '-' }} / {{ currentTime }}年</text>
      </view>
      <!-- 用 css 画的箭头或者符号 -->
      <text class="arrow-right">></text>
    </view>

    <!-- 2. 主页面表格区域 -->
    <view class="table-container">
      <view class="table-header">
        <view class="th date-col">日期</view>
        <view class="th num-col">数量</view>
      </view>

      <view v-if="tableData.length > 0">
        <view 
          class="table-row clickable" 
          v-for="(item, index) in tableData" 
          :key="index"
          :class="{ 'stripe': index % 2 === 1 }" 
          @click="handleDateClick(item)"
        >
          <view class="td date-col">{{ item.日期 }}</view>
          <view class="td num-col" :class="getNumClass(item.数量)">
            {{ item.数量 !== null ? item.数量 : '-' }}
          </view>
        </view>
      </view>

      <view v-else-if="!isLoading" class="empty-box">
        <text>暂无数据</text>
      </view>
    </view>
    
    <!-- 加载中提示 -->
    <view v-if="isLoading" class="loading-box">
      <text>加载中...</text>
    </view>

    <!-- 3. 详情弹窗 (保持不变) -->
    <view class="modal-mask" v-if="showModal" @click.self="closeModal">
      <view class="modal-content">
        <view class="modal-header">
          <text class="modal-title">{{ selectedDate }} 明细</text>
          <text class="close-btn" @click="closeModal">×</text>
        </view>

        <view class="modal-body">
          <view v-if="modalLoading" class="modal-loading">
            <text>加载中...</text>
          </view>
          
          <view v-else class="modal-table">
            <view class="modal-row header-row">
              <text class="col-type">类型</text>
              <text class="col-val">数量</text>
            </view>
            <view v-if="modalData.length > 0">
              <view 
                class="modal-row" 
                v-for="(mItem, mIndex) in modalData" 
                :key="mIndex"
                :class="{ 'total-row': mItem.isTotal }"
              >
                <text class="col-type">{{ mItem.displayName }}</text>
                <text class="col-val" :class="getNumClass(mItem.xiaoji)">{{ mItem.xiaoji }}</text>
              </view>
            </view>
            <view v-else class="empty-mini">
              <text>该日期无有效数据</text>
            </view>
          </view>
        </view>
        
        <view class="modal-footer">
          <view class="confirm-btn" @click="closeModal">关闭</view>
        </view>
      </view>
    </view>

  </view>
</template>

<script setup>
import { ref } from 'vue';
import { onLoad } from '@dcloudio/uni-app';

// --- 主页面状态 ---
const currentName = ref('');
const currentTime = ref('');
const tableData = ref([]);
const isLoading = ref(false);
const currentTypeId = ref(''); // 新增：存储 TypeID

// --- 弹窗状态 ---
const showModal = ref(false);
const modalLoading = ref(false);
const selectedDate = ref('');
const modalData = ref([]);

onLoad((options) => {
  if (options.name && options.time) {
    currentName.value = options.name;
    currentTime.value = options.time;
    getDetailData();
  }
});

// 主页面接口
const getDetailData = () => {
  isLoading.value = true;
  tableData.value = []; 

  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/GetTypeToTalByName',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: { time: currentTime.value, name: currentName.value },
    success: (res) => {
      let responseData = parseJsonSafe(res.data);
      
      if (responseData && !responseData.isError && responseData.result) {
        // --- 修复点：确保变量定义正确 ---
        
        // 1. 先过滤出有效数据
        const validList = responseData.result.filter(item => item.数量 !== null);
        
        // 2. 赋值给表格数据
        tableData.value = validList;

        // 3. 获取 TypeID (如果列表有数据，取第一条的ID)
        if (validList.length > 0) {
            currentTypeId.value = validList[0].TypeID;
        }
      } else {
        uni.showToast({ title: '暂无数据', icon: 'none' });
      }
    },
    fail: (err) => { uni.showToast({ title: '请求失败', icon: 'none' }); },
    complete: () => { isLoading.value = false; }
  });
};

// --- 新增：跳转到分析页面 ---
const goToAnalysis = () => {
  if (!currentTypeId.value) {
    uni.showToast({ title: '暂无数据或类型ID', icon: 'none' });
    return;
  }
  // 跳转
  uni.navigateTo({
    url: `/pages/analysis/analysis?time=${currentTime.value}&typeId=${currentTypeId.value}`
  });
};

// 点击日期处理逻辑 (保持不变)
const handleDateClick = (item) => {
  if (!item.日期) return;
  selectedDate.value = item.日期;
  showModal.value = true;
  getModalData(item.日期);
};

// 获取弹窗数据 (保持不变)
const getModalData = (dateStr) => {
  modalLoading.value = true;
  modalData.value = [];

  uni.request({
    url: 'http://13.94.38.44:8080/TypeList/GetTypeToTal',
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: { time: dateStr },
    success: (res) => {
      let responseData = parseJsonSafe(res.data);

      if (responseData && !responseData.isError && responseData.result) {
        const rawList = responseData.result;
        const filteredList = [];

        rawList.forEach(obj => {
          if (obj.xiaoji === 0) return;
          let finalName = obj.name;
          let isTotal = false;
          if (obj.name === 'TOTAL') {
            finalName = '加总';
            isTotal = true;
          }
          filteredList.push({
            displayName: finalName,
            xiaoji: obj.xiaoji,
            isTotal: isTotal
          });
        });
        modalData.value = filteredList;
      }
    },
    fail: () => { uni.showToast({ title: '详情获取失败', icon: 'none' }); },
    complete: () => { modalLoading.value = false; }
  });
};

const closeModal = () => {
  showModal.value = false;
};

const parseJsonSafe = (data) => {
  if (typeof data === 'string') {
    try { return JSON.parse(data); } catch (e) { return null; }
  }
  return data;
};

const getNumClass = (num) => {
  if (num === null) return 'gray';
  if (num > 0) return 'red';   
  if (num < 0) return 'green'; 
  return '';
};
</script>

<style lang="scss" scoped>
.container {
  min-height: 100vh;
  background-color: #f5f7fa;
  padding: 30rpx;
}

/* 头部按钮样式 (修复后) */
.header-btn {
  background: linear-gradient(90deg, #4a90e2, #67b26f);
  padding: 30rpx;
  border-radius: 16rpx;
  margin-bottom: 30rpx;
  box-shadow: 0 4rpx 12rpx rgba(74, 144, 226, 0.3);
  display: flex;
  justify-content: space-between;
  align-items: center;

  .header-left {
    display: flex;
    flex-direction: column;
  }

  .main-text {
    font-size: 32rpx;
    font-weight: bold;
    color: #fff;
    margin-bottom: 6rpx;
  }
  
  .sub-text {
    font-size: 24rpx;
    color: rgba(255,255,255,0.8);
  }
  
  .arrow-right {
    color: #fff;
    font-size: 36rpx;
    font-weight: bold;
    opacity: 0.8;
  }
  
  &:active {
    opacity: 0.9;
    transform: scale(0.98);
  }
}

.table-container { background-color: #fff; border-radius: 16rpx; overflow: hidden; box-shadow: 0 4rpx 20rpx rgba(0,0,0,0.05); }
.table-header, .table-row { display: flex; padding: 24rpx 30rpx; align-items: center; }
.table-header { background-color: #eef2f9; border-bottom: 1rpx solid #e1e5eb; .th { font-size: 28rpx; font-weight: bold; color: #333; } }
.table-row { border-bottom: 1rpx solid #f0f0f0; &.clickable:active { background-color: #f0f0f0; } &.stripe { background-color: #fafafa; } .td { font-size: 28rpx; color: #666; } }
.date-col { width: 60%; text-align: left; }
.num-col { width: 40%; text-align: right; }
.red { color: #e74c3c; font-weight: bold; }
.green { color: #27ae60; font-weight: bold; }
.gray { color: #ccc; }
.loading-box, .empty-box { text-align: center; padding: 50rpx 0; color: #999; font-size: 26rpx; }

/* 弹窗样式 */
.modal-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background-color: rgba(0,0,0,0.6); z-index: 999; display: flex; align-items: center; justify-content: center; }
.modal-content { width: 80%; background-color: #fff; border-radius: 20rpx; overflow: hidden; display: flex; flex-direction: column; box-shadow: 0 10rpx 30rpx rgba(0,0,0,0.2); }
.modal-header { padding: 30rpx; border-bottom: 1rpx solid #eee; display: flex; justify-content: space-between; align-items: center; .modal-title { font-size: 32rpx; font-weight: bold; color: #333; } .close-btn { font-size: 40rpx; color: #999; padding: 0 10rpx; line-height: 1; } }
.modal-body { padding: 20rpx 30rpx; max-height: 60vh; overflow-y: auto; }
.modal-loading { text-align: center; padding: 40rpx; color: #999; }
.empty-mini { text-align: center; padding: 30rpx; color: #ccc; font-size: 24rpx; }
.modal-table { border: 1rpx solid #eee; border-radius: 10rpx; overflow: hidden; }
.modal-row { display: flex; justify-content: space-between; padding: 20rpx; border-bottom: 1rpx solid #eee; font-size: 28rpx; &:last-child { border-bottom: none; } &.header-row { background-color: #f8f8f8; font-weight: bold; color: #666; } &.total-row { background-color: #fff8e1; .col-type { font-weight: bold; color: #333; } } }
.col-type { color: #333; } .col-val { font-weight: bold; }
.modal-footer { padding: 20rpx 30rpx 30rpx 30rpx; .confirm-btn { background-color: #4a90e2; color: #fff; text-align: center; padding: 20rpx; border-radius: 10rpx; font-size: 28rpx; &:active { opacity: 0.9; } } }
</style>