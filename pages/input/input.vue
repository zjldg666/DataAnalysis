<!-- 录入数据 -->
<template>
	<view class="container">

		<!-- 1. 顶部日期选择 -->
		<view class="header-bar">
			<view class="date-label">当前日期：</view>
			<picker mode="date" :value="currentDate" @change="onDateChange">
				<view class="date-picker-box">
					<text>{{ currentDate ? currentDate : '请选择日期' }}</text>
					<text class="arrow">▼</text>
				</view>
			</picker>
		</view>

		<!-- 2. 中间网格输入区域 -->
		<scroll-view scroll-y class="grid-content">
			<view class="grid-wrapper" v-if="currentDate && dataList.length > 0">
				<view class="grid-item" v-for="(item, index) in dataList" :key="index">
					<view class="square-box">
						<text class="type-name">{{ item.Type }}</text>
						<input class="qty-input" type="number" v-model="item.Qty" placeholder="0"
							placeholder-class="input-placeholder" :style="{ color: getNumColor(item.Qty) }"
							@focus="onInputFocus(item)" @blur="onInputBlur(item)" />
					</view>
				</view>
			</view>

			<view v-if="!currentDate" class="empty-tip">
				<text>请先点击顶部选择日期</text>
			</view>
			<view v-else-if="isLoading" class="loading-box">
				<text>加载数据中...</text>
			</view>
		</scroll-view>

		<!-- 3. 底部悬浮保存按钮 -->
		<view class="footer-btn-area">
			<button class="save-btn" :class="{ 'disabled': !canSave }" :disabled="!canSave" @click="handleSave">
				{{ buttonText }}
			</button>
		</view>

		<!-- 4. [新增] 确认弹窗 -->
		<ConfirmPopup v-model:visible="showConfirm" title="录入确认" confirmText="确认保存" cancelText="返回修改"
			@confirm="onConfirmSave">
			<view class="modal-list-wrapper">
				<text class="modal-tip">请核对以下录入数据：</text>

				<!-- 滚动列表 -->
				<scroll-view scroll-y class="check-list">
					<view v-for="(item, index) in saveList" :key="index" class="check-item"
						:class="{ 'summary-row': item.isSummary }">
						<!-- 左侧：类型名称 -->
						<text class="check-name">{{ item.Type }}</text>

						<!-- 中间：虚线连接 (小计行不需要虚线，或者保持也行) -->
						<view class="check-line" v-if="!item.isSummary"></view>
						<!-- 小计行用空格占位 -->
						<view class="spacer" v-else></view>

						<!-- 右侧：数值 -->
						<!-- 小计根据正负变色，普通行保持原逻辑 -->
						<text class="check-val" :class="{ 
                'val-zero': !item.isSummary && item.Qty === '0',
                'text-red': item.isSummary && Number(item.Qty) < 0,
                'text-blue': item.isSummary && Number(item.Qty) > 0
              }">
							{{ item.Qty }}
						</text>
					</view>
				</scroll-view>

				<!-- 底部汇总 -->
				<view class="modal-summary">
					<!-- 减去1，因为最后一行是小计 -->
					共录入 <text class="highlight-num">{{ saveList.length - 1 }}</text> 项数据
				</view>
			</view>
		</ConfirmPopup>

	</view>
</template>

<script setup>
	import {
		ref,
		onMounted,
		computed
	} from 'vue';
	// 引入组件
	import ConfirmPopup from '@/components/ConfirmPopup.vue';

	// 状态变量
	const currentDate = ref('');
	const dataList = ref([]);
	const dateId = ref('0');
	const isLoading = ref(false);

	// [新增] 弹窗控制与待保存列表
	const showConfirm = ref(false);
	const saveList = ref([]);

	onMounted(() => {
		// 初始化逻辑
	});

	// --- 按钮状态逻辑 ---
	const buttonText = computed(() => {
		if (!currentDate.value) return '请先选择日期';
		if (!hasInput.value) return '请先录入数据';
		return '保存数据';
	});

	const canSave = computed(() => {
		return currentDate.value && hasInput.value;
	});

	const hasInput = computed(() => {
		if (!dataList.value || dataList.value.length === 0) return false;
		return dataList.value.some(item => {
			// 只要 Qty 不是 null/undefined 且不是空字符串，就算有输入 (0也算)
			return item.Qty !== null && item.Qty !== '' && item.Qty !== undefined;
		});
	});

	// --- 获取数据 ---
	const getDataByDay = () => {
		if (!currentDate.value) return;

		isLoading.value = true;
		dataList.value = [];

		uni.request({
			url: 'http://13.94.38.44:8080/TypeList/GetDetailByDay',
			method: 'POST',
			header: {
				'content-type': 'application/json'
			},
			data: {
				time: currentDate.value
			},
			success: (res) => {
				let data = res.data;
				if (typeof data === 'string') {
					try {
						data = JSON.parse(data);
					} catch (e) {}
				}

				if (data && !data.isError && data.result) {

					// --- 核心修改点 ---
					const processedList = data.result.map(item => {
						return {
							TypeID: item.TypeID,
							Type: item.Type,
							ID: item.ID, // ID 必须保留！用于判断是“修改”还是“新增”

							// 无论后端返回 "9999" 还是 null，这里都变成 null
							// 这样页面输入框就会显示 placeholder="0"
							Qty: null
						};
					});

					dataList.value = processedList;
					dateId.value = data.DateID || '0';

				} else {
					uni.showToast({
						title: '获取数据失败',
						icon: 'none'
					});
				}
			},
			fail: () => {
				uni.showToast({
					title: '网络请求失败',
					icon: 'none'
				});
			},
			complete: () => {
				isLoading.value = false;
			}
		});
	};

	const onDateChange = (e) => {
		currentDate.value = e.detail.value;
		getDataByDay();
	};

	// --- 输入框交互优化 ---
	const onInputFocus = (item) => {
		if (Number(item.Qty) === 0) item.Qty = '';
	};

	const onInputBlur = (item) => {
		if (item.Qty === '' || item.Qty === null) item.Qty = null; // 保持 null 显示 placeholder
	};

	// --- 1. 点击保存：数据清洗与弹窗 ---
	const handleSave = () => {
		if (!canSave.value) return;

		const tempSaveList = [];
		let totalSum = 0; // [新增] 用于累加总和

		// 遍历数据进行清洗
		dataList.value.forEach(item => {
			let rawQty = item.Qty;

			// 只有非空值才处理
			if (rawQty !== null && rawQty !== undefined && String(rawQty).trim() !== '') {

				const numVal = Number(rawQty);
				const cleanQty = String(numVal);

				// 累加总和
				totalSum += numVal;

				tempSaveList.push({
					...item,
					Qty: cleanQty
				});
			}
		});

		if (tempSaveList.length === 0) {
			uni.showToast({
				title: '没有需要保存的数据',
				icon: 'none'
			});
			return;
		}

		// [核心新增] 计算小计 (总和取负)
		const xiaojiVal = -totalSum;

		// 将小计作为一个特殊项加入列表末尾，用于弹窗展示
		// 注意：这个项不需要发给后端，只是为了展示
		tempSaveList.push({
			Type: '小计',
			Qty: String(xiaojiVal),
			isSummary: true // 标记一下这是汇总项
		});

		saveList.value = tempSaveList;


		// 请求数据
		const previewData = {
			time: currentDate.value,
			dateID: dateId.value,
			list: tempSaveList.map(item => {
				// 获取原始 ID
				const recordId = item.ID;

				// 如果 ID 是 null/undefined/空字符串，就变为 "0"
				const finalId = (recordId !== null && recordId !== undefined && recordId !== '') ? String(
					recordId) : "0";

				return {
					TypeID: item.TypeID,
					Qty: item.Qty,
					Type: item.Type,
					ID: finalId // 这里传入处理后的 ID
				};
			})
		};

		console.log('====== [input页面] 准备保存的数据 ======');
		console.log(JSON.stringify(previewData, null, 2));

		showConfirm.value = true;
	};

	// --- 2. 确认保存回调 ---
	const onConfirmSave = () => {
		showConfirm.value = false; // 关闭弹窗
		executeSave(); // 执行请求
	};


	const executeSave = () => {
		uni.showLoading({
			title: '保存中...',
			mask: true
		});

		//  过滤掉 isSummary 为 true 的项 (即小计行)
		const realItems = saveList.value.filter(item => !item.isSummary);

		//  这里遍历 realItems，把小计去除
		const listParam = realItems.map(item => { // <--- 之前这里写错了

			// --- ID 处理逻辑 ---
			const recordId = item.ID;
			const finalId = (recordId !== null && recordId !== undefined && recordId !== '') ? String(
				recordId) : "0";

			return {
				TypeID: item.TypeID,
				Qty: item.Qty,
				Type: item.Type,
				ID: finalId
			};
		});

		uni.request({
			url: 'http://13.94.38.44:8080/TypeList/UpdateTypeList',
			method: 'POST',
			header: {
				'content-type': 'application/json'
			},
			data: {
				time: currentDate.value,
				dateID: dateId.value, // 这里的 dateId 就是 GetDetailByDay 返回的 "1"
				list: listParam
			},
			success: (res) => {
				let data = res.data;
				if (typeof data === 'string') {
					try {
						data = JSON.parse(data);
					} catch (e) {}
				}

				if (data && data.isError === false) {
					uni.showToast({
						title: '保存成功',
						icon: 'success'
					});
					setTimeout(() => {
						uni.navigateBack();
					}, 1500);
				} else {
					uni.showToast({
						title: '保存失败',
						icon: 'none'
					});
				}
			},
			fail: (err) => {
				console.error(err);
				uni.showToast({
					title: '网络请求错误',
					icon: 'none'
				});
			},
			complete: () => {
				uni.hideLoading();
			}
		});
	};

	const getNumColor = (val) => {
		if (val === null || val === '') return '#333';
		const num = Number(val);
		if (num > 0) return '#007aff';
		if (num < 0) return '#ff3b30';
		return '#333';
	};
</script>

<style lang="scss" scoped>
	.container {
		@include page-container;
		padding: 0;
		padding-bottom: 40rpx;
		@include flex-col;
	}

	/* --- header-bar 样式保持不变 --- */
	.header-bar {
		background-color: $bg-color-header-dark;
		padding: 30rpx 40rpx 40rpx;
		@include flex-between;
		box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.1);
		z-index: 10;
		border-bottom-left-radius: 30rpx;
		border-bottom-right-radius: 30rpx;
		margin-bottom: 20rpx;

		.date-label {
			font-size: 30rpx;
			font-weight: bold;
			color: $text-color-white;
		}

		.date-picker-box {
			background-color: rgba(255, 255, 255, 0.2);
			padding: 10rpx 30rpx;
			border-radius: 30rpx;
			@include flex-center;
			color: $text-color-white;
			font-weight: bold;
			font-size: 30rpx;
			border: 1px solid rgba(255, 255, 255, 0.3);

			.arrow {
				margin-left: 10rpx;
				font-size: 24rpx;
				opacity: 0.8;
				color: $text-color-orange;
			}
		}
	}

	/* --- grid-content --- */

	.grid-content {
		flex: 1;
		padding: 0 10rpx;
		/* 稍微减小整体左右内边距，让中间空间更大 */
		box-sizing: border-box;
		overflow-y: auto;

		&::-webkit-scrollbar {
			display: none;
		}
	}

	.empty-tip {
		padding-top: 200rpx;
		text-align: center;
		color: $text-color-sub;
		font-size: 30rpx;
	}

	.loading-box {
		text-align: center;
		padding: 60rpx;
		color: $text-color-sub;
		font-size: 26rpx;
	}

	.grid-wrapper {
		display: flex;
		flex-wrap: wrap;
		padding-bottom: 180rpx;

		/* 1. 改为 0 或者正数，防止上移 */
		margin-top: 0;
		/* 一点顶部内边距，让第一行和日期栏有一点距离 */
		padding-top: 4rpx;
	}


	.grid-item {
		width: 25%;
		/* 一行四个 */
		padding: 8rpx;
		box-sizing: border-box;
	}

	.square-box {
		@include card-box;
		aspect-ratio: 1 / 1;
		@include flex-col;
		justify-content: space-between;
		padding: 16rpx 10rpx;
		box-sizing: border-box;
		transition: all 0.2s;
		border: 1rpx solid transparent;

		&:focus-within {
			transform: translateY(-4rpx);
			box-shadow: 0 8rpx 20rpx rgba(0, 0, 0, 0.08);
			border-color: #333;
		}

		.type-name {
			font-size: 28rpx;
			font-weight: 900;
			color: $text-color-main;
			text-align: center;
			background-color: #f5f5f5;
			width: 100%;
			padding: 6rpx 0;
			border-radius: 8rpx;
			white-space: nowrap;
			overflow: hidden;
			text-overflow: ellipsis;
		}

		.qty-input {
			width: 100%;
			height: 60rpx;
			background-color: $bg-color-card;
			text-align: center;
			font-size: 32rpx;
			font-weight: bold;
			color: $text-color-main;
			border-bottom: 2rpx solid #eee;

			&:focus {
				border-bottom-color: #000;
			}
		}
	}

	.input-placeholder {
		color: $text-color-placeholder;
		font-weight: normal;
		font-size: 30rpx;
	}

	/* --- 底部按钮 --- */
	.footer-btn-area {
		position: fixed;
		bottom: 50rpx;
		left: 40rpx;
		right: 40rpx;
		z-index: 20;
	}

	.save-btn {
		background: $bg-color-header-dark;
		color: $text-color-white;
		height: 90rpx;
		line-height: 90rpx;
		border-radius: $radius-card;
		font-size: 32rpx;
		font-weight: bold;
		box-shadow: 0 8rpx 24rpx rgba(0, 0, 0, 0.15);
		border: none;

		&:active {
			transform: scale(0.98);
			background: #000;
		}

		&.disabled {
			background: #e0e0e0;
			color: $text-color-sub;
			box-shadow: none;
		}
	}

	/* --- [新增] 弹窗列表样式 --- */
	.modal-list-wrapper {
		width: 100%;
		display: flex;
		flex-direction: column;
	}

	.modal-tip {
		font-size: 26rpx;
		color: #999;
		text-align: center;
		margin-bottom: 20rpx;
	}

	.check-list {
		max-height: 500rpx;
		background-color: #f9f9f9;
		border-radius: 12rpx;
		padding: 20rpx;
		box-sizing: border-box;
		margin-bottom: 20rpx;
	}

	.check-item {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-bottom: 16rpx;
		font-size: 28rpx;

		&:last-child {
			margin-bottom: 0;
		}
	}

	.check-name {
		font-weight: bold;
		color: #333;
		flex-shrink: 0;
	}

	/* 虚线连接 */
	.check-line {
		flex: 1;
		border-bottom: 2rpx dashed #e0e0e0;
		margin: 0 20rpx;
		position: relative;
		top: -6rpx;
	}

	.check-val {
		font-weight: 900;
		color: #007aff;
		font-size: 30rpx;
		flex-shrink: 0;
	}

	.val-zero {
		color: #999;
	}

	.modal-summary {
		text-align: right;
		font-size: 24rpx;
		color: #666;
	}

	.highlight-num {
		color: #007aff;
		font-weight: bold;
		margin: 0 6rpx;
		font-size: 32rpx;
	}

	/* 小计行特殊样式 */
	.summary-row {
		border-top: 2rpx solid #eee;
		/* 顶部分割线 */
		margin-top: 10rpx;
		padding-top: 16rpx;
		font-weight: bold;

		.check-name {
			font-size: 30rpx;
			color: #000;
		}

		.check-val {
			font-size: 34rpx;
			/* 数字大一点 */
		}
	}

	.spacer {
		flex: 1;
	}

	/* 颜色复用 */
	.text-red {
		color: #ff3b30 !important;
	}

	.text-blue {
		color: #007aff !important;
	}
</style>