<template>
	<view class="container">

		<!-- === 固定区域开始 === -->

		<!-- 1. 顶部标题 -->
		<view class="page-header">
			<text class="title">类型统计</text>
			<text class="sub-title">{{ currentTime }}年 </text>
		</view>

		<!-- 2. 统计概览 (六个格子) -->
		<view class="stat-grid" v-if="totalStat">
			<view class="stat-item" v-for="(key, index) in statKeys" :key="index">
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

		<!-- 3. 列表区域 (撑满剩余空间) -->
		<view class="list-container" v-if="listData.length > 0">

			<!-- 表头 (固定在卡片顶部) -->
			<view class="list-header">
				<text class="th col-1">名称</text>
				<text class="th col-2">次数</text>
				<text class="th col-3">比例</text>
				<text class="th col-4">总数</text>
			</view>

			<!-- 数据行 (只有这里滚动) -->
			<scroll-view scroll-y class="list-scroll" show-scrollbar="true">
				<view class="list-row" v-for="(item, index) in listData" :key="index">
					<view class="td col-1 bold">{{ item[0] }}</view>
					<view class="td col-2">{{ item[1] }}</view>
					<view class="td col-3 highlight">{{ item[2] }}</view>
					<view class="td col-4" :class="getTotalClass(item[3])">{{ item[3] }}</view>
				</view>
			</scroll-view>

		</view>

	</view>
</template>

<script setup>
	import {
		ref
	} from 'vue';
	import {
		onLoad
	} from '@dcloudio/uni-app';

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
			header: {
				'content-type': 'application/json'
			},
			data: {
				time: currentTime.value,
				typeId: currentTypeId.value
			},
			success: (res) => {
				let data = res.data;
				if (typeof data === 'string') {
					try {
						data = JSON.parse(data);
					} catch (e) {}
				}

				if (data && !data.isError) {
					// 1. 处理 list
					listData.value = data.list || [];

					// 2. 处理 total (接口返回的是 total:[{...}])，取数组第一个对象
					if (data.total && data.total.length > 0) {
						totalStat.value = data.total[0];
					}
				} else {
					uni.showToast({
						title: '获取数据失败',
						icon: 'none'
					});
				}
			},
			fail: () => {
				uni.showToast({
					title: '网络错误',
					icon: 'none'
				});
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
	/* 1. 页面容器：全屏高度，Flex 纵向布局 */
	.container {
		height: 100vh;
		/* 关键：锁定屏幕高度 */
		background-color: #f5f7fa;
		padding: 30rpx 30rpx 0 30rpx;
		/* 底部padding去掉，交给内部处理 */
		display: flex;
		flex-direction: column;
		overflow: hidden;
		/* 防止页面整体滚动 */
		box-sizing: border-box;
	}

	/* 固定区域不伸缩 */
	.page-header,
	.stat-grid,
	.loading-box {
		flex-shrink: 0;
	}

	/* 顶部标题 */
	.page-header {
		margin-bottom: 20rpx;

		.title {
			font-size: 36rpx;
			font-weight: bold;
			color: #333;
			margin-right: 20rpx;
		}

		.sub-title {
			font-size: 24rpx;
			color: #999;
		}
	}

	/* --- 上方六宫格统计 --- */
	.stat-grid {
		display: flex;
		flex-wrap: wrap;
		justify-content: space-between;
		margin-bottom: 20rpx;
	}

	.stat-item {
		width: 48%;
		margin-bottom: 20rpx;
	}

	.stat-card {
		background-color: #fff;
		border-radius: 12rpx;
		padding: 24rpx;
		display: flex;
		justify-content: space-between;
		align-items: center;
		box-shadow: 0 2rpx 10rpx rgba(0, 0, 0, 0.03);

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

	/* --- 下方列表容器 (关键修改) --- */
	.list-container {
		flex: 1;
		/* 占据剩余所有空间 */
		min-height: 0;
		/* 防止Flex子元素溢出 */
		background-color: #fff;
		border-top-left-radius: 16rpx;
		border-top-right-radius: 16rpx;
		/* 底部圆角去掉或保留看喜好，这里保留 */
		border-bottom-left-radius: 0;
		border-bottom-right-radius: 0;

		box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.05);
		display: flex;
		flex-direction: column;
		overflow: hidden;
		/* 裁剪圆角 */
		margin-bottom: 0;
		/* 贴底 */
	}

	.list-header {
		flex-shrink: 0;
		/* 表头不缩放 */
		display: flex;
		padding: 24rpx 20rpx;
		background-color: #eef2f9;
		border-bottom: 1rpx solid #e1e5eb;

		.th {
			font-size: 26rpx;
			font-weight: bold;
			color: #333;
			text-align: center;
		}
	}

	/* --- 滚动区域 (关键修改) --- */
	/* --- 滚动区域 --- */
	.list-scroll {
		flex: 1;
		height: 100%;
		width: 100%;

		/* 
     === 修改点：自定义滚动条样式 === 
     注意：这在 H5 和 App(Webview渲染) 中生效。
  */

		/* 1. 滚动条整体 */
		& ::-webkit-scrollbar {
			width: 8rpx;
			/* 宽度设置得细一点，不占位置 */
			height: 8rpx;
			background-color: transparent;
			/* 背景透明 */
		}

		/* 2. 滚动条轨道 (Track) */
		& ::-webkit-scrollbar-track {
			background-color: transparent;
			/* 轨道完全透明，看起来像悬浮 */
			border-radius: 4rpx;
		}

		/* 3. 滚动条滑块 (Thumb) - 也就是那个拖动条 */
		& ::-webkit-scrollbar-thumb {
			/* 使用 rgba 设置半透明黑色，适应浅色背景，非常低调 */
			background-color: rgba(227, 227, 227, 0.3);
			border-radius: 10rpx;
			/* 圆角，看起来更柔和 */

			/* 鼠标/手指悬停或按住时稍微深一点点，提供反馈 (可选) */
			&:hover,
			&:active {
				background-color: rgba(249, 249, 249, 0.3);
			}
		}
	}

	.list-row {
		display: flex;
		padding: 24rpx 20rpx;
		align-items: center;
		border-bottom: 1rpx solid #f0f0f0;

		/* 给最后一行加点底部留白，防止贴太紧 */
		&:last-child {
			border-bottom: none;
			padding-bottom: 40rpx;
		}

		.td {
			font-size: 26rpx;
			color: #666;
			text-align: center;
		}
	}

	/* 列宽分配 (一共100%) */
	.col-1 {
		width: 20%;
		text-align: left;
		padding-left: 10rpx;
	}

	.col-2 {
		width: 20%;
	}

	.col-3 {
		width: 25%;
	}

	.col-4 {
		width: 35%;
		text-align: right;
		padding-right: 10rpx;
	}

	/* 字体样式 */
	.bold {
		font-weight: bold;
		color: #333;
	}

	.highlight {
		color: #007aff;
		background-color: rgba(0, 122, 255, 0.05);
		padding: 4rpx 8rpx;
		border-radius: 8rpx;
	}

	.red-text {
		color: #e74c3c;
	}

	.green-text {
		color: #27ae60;
	}

	.loading-box {
		text-align: center;
		padding: 30rpx;
		color: #999;
		flex-shrink: 0;
	}

	.text-blue {
		color: #007aff !important;
	}

	.text-red {
		color: #ff3b30 !important;
	}

	.text-gray {
		color: #ccc !important;
	}

	.text-black {
		color: #333 !important;
	}
</style>