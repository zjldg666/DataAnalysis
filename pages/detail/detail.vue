<template>
  <view class="page-container">
    
    <!-- 1. 顶部头部区域 (黑白风格) -->
        <view class="header-section"> 
          <!-- 第一行：可点击的标题 (带右箭头) -->
          <view class="analysis-link" @click="goToAnalysis">
            <text class="link-text">大数据分析 (点击查看统计)</text>
            <text class="link-arrow">></text>
          </view>
    
          <!-- 第二行：信息展示 -->
          <view class="info-row">
            <text class="info-label">当前类型:</text>
            <view class="info-badge">{{ currentName }}</view>
            <text class="info-divider">/</text>
            <text class="info-year">{{ currentTime }}年</text>
          </view>
    
        </view>

    <!-- 2. 主列表区域 -->
    <view class="list-container">
      <!-- 表头 -->
      <view class="list-header">
        <text class="th-left">日期</text>
        <text class="th-right">数量</text>
      </view>

      <!-- 列表内容 -->
      <view v-if="tableData.length > 0">
        <view 
          class="list-item" 
          v-for="(item, index) in tableData" 
          :key="index"
          @click="handleDateClick(item)"
        >
          <!-- 左侧：日期 -->
          <text class="date-text">{{ item.日期 }}</text>
          
          <!-- 右侧：数量 (带颜色) -->
          <text class="num-text" :class="getNumClass(item.数量)">
            {{ item.数量 !== null ? item.数量 : '-' }}
          </text>
          
          <!-- 小箭头暗示可点 -->
          <text class="item-arrow">></text>
        </view>
      </view>
      
      <!-- 空状态 -->
      <view v-else-if="!isLoading" class="empty-state">
        <text>暂无数据</text>
      </view>
    </view>
    
    <!-- 加载中 -->
    <view v-if="isLoading" class="loading-state">
      <text>加载中...</text>
    </view>

    <!-- 3. 详情弹窗 (风格微调) -->
    <view class="modal-mask" v-if="showModal" @click.self="closeModal">
      <view class="modal-card">
        <view class="modal-header">
          <text class="modal-title">{{ selectedDate }} 每日明细</text>
          <view class="close-icon" @click="closeModal">×</view>
        </view>

        <scroll-view scroll-y class="modal-body">
          <view v-if="modalLoading" class="loading-state">
            <text>加载中...</text>
          </view>
          
          <view v-else class="detail-list">
            <view class="detail-header">
              <text>类型</text>
              <text>小计</text>
            </view>
            
            <view v-if="modalData.length > 0">
              <view 
                class="detail-row" 
                v-for="(mItem, mIndex) in modalData" 
                :key="mIndex"
                :class="{ 'total-row': mItem.isTotal }"
              >
                <text class="row-name">{{ mItem.displayName }}</text>
                <text class="row-val" :class="getNumClass(mItem.xiaoji)">{{ mItem.xiaoji }}</text>
              </view>
            </view>
            <view v-else class="empty-mini">
              <text>该日期无有效数据</text>
            </view>
          </view>
        </scroll-view>
        
        <view class="modal-footer">
          <view class="close-btn" @click="closeModal">关闭</view>
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
const currentTypeId = ref(''); // 存储 TypeID
const selectedQty = ref(null); // 保存选中日期的数值
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
  selectedQty.value = item.数量; // 【关键】记录这一行的数量，用于后续匹配
  
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
        
        // --- 步骤1：将扁平数组切分为多个组 ---
        // 逻辑：遇到 name 为 TOTAL 就截断
        let allGroups = [];
        let tempGroup = [];
        
        rawList.forEach(item => {
          tempGroup.push(item);
          if (item.name === 'TOTAL') {
            allGroups.push(tempGroup); // 存入一组
            tempGroup = []; // 清空，准备下一组
          }
        });

        // --- 步骤2：找到匹配的那一组 ---
        // 目标：找到那个组里，有一条数据的 name 等于当前类型(currentName)，且数量等于点击数量(selectedQty)
        let targetGroup = [];
        
        // 遍历所有组寻找
        targetGroup = allGroups.find(group => {
          return group.some(gItem => {
            // 注意数值比较可能存在的类型差异，稍微转一下 string 或 number 比较更稳
            // currentName.value 就是你在列表页选的类型，比如 "ZZ"
            return gItem.name === currentName.value && gItem.xiaoji === selectedQty.value;
          });
        });

        // 如果没找到（极少情况），默认显示第一组或者空
        if (!targetGroup) {
             // 容错：如果找不到精准匹配的，就暂且不显示，或者显示全部(看你需求)
             targetGroup = []; 
        }

        // --- 步骤3：处理显示数据 (过滤0，改名) ---
        const filteredList = [];
        targetGroup.forEach(obj => {
          if (obj.xiaoji === 0) return; // 过滤 0
          
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

// 数字颜色判断逻辑
const getNumClass = (num) => {
  if (num === null || num === undefined) return 'text-gray';
  // 修改点：正数蓝，负数红
  if (num > 0) return 'text-blue';   
  if (num < 0) return 'text-red'; 
  return 'text-black'; // 0 显示黑色
};
</script>

<style lang="scss">
/* 全局底色 */
.page-container {
  min-height: 100vh;
  background-color: #f2f3f5; /* 统一灰底 */
  padding-bottom: 40rpx;
}

/* --- 1. 头部区域 (仿 Index 风格) --- */

.header-section {
  background-color: #1a1a1a;
  /* 上下左右留出足够的呼吸空间 */
  padding: 30rpx 40rpx 40rpx;
  border-bottom-left-radius: 30rpx;
  border-bottom-right-radius: 30rpx;
  
  display: flex;
  flex-direction: column; /* 改为垂直排列 */
  gap: 20rpx; /* 两行之间的间距 */
  
  box-shadow: 0 4rpx 12rpx rgba(0,0,0,0.1);
}

/* 第一行：跳转链接 */
.analysis-link {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.1); /* 淡淡的背景框 */
  padding: 16rpx 24rpx;
  border-radius: 12rpx;
  border: 1px solid rgba(255, 255, 255, 0.2); /* 细微边框 */
  
  &:active {
    background-color: rgba(255, 255, 255, 0.2);
  }
}

.link-text {
  font-size: 28rpx;
  color: #fff;
  font-weight: bold;
}

.link-arrow {
  font-size: 28rpx;
  color: #ff9900; /* 橙色箭头，醒目一点 */
  font-weight: bold;
}

/* 第二行：信息展示 */
.info-row {
  display: flex;
  align-items: center;
  /* 稍微缩进一点，更有层次感 */
  padding-left: 10rpx; 
}

.info-label {
  font-size: 24rpx;
  color: rgba(255, 255, 255, 0.6);
  margin-right: 12rpx;
}

.info-badge {
  background-color: #ffffff;
  color: #000000;
  font-size: 26rpx;
  font-weight: bold;
  padding: 4rpx 12rpx;
  border-radius: 8rpx;
}

.info-divider {
  color: rgba(255, 255, 255, 0.4);
  margin: 0 16rpx;
  font-size: 24rpx;
}

.info-year {
  font-size: 30rpx;
  color: #ffffff;
  font-weight: bold;
}

/* --- 2. 列表区域 --- */
.list-container {
  margin: 20rpx; /* 留边距 */
  background-color: #fff;
  border-radius: 16rpx;
  overflow: hidden;
  box-shadow: 0 2rpx 8rpx rgba(0,0,0,0.05);
}

.list-header {
  background-color: #f8f8f8;
  padding: 20rpx 30rpx;
  display: flex;
  justify-content: space-between;
  border-bottom: 1rpx solid #eee;
  
  text {
    font-size: 26rpx;
    color: #999;
    font-weight: bold;
  }
}

.list-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx 30rpx;
  border-bottom: 1rpx solid #f5f5f5;
  
  &:active {
    background-color: #fafafa;
  }
  
  &:last-child {
    border-bottom: none;
  }
}

.date-text {
  font-size: 28rpx;
  color: #333;
  flex: 1;
}

.num-text {
  font-size: 30rpx;
  font-weight: bold;
  margin-right: 16rpx; /* 给箭头留位置 */
}

.item-arrow {
  font-size: 24rpx;
  color: #ccc;
}

/* --- 颜色类 --- */
.text-blue { color: #007aff !important; }
.text-red { color: #ff3b30 !important; }
.text-gray { color: #ccc !important; }
.text-black { color: #333 !important; }

/* 状态提示 */
.loading-state, .empty-state {
  text-align: center;
  padding: 60rpx 0;
  color: #999;
  font-size: 26rpx;
}

/* --- 3. 弹窗样式 --- */
.modal-mask {
  position: fixed; top: 0; left: 0; right: 0; bottom: 0;
  background-color: rgba(0,0,0,0.6);
  z-index: 999;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-card {
  width: 80%;
  background-color: #fff;
  border-radius: 24rpx;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 70vh;
}

.modal-header {
  padding: 30rpx;
  border-bottom: 1rpx solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #fafafa;
}

.modal-title {
  font-size: 30rpx;
  font-weight: bold;
  color: #333;
}

.close-icon {
  font-size: 40rpx;
  color: #999;
  line-height: 1;
  padding: 0 10rpx;
}

/* 弹窗主体内容区 */
.modal-body {
  padding: 0 30rpx;
  overflow-y: auto;
  /* 确保主体不超出父容器宽度 */
  width: 100%; 
  box-sizing: border-box; 
}

.detail-list {
  padding: 20rpx 0;
  width: 100%;
}

/* 表头：类型 | 小计 */
.detail-header {
  display: flex;
  justify-content: space-between;
  padding-bottom: 16rpx;
  border-bottom: 2rpx solid #eee;
  margin-bottom: 10rpx;
  width: 100%;
  
  text { 
    font-size: 24rpx; 
    color: #999; 
    font-weight: bold; 
  }
}

/* 列表行 */
.detail-row {
  display: flex;
  justify-content: space-between;
  align-items: center; /* 垂直居中 */
  padding: 16rpx 0;
  border-bottom: 1rpx solid #f9f9f9;
  font-size: 28rpx;
  width: 100%; /* 强制占满 */
  
  &.total-row {
    margin-top: 10rpx;
    background-color: #f0f0f0;
    padding: 16rpx 10rpx; /* 加点内边距 */
    border-radius: 8rpx;
    font-weight: bold;
    /* 总计行加了padding，宽度要减去padding，防止撑破 */
    width: calc(100% - 20rpx); 
    margin-left: auto;
    margin-right: auto;
  }
}

/* 名字列：限制宽度，超出省略 */
.row-name { 
  color: #333; 
  flex: 1; /* 占据剩余空间 */
  margin-right: 20rpx; /* 离右边数字远一点 */
  
  /* 防止长文字撑开 */
  white-space: nowrap; 
  overflow: hidden;
  text-overflow: ellipsis;
}

/* 数值列：不换行 */
.row-val { 
  font-weight: 600; 
  flex-shrink: 0; /* 防止被挤压 */
  text-align: right;
  min-width: 80rpx; /* 给个最小宽度对齐 */
}

.modal-footer {
  padding: 20rpx 30rpx 30rpx;
}

.close-btn {
  background-color: #333; /* 黑钮 */
  color: #fff;
  text-align: center;
  padding: 20rpx;
  border-radius: 12rpx;
  font-size: 28rpx;
  font-weight: 500;
  
  &:active { opacity: 0.9; }
}

.empty-mini { text-align: center; padding: 40rpx; color: #ccc; font-size: 24rpx; }
</style>