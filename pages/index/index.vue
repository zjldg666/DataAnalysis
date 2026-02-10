<template>
	<view class="page-container">

		<!-- 头部区域：严格的三行布局 -->
		<view class="header-section" v-if="totalData">

			<!-- 第一行：年份选择 +设置 -->
			<view class="header-row year-row-relative">

				<!-- 年份选择器 (保持居中) -->
				<picker mode="selector" :range="yearList" :value="yearIndex" @change="onYearChange"
					class="year-picker-wrapper">
					<view class="year-picker-box">
						<text class="year-text">{{ currentYear }}年</text>
						<text class="uni-icon-arrow">▼</text>
					</view>
				</picker>

				<!-- [新增] 右侧设置按钮 -->
				<view class="settings-btn" @click="openSettings">
					<text class="settings-icon">⚙</text>
				</view>
			</view>

			<!-- 第二行：绝对居中对齐布局 -->
			<view class="header-row stats-action-row">

				<!-- 【左侧容器】占位 1 -->
				<view class="side-wrapper left-align">
					<view class="stat-item">
						<text class="stat-label">加总数</text>
						<text class="stat-num">{{ totalData.xiaoji }}</text>
					</view>
				</view>

				<!-- 【中间容器】自然宽度，被左右挤在正中间 -->
				<view class="center-wrapper">
					<view class="stat-item total-btn" @click="goYearTable">
						<view class="btn-content">
							<view class="label-box">
								<text class="stat-label">总次数</text>
							</view>
							<text class="stat-num">{{ totalData.changshu }}</text>
							<text class="btn-hint">点击查看明细</text>
						</view>
					</view>
				</view>

				<!-- 【右侧容器】占位 1 -->
				<view class="side-wrapper right-align">
					<view class="record-btn" :class="{ 'disabled-btn': isLimit }" @click="handleRecord">
						<text class="btn-icon" v-if="!isLimit">+</text>
						<text>{{ isLimit ? '无录入权限' : '录入数据' }}</text>
					</view>
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
						<text class="xiaoji-val" :class="Number(item.xiaoji) < 0 ? 'text-red' : 'text-blue'">
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

		<!-- [修改] 隐藏 DeviceIdBox，保留逻辑 -->
		<view style="display: none;">
			<DeviceIdBox @loaded="onDeviceLoaded" />
		</view>

		<!-- 设备码弹窗 -->
		<view class="modal-mask" v-if="showModal" @click="closeSettings">
			<view class="modal-content" @click.stop>
				<view class="modal-header">设备信息</view>
				<view class="modal-body">
					<text class="modal-label">设备识别码 (SN):</text>
					<text class="modal-sn">{{ deviceSn || '获取中...' }}</text>
				</view>
				<view class="modal-footer">
					<button class="modal-btn copy-btn" @click="copySn">复制设备码</button>
				</view>
			</view>
		</view>

	</view>
</template>

<script setup>
	import {
		ref,
		onMounted
	} from 'vue';
	import {
		onShow
	} from '@dcloudio/uni-app';
	import checkUpdate from '@/uni_modules/uni-upgrade-center-app/utils/check-update'
	import DeviceIdBox from '@/components/DeviceIdBox.vue';
	// --- 状态变量 ---
	const currentYear = ref(''); // 当前选中的年份
	const currentLimitYear = ref(''); // 当前实际年份
	const yearList = ref([]); // 年份列表
	const yearIndex = ref(0); // Picker 索引
	const totalData = ref(null); // 头部统计数据
	const listData = ref([]); // 列表数据
	const deviceSn = ref(''); // 存储设备码
	const isLimit = ref(false); // 存储权限状态 (true=禁止录入)

	const showModal = ref(false); //设置弹窗
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

	// 打开设置弹窗
	const openSettings = () => {
		showModal.value = true;
	};

	// 关闭弹窗
	const closeSettings = () => {
		showModal.value = false;
	};

	// 复制设备码
	const copySn = () => {
		if (!deviceSn.value) return;

		uni.setClipboardData({
			data: deviceSn.value,
			success: () => {
				uni.showToast({
					title: '复制成功',
					icon: 'success'
				});
				// 可选：复制后自动关闭弹窗
				// showModal.value = false; 
			}
		});
	};

	const getData = () => {
		uni.showLoading({
			title: '加载中...',
			mask: true
		});

		const url = 'http://13.94.38.44:8080/TypeList/GetTotalByYear';

		uni.request({
			url: url,
			method: 'POST',
			header: {
				'content-type': 'application/json'
			},
			data: {
				time: currentYear.value,
				sn: deviceSn.value
			},
			success: (res) => {
				let responseData = res.data;
				// 兼容处理字符串类型的返回
				if (typeof responseData === 'string') {
					try {
						responseData = JSON.parse(responseData);
					} catch (e) {}
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
				uni.showToast({
					title: '网络请求失败',
					icon: 'none'
				});
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
			newTotal = {
				name: 'TOTAL',
				xiaoji: 0,
				zhengshu: 0,
				changshu: 0
			};
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
			uni.showToast({
				title: '暂无录入权限',
				icon: 'none'
			});
			return;
		}

		uni.navigateTo({
			url: '/pages/input/input'
		});
	};

	const goYearTable = () => {
		uni.navigateTo({
			url: `/pages/yearTable/yearTable?time=${currentYear.value}&limit=${isLimit.value}`
		});
	};
</script>

<style lang="scss" scoped>
	/* 全局页面容器 */
	.page-container {
		min-height: 100vh;
		background-color: $bg-color-page;

		display: flex;
		flex-direction: column;
		position: relative;
	}


	/* --- 1. 头部样式：绝对居中对齐版 --- */
	.header-section {
		background-color: #1a1a1a;
		border-bottom-left-radius: 30rpx;
		border-bottom-right-radius: 30rpx;
		padding: 20rpx 30rpx 30rpx;
		color: #ffffff;

		@include flex-col;
		gap: 20rpx;
		box-shadow: 0 5rpx 15rpx rgba(0, 0, 0, 0.1);
	}

	/* 第一行：年份居中 */
	.year-row-center {
		display: flex;
		justify-content: center;
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

	/* 第二行：Flex 布局容器 */
	.stats-action-row {
		display: flex;
		align-items: center;
		/* 垂直居中 */
		padding: 0 5rpx;
	}

	/* 关键样式：左右两侧容器 */
	.side-wrapper {
		flex: 1;
		/* 关键：左右两侧平分剩余空间，强制宽度相等 */
		display: flex;
		min-width: 0;
		/* 防止内容过长撑破布局 */
	}

	/* 左侧内部靠左 */
	.left-align {
		justify-content: flex-start;
	}

	/* 右侧内部靠右 */
	.right-align {
		justify-content: flex-end;
	}

	/* 中间容器：不设 flex，保持自身宽度 */
	.center-wrapper {
		display: flex;
		justify-content: center;
	}

	/* 以下保持你原有的组件样式 */

	/* 统计项通用 */
	.stat-item {
		@include flex-col;
		justify-content: center;
	}

	/* 总次数按钮 (原样) */
	.total-btn {
		background: linear-gradient(135deg, #333333, #444444);
		border: 1px solid rgba(255, 255, 255, 0.3);
		border-radius: $radius-card;
		padding: 10rpx 24rpx;
		box-shadow: 0 4rpx 10rpx rgba(0, 0, 0, 0.3);
		position: relative;
		overflow: hidden;

		&:active {
			transform: scale(0.96);
			background: #222;
		}
	}

	.btn-content {
		@include flex-col;
		align-items: center;
	}

	.btn-hint {
		font-size: 18rpx;
		color: rgba(255, 255, 255, 0.5);
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

	.stat-num {
		font-size: 48rpx;
		font-weight: bold;
		line-height: 1.2;
	}

	/* 录入按钮 (原样) */
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
		white-space: nowrap;
		/* 防止文字换行 */

		&.disabled-btn {
			background-color: #555;
			color: #aaa;
			pointer-events: none;
		}
	}

	.btn-icon {
		margin-right: 6rpx;
		font-size: 30rpx;
		line-height: 1;
		margin-top: -4rpx;
	}

	/* --- 2. 列表区域 --- */
	.loading-state,
	.empty-state {
		padding: 100rpx 0;
		text-align: center;
		color: $text-color-sub;
		/* 使用统一灰 */
		font-size: 28rpx;
	}

	.grid-section {
		background-color: $bg-color-page;
		/* 列表区特定灰底 */
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
		@include card-box;
		/* 引入卡片混合宏：背景、圆角、阴影 */
		border-radius: 12rpx;
		/* 如果需要比通用圆角小，可以覆盖，否则删除此行 */
		padding: 16rpx 10rpx;
		@include flex-col;
		gap: 12rpx;
	}

	/* --- 第一排：名字 + 场/正 --- */
	.card-top-row {
		@include flex-center;
		/* 修改1：减小间距，给名字腾出空间 (原 16rpx -> 6rpx) */
		gap: 6rpx;
		margin-bottom: 8rpx;
		/* 增加：确保内容两端对齐或紧凑排列，防止溢出 */
		justify-content: space-between;
	}

	/* 名字方块 */
	.name-square {
		/* 修改2：宽度加宽，足以容纳3个34rpx的字 (原 85rpx -> 108rpx) */
		width: 108rpx;
		height: 85rpx;
		background-color: #000000;
		border-radius: 12rpx;

		@include flex-center;
		/* 保持居中 */
		flex-shrink: 0;
		/* 防止被挤压变形 */

		/* 增加：内边距设为0，确保文字最大化利用空间 */
		padding: 0;

		text {
			font-size: 34rpx;
			/* 保持字体大小不变 */
			font-weight: bold;
			color: #ffffff;

			/* 增加：防止3个字换行，强制一行显示 */
			white-space: nowrap;
			line-height: 1;
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
			color: $text-color-sub;
			/* 统一灰色 */
			margin-right: 6rpx;
		}

		.val {
			color: $text-color-main;
			/* 统一深色 */
			font-weight: 600;
			font-size: 24rpx;
		}
	}

	/* --- 第二排：小计 --- */
	.card-bottom-row {
		background-color: #f8f8f8;
		border-radius: 6rpx;
		padding: 6rpx 0;

		@include flex-center;
		/* 替换 flex; justify-content: center; align-items: center */
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
		color: $text-color-red !important;
		/* 使用变量 */
	}

	/* 正数显示蓝色 */
	.text-blue {
		color: $text-color-blue !important;
		/* 使用变量 */
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

	.year-row-relative {
		position: relative;
		/* 关键：为了让设置按钮绝对定位 */
		display: flex;
		justify-content: center;
		/* 让年份居中 */
		align-items: center;
		height: 60rpx;
		margin-bottom: 10rpx;
	}

	/* 年份选择器包装 */
	.year-picker-wrapper {
		z-index: 1;
	}

	.year-picker-box {
		display: inline-flex;
		align-items: center;
		background-color: rgba(255, 255, 255, 0.15);
		padding: 8rpx 20rpx;
		border-radius: 8rpx;

		.year-text {
			font-size: 30rpx;
			font-weight: bold;
			margin-right: 10rpx;
		}

		.uni-icon-arrow {
			font-size: 20rpx;
			opacity: 0.6;
		}
	}

	/* [新增] 设置按钮样式 */
	.settings-btn {
		position: absolute;
		right: 0;
		/* 靠右对齐 */
		top: 50%;
		transform: translateY(-50%);
		/* 垂直居中 */

		width: 60rpx;
		height: 60rpx;
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: 50%;
		background-color: rgba(255, 255, 255, 0.1);
		/* 轻微背景 */

		&:active {
			background-color: rgba(255, 255, 255, 0.2);
		}
	}

	.settings-icon {
		font-size: 34rpx;
		color: rgba(255, 255, 255, 0.8);
		margin-top: -4rpx;
		/* 微调图标位置 */
	}

	.modal-mask {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0, 0, 0, 0.6);
		z-index: 999;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.modal-content {
		width: 80%;
		max-width: 600rpx;
		background-color: #fff;
		border-radius: 20rpx;
		padding: 40rpx;
		box-sizing: border-box;
		animation: popIn 0.2s ease-out;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.modal-header {
		font-size: 34rpx;
		font-weight: bold;
		color: #333;
		margin-bottom: 30rpx;
	}

	.modal-body {
		width: 100%;
		background-color: #f5f5f5;
		padding: 20rpx;
		border-radius: 12rpx;
		margin-bottom: 40rpx;
		text-align: center;
		word-break: break-all;
		/* 防止长字符串撑破 */
	}

	.modal-label {
		display: block;
		font-size: 26rpx;
		color: #888;
		margin-bottom: 10rpx;
	}

	.modal-sn {
		font-size: 32rpx;
		font-weight: bold;
		color: #333;
		font-family: monospace;
		/* 等宽字体显示SN码更好看 */
	}

	.modal-footer {
		width: 100%;
	}

	.modal-btn {
		width: 100%;
		height: 80rpx;
		line-height: 80rpx;
		border-radius: 40rpx;
		font-size: 30rpx;
		font-weight: bold;
		border: none;

		&.copy-btn {
			background-color: #007aff;
			/* 蓝色主题 */
			color: #fff;
			box-shadow: 0 6rpx 16rpx rgba(0, 122, 255, 0.3);

			&:active {
				opacity: 0.9;
			}
		}
	}

	@keyframes popIn {
		from {
			transform: scale(0.9);
			opacity: 0;
		}

		to {
			transform: scale(1);
			opacity: 1;
		}
	}
</style>