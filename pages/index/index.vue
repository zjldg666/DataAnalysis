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
        <view class="record-btn" @click="handleRecord">
          <text class="btn-icon">+</text>
          <text>录入数据</text>
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

  </view>
</template>

<script setup>
import { ref,onMounted} from 'vue';
import { onShow } from '@dcloudio/uni-app'; 
import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update'
// --- 状态变量 ---
const currentYear = ref('');      // 当前选中的年份
const currentLimitYear = ref(''); // 当前实际年份
const yearList = ref([]);         // 年份列表
const yearIndex = ref(0);         // Picker 索引
const totalData = ref(null);      // 头部统计数据
const listData = ref([]);         // 列表数据
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
  if (currentYear.value) {
    getData();
  }
});

// --- 方法 ---

const getData = () => {
  uni.showLoading({ title: '加载中...', mask: true });
  
  const url = 'http://13.94.38.44:8080/TypeList/GetTotalByYear';

  uni.request({
    url: url,
    method: 'POST',
    header: { 'content-type': 'application/json' },
    data: {
      time: currentYear.value
    },
    success: (res) => {
      let responseData = res.data;
      // 兼容处理字符串类型的返回
      if (typeof responseData === 'string') {
        try { responseData = JSON.parse(responseData); } catch (e) {}
      }

      if (responseData && !responseData.isError) {
        processData(responseData.result || []);
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
  uni.navigateTo({
    url: '/pages/input/input'
  });
};

const goYearTable = () => {
  uni.navigateTo({
    url: `/pages/yearTable/yearTable?time=${currentYear.value}`
  });
};
</script>

<style lang="scss">
/* 全局页面底色 */
.page-container {
  min-height: 100vh;
  background-color: #ffffff;
  padding-bottom: 40rpx;
  display: flex;
  flex-direction: column;
}

/* --- 1. 头部样式：高度压缩，黑白风 --- */
.header-section {
  background-color: #1a1a1a; /* 纯黑背景 */
  border-bottom-left-radius: 30rpx;
  border-bottom-right-radius: 30rpx;
  
  /* 修改点：大幅减小内边距，压缩高度 */
  padding: 20rpx 30rpx 30rpx; 
  
  color: #ffffff;
  display: flex;
  flex-direction: column;
  
  /* 修改点：减小行与行之间的间距 */
  gap: 10rpx; 
  
  box-shadow: 0 5rpx 15rpx rgba(0, 0, 0, 0.1);
}

/* 第一行：年份 */
.year-row {
  display: flex;
  align-items: center;
  height: 60rpx; /* 固定高度，防止撑大 */
}

.year-picker-box {
  display: inline-flex;
  align-items: center;
  background-color: rgba(255, 255, 255, 0.15);
  padding: 8rpx 20rpx; /* 减小内边距 */
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
  display: flex;
  justify-content: space-between;
  align-items: stretch; /* 让左右高度一致 */
  padding: 0 10rpx;
  margin-top: 10rpx;
}

.stat-item {
  display: flex;
  flex-direction: column;
  justify-content: center;
}


.total-btn {
  background: linear-gradient(135deg, #333333, #444444); /* 深色渐变背景 */
  border: 1px solid rgba(255, 255, 255, 0.3); /* 明显的边框 */
  border-radius: 16rpx;
  padding: 10rpx 24rpx; /* 增加内边距 */
  box-shadow: 0 4rpx 10rpx rgba(0,0,0,0.3); /* 按钮阴影 */
  margin-left: 40rpx; /* 拉开和左边的距离 */
  position: relative;
  overflow: hidden;
  
  /* 点击时的按压效果 */
  &:active {
    transform: scale(0.96);
    background: #222;
  }
}

.btn-content {
  display: flex;
  flex-direction: column;
  align-items: flex-end; /* 内容靠右对齐 */
}


/* 按钮内的文字提示 */
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
  font-size: 48rpx; /* 稍微减小一点，避免过于夸张 */
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
  padding: 10rpx 24rpx; /* 按钮变小一点 */
  border-radius: 6rpx;
  font-size: 24rpx;
  font-weight: bold;
  display: flex;
  align-items: center;
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
  color: #999;
  font-size: 28rpx;
}

.grid-section {
  background-color: #f2f3f5;
  padding: 10rpx;
  border-radius: 16rpx;
  display: flex;
  flex-wrap: wrap;
}

/* 单个卡片容器 */
.grid-item-wrapper {
  width: 33.333%; 
  padding: 8rpx; /* 间隙稍微调小一点，让卡片宽一点 */
  box-sizing: border-box; 
  /* 这里加上之前说的底部间距 */
  margin-bottom: 16rpx; 
}

/* 卡片本体：改为纵向布局 */
.data-card {
  background-color: #ffffff;
  border-radius: 12rpx;
  padding: 16rpx 10rpx;
  display: flex;
  flex-direction: column; /* 垂直排列 */
  gap: 12rpx; /* 上下两排的间距 */
  box-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.05);
}

/* --- 第一排：名字 + 场/正 --- */
/* --- 第一排：名字 + 场/正 --- */
.card-top-row {
  display: flex;
  align-items: center; /* 垂直居中 */
  
  /* 修改点1：改为居中对齐，不再是 space-between */
  justify-content: center; 
  
  /* 修改点2：给名字和数据之间加固定的间距 */
  gap: 16rpx; 
  
  /* 微调：给下方的小计留点呼吸空间 */
  margin-bottom: 8rpx;
}

/* 名字方块 */
.name-square {
  /* 修改点3：大幅加大尺寸 */
  width: 85rpx;  
  height: 85rpx;
  background-color: #000000;
  border-radius: 12rpx; /* 圆角也稍微大一点 */
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  
  text {
    /* 修改点4：名字字体变大 */
    font-size: 34rpx; 
    font-weight: bold;
    color: #ffffff;
  }
}

/* 右边的数据区 */
.top-stats {
  display: flex;
  flex-direction: column;
  /* 修改点5：去掉右对齐，改为自然左对齐或居中 */
  justify-content: center; 
  gap: 6rpx; /* 场和正之间的间距稍微拉大 */
}

.mini-stat {
  display: flex;
  align-items: center;
  font-size: 22rpx; /* 字体稍微加大一丢丢 */
  line-height: 1;
  
  .label {
    color: #999;
    margin-right: 6rpx;
  }
  .val {
    color: #333;
    font-weight: 600;
    font-size: 24rpx; /* 数字也稍微大一点 */
  }
}

/* --- 第二排：小计 --- */
.card-bottom-row {
  background-color: #f8f8f8; /* 给小计加个浅底色，更像报表 */
  border-radius: 6rpx;
  padding: 6rpx 0;
  display: flex;
  justify-content: center; /* 居中显示 */
  align-items: center;
  gap: 10rpx;
}

.xiaoji-label {
  font-size: 20rpx;
  color: #666;
}

.xiaoji-val {
  font-size: 30rpx; /* 小计数字加大 */
  color: #000;
  font-weight: 900; /* 加粗 */
  line-height: 1;
}

/* 负数显示红色 */
.text-red {
  color: #ff3b30 !important; /* 鲜艳的红 */
}

/* 正数显示蓝色 */
.text-blue {
  color: #007aff !important; /* 鲜艳的蓝 */
}
</style>