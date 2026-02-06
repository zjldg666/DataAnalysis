<template>
  <view class="page-container">
    
    <!-- 头部区域：严格的三行布局 -->
    <view class="header-section" v-if="totalData">
      <!-- 第一行：年份选择 -->
      <view class="header-row year-row">
        <picker 
          mode="selector" 
          :range="yearList" 
          :value="yearIndex" 
          @change="onYearChange"
        >
          <view class="year-picker-box">
            <text class="year-text">{{ currentYear }}年</text>
            <text class="uni-icon-arrow">▼</text>
          </view>
        </picker>
      </view>

      <!-- 第二行：核心数据展示 -->
      <view class="header-row stats-row">
        <!-- 左侧：加总数 (保持原样) -->
        <view class="stat-item">
          <text class="stat-label">加总数</text>
          <text class="stat-num">{{ totalData.xiaoji }}</text>
        </view>
        
        <!-- 右侧：总次数 (修改点：增加了 total-btn 类名，结构微调) -->
        <view class="stat-item total-btn" @click="goYearTable">
          <view class="btn-content">
            <view class="label-box">
              <text class="stat-label">总次数</text>
             
            </view>
            <text class="stat-num">{{ totalData.changshu }}</text>
            <text class="btn-hint">点击查看明细</text> <!-- 新增一行小字提示 -->
          </view>
        </view>
      </view>

      <!-- 第三行：操作按钮（右对齐） -->
      <view class="header-row action-row">
        <view 
          class="record-btn" 
          :class="{ 'disabled-btn': isLimit }" 
          @click="handleRecord"
        >
          <text class="btn-icon" v-if="!isLimit">+</text>
          <text>{{ isLimit ? '无录入权限' : '录入数据' }}</text>
        </view>
      </view>
    </view>

    <!-- 加载状态 -->
    <view v-else class="loading-state">
      <text>数据加载中...</text>
    </view>

    <!-- 列表区域：每行3个 -->

        <view class="grid-section" v-if="listData.length > 0">
          <view class="grid-item-wrapper" v-for="(item, index) in listData" :key="index" @click="goDetail(item)">
            <view class="data-card">
              
              <!-- 上半部分：名字 + 场数 + 正数 -->
              <view class="card-top-row">
                <!-- 名字方块 -->
                <view class="name-square">
                  <text>{{ item.name }}</text>
                </view>
                
                <!-- 右边的数据 (场数、正数) -->
                <view class="top-stats">
                  <view class="mini-stat">
                    <text class="label">场:</text>
                    <text class="val">{{ item.changshu }}</text>
                  </view>
                  <view class="mini-stat">
                    <text class="label">正:</text>
                    <text class="val">{{ item.zhengshu }}</text>
                  </view>
                </view>
              </view>
              
              <!-- 下半部分：小计 (分割线隔开) -->
			  <view class="card-bottom-row">
				  <text class="xiaoji-label">小计</text>
				  <!-- 
					  判断逻辑：
					  Number(item.xiaoji) < 0  -> 红色
					  否则 -> 蓝色 
				  -->
				  <text 
				  class="xiaoji-val" 
				  :class="Number(item.xiaoji) < 0 ? 'text-red' : 'text-blue'"
				  >
				  {{ item.xiaoji }}
				  </text>
			  </view>
    
            </view>
          </view>
        </view>

    <!-- 空数据状态 -->
    <view v-if="totalData && listData.length === 0" class="empty-state">
      <text>该年份暂无明细数据</text>
    </view>
	
	<view class="footer-fixed">
      <!-- 监听子组件加载完成事件 -->
      <DeviceIdBox @loaded="onDeviceLoaded" />
    </view>
  </view>
</template>

<script setup>
import { ref,onMounted} from 'vue';
import { onShow } from '@dcloudio/uni-app'; 
import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update'
import DeviceIdBox from '@/components/DeviceIdBox.vue';
// --- 状态变量 ---
const currentYear = ref('');      // 当前选中的年份
const currentLimitYear = ref(''); // 当前实际年份
const yearList = ref([]);         // 年份列表
const yearIndex = ref(0);         // Picker 索引
const totalData = ref(null);      // 头部统计数据
const listData = ref([]);         // 列表数据
const deviceSn = ref('');   // 存储设备码
const isLimit = ref(false); // 存储权限状态 (true=禁止录入)

onMounted(() => {	
//检查更新
	checkUpdate();	   
})
// --- 生命周期 ---
onShow(() => {
  // 1. 初始化年份逻辑 (加个判断：只有第一次进入时才生成年份，防止返回时重置用户的选择)
  if (yearList.value.length === 0) {
    const date = new Date();
    const nowYear = date.getFullYear(); 
    
    let years = [];
    for (let i = nowYear; i >= 2000; i--) {
      years.push(i.toString());
    }
    yearList.value = years;
    yearIndex.value = 0;
    currentYear.value = years[0];
  }

  // 2. 每次页面显示（含返回）都重新调用接口刷新数据
  // 确保有年份才请求
  if (currentYear.value && deviceSn.value) {
    getData();
  }
});

// 监听组件加载完成事件
const onDeviceLoaded = (sn) => {
  deviceSn.value = sn;
  // 拿到设备码后，发起请求
  if (currentYear.value) {
    getData();
  }
};

const getData = () => {
  uni.showLoading({ title: '加载中...', mask: true });
  
  const url = 'http://13.94.38.44:8080/TypeList/GetTotalByYear';

  uni.request({
    url: url,
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentYear.value,
	  sn: deviceSn.value
    },
    success: (res) => {
      let responseData = res.data;
      // 兼容处理字符串类型的返回
      if (typeof responseData === 'string') {
        try { responseData = JSON.parse(responseData); } catch (e) {}
      }

      if (responseData && !responseData.isError) {
        processData(responseData.result || []);
		isLimit.value = responseData.limit === true;
      } else {
        processData([]);
      }
    },
    fail: (err) => {
      console.error(err);
      uni.showToast({ title: '网络请求失败', icon: 'none' });
      processData([]);
    },
    complete: () => {
      uni.hideLoading();
    }
  });
};

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

  // 兜底逻辑：如果没有TOTAL，手动生成0数据
  if (!newTotal) {
    newTotal = { name: 'TOTAL', xiaoji: 0, zhengshu: 0, changshu: 0 };
  }

  listData.value = newList;
  totalData.value = newTotal;
};

// 年份切换
const onYearChange = (e) => {
  const index = e.detail.value;
  const selectedYear = yearList.value[index];

  if (selectedYear !== currentYear.value) {
    yearIndex.value = index;
    currentYear.value = selectedYear;
    getData();
  }
};

// 页面跳转
const goDetail = (item) => {
  uni.navigateTo({
    url: `/pages/detail/detail?name=${item.name}&time=${currentYear.value}`
  });
};

const handleRecord = () => {
  // 权限拦截
  if (isLimit.value) {
    uni.showToast({ title: '暂无录入权限', icon: 'none' });
    return;
  }
  
  uni.navigateTo({ url: '/pages/input/input' });
};

const goYearTable = () => {
  uni.navigateTo({
    url: `/pages/yearTable/yearTable?time=${currentYear.value}`
  });
};
</script>

<style lang="scss" scoped>
/* 全局页面容器 */
.page-container {
  min-height: 100vh;
  background-color: $bg-color-card;
 
  padding-bottom: 180rpx; 
  display: flex;
  flex-direction: column;
  position: relative;
}

/* --- 1. 头部样式：高度压缩，黑白风 --- */
.header-section {
  background-color: #1a1a1a; /* 特殊深色背景保留 */
  border-bottom-left-radius: 30rpx;
  border-bottom-right-radius: 30rpx;
  padding: 20rpx 30rpx 30rpx;
  color: #ffffff;
  
  @include flex-col;
  gap: 10rpx;
  box-shadow: 0 5rpx 15rpx rgba(0, 0, 0, 0.1);
}

/* 第一行：年份 */
.year-row {
  display: flex;
  align-items: center;
  height: 60rpx;
}

.year-picker-box {
  display: inline-flex;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.15);
  padding: 8rpx 20rpx;
  border-radius: 8rpx;
}

.year-text {
  font-size: 30rpx;
  font-weight: bold;
  margin-right: 10rpx;
}

.uni-icon-arrow {
  font-size: 20rpx;
  opacity: 0.6;
}

/* 第二行：核心数据 */
.stats-row {
  @include flex-between; /* 替换 justify-content: space-between */
  align-items: stretch; /* 特殊对齐保留 */
  padding: 0 10rpx;
  margin-top: 10rpx;
}

.stat-item {
  @include flex-col;
  justify-content: center;
}

.total-btn {
  background: linear-gradient(135deg, #333333, #444444);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: $radius-card; /* 使用统一圆角 */
  padding: 10rpx 24rpx;
  box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.3);
  margin-left: 40rpx;
  position: relative;
  overflow: hidden;

  &:active {
    transform: scale(0.96);
    background: #222;
  }
}

.btn-content {
  @include flex-col;
  align-items: flex-end;
}

.btn-hint {
  font-size: 18rpx;
  color: rgba(255,255,255,0.5);
  margin-top: 4rpx;
}

.label-box {
  display: flex;
  align-items: center;
  gap: 8rpx;
}

.stat-label {
  font-size: 24rpx;
  color: rgba(255, 255, 255, 0.6);
}

.link-hint {
  font-size: 20rpx;
  color: rgba(255, 255, 255, 0.4);
}

.stat-num {
  font-size: 48rpx;
  font-weight: bold;
  line-height: 1.2;
}

/* 第三行：按钮 (右下角) */
.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 10rpx;
}

.record-btn {
  background-color: #ffffff;
  color: #000000;
  padding: 10rpx 24rpx;
  border-radius: 6rpx;
  font-size: 24rpx;
  font-weight: bold;
  display: flex;
  align-items: center;
    transition: all 0.3s;
  
    /* 新增禁用样式 */
    &.disabled-btn {
      background-color: #555; /* 深灰 */
      color: #aaa;
      pointer-events: none; /* 甚至可以加这个直接禁点，双重保险 */
    }
}

.btn-icon {
  margin-right: 6rpx;
  font-size: 30rpx;
  line-height: 1;
  margin-top: -4rpx;
}

/* --- 2. 列表区域 --- */
.loading-state, .empty-state {
  padding: 100rpx 0;
  text-align: center;
  color: $text-color-sub; /* 使用统一灰 */
  font-size: 28rpx;
}

.grid-section {
  background-color: #f2f3f5; /* 列表区特定灰底 */
  padding: 10rpx;
  border-radius: $radius-card;
  display: flex;
  flex-wrap: wrap;
}

/* 单个卡片容器 */
.grid-item-wrapper {
  width: 33.333%;
  padding: 8rpx;
  box-sizing: border-box;
  margin-bottom: 8rpx;
}

/* 卡片本体：改为纵向布局 */
.data-card {
  @include card-box; /* 引入卡片混合宏：背景、圆角、阴影 */
  border-radius: 12rpx; /* 如果需要比通用圆角小，可以覆盖，否则删除此行 */
  padding: 16rpx 10rpx;
  @include flex-col;
  gap: 12rpx;
}

/* --- 第一排：名字 + 场/正 --- */
.card-top-row {
  @include flex-center; /* 替换 display:flex; align-items:center; justify-content:center */
  gap: 16rpx;
  margin-bottom: 8rpx;
}

/* 名字方块 */
.name-square {
  width: 85rpx;
  height: 85rpx;
  background-color: #000000;
  border-radius: 12rpx;
  
  @include flex-center; /* 完美居中 */
  flex-shrink: 0;

  text {
    font-size: 34rpx;
    font-weight: bold;
    color: #ffffff;
  }
}

/* 右边的数据区 */
.top-stats {
  @include flex-col;
  justify-content: center;
  gap: 6rpx;
}

.mini-stat {
  display: flex;
  align-items: center;
  font-size: 22rpx;
  line-height: 1;

  .label {
    color: $text-color-sub; /* 统一灰色 */
    margin-right: 6rpx;
  }
  .val {
    color: $text-color-main; /* 统一深色 */
    font-weight: 600;
    font-size: 24rpx;
  }
}

/* --- 第二排：小计 --- */
.card-bottom-row {
  background-color: #f8f8f8;
  border-radius: 6rpx;
  padding: 6rpx 0;
  
  @include flex-center; /* 替换 flex; justify-content: center; align-items: center */
  gap: 10rpx;
}

.xiaoji-label {
  font-size: 20rpx;
  color: #666;
}

.xiaoji-val {
  font-size: 30rpx;
  color: #000;
  font-weight: 900;
  line-height: 1;
}

/* 负数显示红色 */
.text-red {
  color: $text-color-red !important; /* 使用变量 */
}

/* 正数显示蓝色 */
.text-blue {
  color: $text-color-blue !important; /* 使用变量 */
}
/* 修改后的 .footer-fixed */
.footer-fixed {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  
  /* 去掉背景色，让它透明 */
  background-color: transparent; 
  
  /* 调整内边距，只留必要的安全距离（比如左右留空，底部防遮挡） */
  padding: 0 30rpx 4rpx 30rpx; 
  
  box-sizing: border-box;
  z-index: 99;
  
  /* 去掉阴影，因为组件自己有阴影 */
  box-shadow: none; 
}
</style>