<!-- 点击总次数 -->
<template>
	<!-- 1. 容器改为 Flex 布局结构 -->
	<view class="container" :class="{ 'mode-delete': isDeleteMode }">
		<!-- 顶部：固定 -->
		<view class="page-title">
			<text>{{ currentTime }}年度数据明细表</text>
			<text class="sub-text">← 左右滑动查看更多类型 →</text>
		</view>
		<!-- 中间：表格区域 (占满剩余空间) -->
		<view class="table-wrapper" v-if="tableData.length > 0">
			<view class="table-scroller">

				<!-- 表头 -->
				<view class="header-row">

					<!--  删除模式下的全选占位符  -->
					<view class="th col-check" v-if="isDeleteMode">
						<text>勾选</text>
					</view>

					<view class="th" v-for="(col, index) in columns" :key="index"
						:class="{ 'col-date': col === 'Date', 'col-total': col === 'TOTAL' }">
						{{ getColName(col) }}
					</view>
				</view>

				<!-- 数据行 -->
				<view class="data-row" v-for="(row, rIndex) in tableData" :key="rIndex"
					:class="{ 'stripe': rIndex % 2 === 1 }" @click="toggleSelectRow(row)">

					<!-- 删除复选框列 -->
					<view class="td col-check" v-if="isDeleteMode">
						<!-- 修改判断条件：使用 getRowKey(row) -->
						<view class="checkbox" :class="{ checked: selectedRows.includes(getRowKey(row)) }">
							<text v-if="selectedRows.includes(getRowKey(row))">✔</text>
						</view>
					</view>

					<view class="td" v-for="(col, cIndex) in columns" :key="cIndex" :class="{ 
              'col-date': col === 'Date', 
              'col-total': col === 'TOTAL',
              'text-blue': isNumber(row[col]) && row[col] > 0,
              'text-red': isNumber(row[col]) && row[col] < 0
            }">
						<!-- 1. 日期列 -->
						<block v-if="col === 'Date'">
							<!-- 删除模式下禁用修改日期 -->
							<picker mode="date" :value="row[col]" :disabled="isDeleteMode"
								@change="(e) => onRowDateChange(e, row)">
								<view class="date-picker-cell">
									<text>{{ row[col] }}</text>
									<text class="edit-icon" v-if="!isDeleteMode">✎</text>
								</view>
							</picker>
						</block>

						<!-- 2. 小计 -->
						<block v-else-if="col === 'TOTAL'">
							{{ formatValue(row[col]) }}
						</block>

						<!-- 3. 输入框 -->
						<block v-else>
							<!-- 删除模式下，输入框应该只读，防止误触 -->
							<input class="table-input" :class="{ 
                   'text-blue': Number(row[col]) > 0, 
                   'text-red': Number(row[col]) < 0 
                 }" type="number" v-model="row[col]" placeholder="0" placeholder-class="input-ph"
								:disabled="isDeleteMode" />
						</block>

					</view>
				</view>

			</view>
		</view>

		<!-- 加载/空状态 (放在 table-wrapper 同级或者里面都可以，这里建议用 v-else 控制 table-wrapper) -->
		<view v-if="isLoading" class="loading-box"><text>加载数据中...</text></view>
		<view v-if="!isLoading && tableData.length === 0" class="empty-box"><text>暂无数据</text></view>
		<view class="footer-area">
			<!-- 双按钮容器 -->
			<view class="footer-btns">

				<!-- 左侧：删除按钮 -->
				<button class="action-btn btn-delete" :class="{ 'disabled': isLimit, 'active': isDeleteMode }"
					:disabled="isLimit" @click="isDeleteMode ? handleDelete() : toggleDeleteMode()">
					{{ isLimit ? '无删除权限' : (isDeleteMode ? '确认删除 (' + selectedRows.length + ')' : '删除数据') }}
				</button>

				<!-- 右侧：修改按钮 -->
				<!-- 如果处于删除模式，显示“取消” -->
				<button class="action-btn btn-modify"
					:class="{ 'disabled': isLimit && !isDeleteMode, 'cancel-mode': isDeleteMode }"
					:disabled="isLimit && !isDeleteMode" @click="isDeleteMode ? toggleDeleteMode() : handleModify()">
					{{ isDeleteMode ? '取消' : (isLimit ? '无修改权限' : '修改数据') }}
				</button>

			</view>
		</view>
		<!-- 确认弹窗 -->
		<ConfirmPopup v-model:visible="showConfirm" title="修改详情确认" confirmText="确定" cancelText="取消"
			@confirm="onConfirmSave">
			<!-- 插槽内容 -->
			<view class="modal-content-box">
				<text class="modal-tip">以下数据将发生变更：</text>

				<!-- 滚动区域：防止内容太长 -->
				<scroll-view scroll-y class="change-list-scroll">

					<!-- 循环每一天 -->
					<view v-for="(dayItem, dIndex) in changeDetails" :key="dIndex" class="change-group">
						<!-- 日期标题 -->
						<view class="change-date-row">
							<text class="icon-date">📅</text>
							<text class="date-text">{{ dayItem.date }}</text>
						</view>

						<!-- 循环这一天的修改项 -->
						<view v-for="(item, iIndex) in dayItem.items" :key="iIndex" class="change-item"
							:class="{ 'summary-row': item.isSummary }">
							<text class="type-name">{{ item.type }}</text>

							<!-- 普通修改项：显示 旧 -> 新 -->
							<view class="val-box" v-if="!item.isSummary">
								<text class="old-val">{{ item.oldVal }}</text>
								<text class="arrow">→</text>
								<text class="new-val">{{ item.newVal }}</text>
							</view>

							<!-- 小计项：只显示数值 -->
							<view class="val-box" v-else>
								<text class="summary-val" :class="{ 
	                'text-red': Number(item.newVal) < 0,
	                'text-blue': Number(item.newVal) > 0
	              }">
									{{ item.newVal }}
								</text>
							</view>

						</view>
					</view>

				</scroll-view>

				<view class="modal-summary">
					共涉及 <text class="highlight-num">{{ requestsToSend.length }}</text> 个日期记录
				</view>
			</view>
		</ConfirmPopup>

		<!-- 删除确认弹窗 -->
		<ConfirmPopup v-model:visible="showDeleteConfirm" title="" confirmText="确认删除" cancelText="我再想想"
			@confirm="onConfirmDelete">
			<view class="delete-modal-content">


				<text class="modal-text">确定要删除选中的 <text class="highlight-num">{{ selectedRows.length }}</text>
					条记录吗？</text>
				<text class="sub-text">请谨慎操作。</text>

				<!-- 滚动列表：显示日期和小计 -->
				<scroll-view scroll-y class="delete-list-scroll">
					<view class="delete-list-item" v-for="(item, index) in deleteListDetails" :key="item.key">
						<!-- 左侧：日期 -->
						<view class="del-left">
							<text class="del-icon">📅</text>
							<text class="del-date">{{ item.date }}</text>
						</view>

						<!-- 右侧：小计 -->
						<view class="del-right">
							<text class="del-label">小计:</text>
							<text class="del-val" :class="{ 
	                  'text-red': Number(item.total) < 0,
	                  'text-blue': Number(item.total) > 0
	                }">
								{{ item.total }}
							</text>
						</view>
					</view>
				</scroll-view>

				<!-- 底部汇总 (可选，显示所有选中项的总和) -->
				<!-- <view class="delete-summary">
	           选中项总计: {{ deleteListDetails.reduce((acc, cur) => acc + Number(cur.total), 0) }}
	        </view> -->

			</view>
		</ConfirmPopup>
	</view>
</template>

<script setup>
	import {
		ref,
		computed
	} from 'vue';
	import {
		onLoad
	} from '@dcloudio/uni-app';
	import ConfirmPopup from '@/components/ConfirmPopup.vue';

	const currentTime = ref('');
	const tableData = ref([]);
	const columns = ref([]);
	const isLoading = ref(false);
	const scrollLeft = ref(0);
	const isLimit = ref(false); // 权限变量

	// 存储类型定义(dt)，用于保存时查找 TypeID
	const typeList = ref([]);

	// 控制弹窗显示的变量
	const showConfirm = ref(false);

	// 用于存储原始数据的快照，用于对比哪些被修改了
	const originalTableData = ref([]);

	//  用于存储待发送的请求数据
	const requestsToSend = ref([]);

	//  用于存储具体的变更详情，用于弹窗展示
	const changeDetails = ref([]);
	// 删除功能相关状态
	const isDeleteMode = ref(false); // 是否处于删除选择模式
	const selectedRows = ref([]); // 存储被选中的行的 DateID (或其他唯一标识)
	// 控制删除弹窗显示
	const showDeleteConfirm = ref(false);
	onLoad((options) => {
		if (options.time) {
			currentTime.value = options.time;
			if (options.limit !== undefined) {
				isLimit.value = options.limit === 'true';
			}
			getData();
		}
	});

	// --- 日期修改事件 ---
	const onRowDateChange = (e, row) => {
		if (isLimit.value) return; // 无权限拦截
		const newDate = e.detail.value;
		// 直接更新当前行的数据
		row.Date = newDate;
	};

	const getData = () => {
		isLoading.value = true;
		tableData.value = [];

		uni.request({
			url: 'http://13.94.38.44:8080/TypeList/GetDetailByYear',
			method: 'POST',
			header: {
				'content-type': 'application/json'
			},
			data: {
				time: currentTime.value
			},
			success: (res) => {
				let data = res.data;
				if (typeof data === 'string') {
					try {
						data = JSON.parse(data);
					} catch (e) {}
				}
				if (data && !data.isError && data.dtMain) {
					//  保存 Type 定义，后续保存要用
					if (data.dt) {
						typeList.value = data.dt;
					}
					processData(data.dtMain);
				} else {
					uni.showToast({
						title: '暂无数据',
						icon: 'none'
					});
				}
			},
			fail: () => {
				uni.showToast({
					title: '请求失败',
					icon: 'none'
				});
			},
			complete: () => {
				isLoading.value = false;
			}
		});
	};

	const processData = (list) => {
		if (list.length === 0) return;

		// 1. 处理列名 (保持不变)
		const allKeys = Object.keys(list[0]);
		const dynamicKeys = allKeys.filter(key =>
			key !== 'DateID' && key !== 'Date' && key !== 'TOTAL' && !key.endsWith('ID')
		);
		columns.value = ['Date', 'TOTAL', ...dynamicKeys];

		// 2. 赋值当前数据
		tableData.value = list;

		// 3.深拷贝一份原始数据，作为对比基准
		// 必须用 JSON.parse(JSON.stringify()) 断开引用，否则 tableData 变了它也会变
		originalTableData.value = JSON.parse(JSON.stringify(list));

		scrollLeft.value = 0;
	};
	// [新增] 计算属性：获取选中行的详细信息 (日期 + 小计)
	const deleteListDetails = computed(() => {
		return selectedRows.value.map(key => {
			// 1. 在 tableData 中找到对应的行数据
			const row = tableData.value.find(r => getRowKey(r) === key);

			// 2. 如果找到了，返回需要的格式
			if (row) {
				return {
					date: row.Date,
					total: row.TOTAL, // 获取小计
					key: key
				};
			}
			return null;
		}).filter(item => item !== null); // 过滤掉异常空值
	});
	// --- 进入/退出删除模式 ---
	const toggleDeleteMode = () => {
		if (isLimit.value) return;

		isDeleteMode.value = !isDeleteMode.value;
		selectedRows.value = []; // 每次切换清空选择

		if (isDeleteMode.value) {
			uni.showToast({
				title: '请选择要删除的日期',
				icon: 'none'
			});
		}
	};

	// 生成每一行的唯一标识符 (日期_ID)
	const getRowKey = (row) => {
		return `${row.Date}_${row.DateID}`;
	};
	// ---选中/取消选中某一行 ---
	const toggleSelectRow = (row) => {
		if (!isDeleteMode.value) return;

		// 使用组合键，而不是单纯的 DateID
		const key = getRowKey(row);

		const index = selectedRows.value.indexOf(key);
		if (index > -1) {
			selectedRows.value.splice(index, 1); // 取消选中
		} else {
			selectedRows.value.push(key); // 选中
		}
	};

	// ---点击确认删除按钮 ---
	const handleDelete = () => {
		if (selectedRows.value.length === 0) {
			uni.showToast({
				title: '请至少选择一条数据',
				icon: 'none'
			});
			return;
		}

		showDeleteConfirm.value = true;
	};

	// --- 弹窗确认回调 ---
	const onConfirmDelete = () => {
		// 关闭弹窗
		showDeleteConfirm.value = false;

		//  执行删除请求
		executeDelete();
	};

	// --- 执行删除请求 ---
	const executeDelete = () => {
		uni.showLoading({
			title: '删除中...',
			mask: true
		});

		const deletePromises = selectedRows.value.map(key => {
			// 通过组合键找到原始行数据
			const row = tableData.value.find(r => getRowKey(r) === key);

			if (!row) return Promise.resolve();

			return new Promise((resolve, reject) => {
				uni.request({
					url: 'http://13.94.38.44:8080/TypeList/DeleteList',
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: {
						time: row.Date,
						dateID: String(row.DateID)
					},
					success: (res) => {
						resolve(res.data);
					},
					fail: reject
				});
			});
		});

		Promise.all(deletePromises)
			.then(() => {
				uni.hideLoading();
				uni.showToast({
					title: '删除成功',
					icon: 'success'
				});
				isDeleteMode.value = false;
				selectedRows.value = [];
				setTimeout(() => {
					getData();
				}, 1000);
			})
			.catch((err) => {
				uni.hideLoading();
				console.error(err);
				uni.showToast({
					title: '删除部分失败',
					icon: 'none'
				});
				getData();
			});
	};


	// --- 点击修改按钮：准备数据并弹窗 ---
	const handleModify = () => {
		if (isLimit.value) return;

		const tempRequests = [];
		const tempDetails = [];

		// 遍历每一行 (每一天)
		tableData.value.forEach((row, rowIndex) => {
			const originalRow = originalTableData.value[rowIndex];
			if (!originalRow) return;

			const listPayload = [];
			let rowHasChange = false;
			const rowChanges = [];

			// [新增] 用于计算这一行的新值总和 (计算小计用)
			let dailySum = 0;

			// 1. 检查日期是否被修改
			if (row.Date !== originalRow.Date) {
				rowHasChange = true;
				rowChanges.push({
					type: '日期变更',
					oldVal: originalRow.Date,
					newVal: row.Date
				});
			}

			// 2. 遍历每一列 (数据列)
			columns.value.forEach(colName => {
				if (colName === 'Date' || colName === 'TOTAL') return;

				const currentVal = normalize(row[colName]);
				const originalVal = normalize(originalRow[colName]);

				// 检测变动 (用于UI展示)
				if (currentVal !== originalVal) {
					rowHasChange = true;
					rowChanges.push({
						type: colName,
						oldVal: originalVal === '' ? '(空)' : originalVal,
						newVal: currentVal === '' ? '(空)' : currentVal
					});
				}

				// [新增] 累加当前这一行所有 Type 的新值 (不管有没有变动，只要是数字就加)
				// 注意：空字符串按 0 处理
				const numVal = currentVal === '' ? 0 : Number(currentVal);
				if (!isNaN(numVal)) {
					dailySum += numVal;
				}

				// 组装数据 (请求格式保持不变：传整行)
				const typeDef = typeList.value.find(t => t.Type === colName);
				if (typeDef) {
					const dynamicIdKey = colName + 'ID';
					const recordId = row[dynamicIdKey];
					const finalId = (recordId !== null && recordId !== undefined && recordId !== '') ?
						String(recordId) : "0";

					listPayload.push({
						TypeID: String(typeDef.ID),
						Qty: currentVal,
						Type: colName,
						ID: finalId
					});
				}
			});

			if (rowHasChange) {
				// 组装请求参数
				const requestData = {
					time: row.Date,
					dateID: String(row.DateID),
					list: listPayload
				};

				tempRequests.push({
					url: 'http://13.94.38.44:8080/TypeList/UpdateTypeList',
					data: requestData
				});

				// [核心新增] 计算这一天的小计 (总和取负)
				const xiaojiVal = -dailySum;

				// 把小计也作为一个 change item 加进去，专门为了弹窗展示
				// 为了区分，我们可以给它加个特殊标记 isSummary
				rowChanges.push({
					type: '小计',
					oldVal: '', // 小计没有所谓的旧值对比，或者你可以算旧的总和
					newVal: String(xiaojiVal),
					isSummary: true // 标记
				});

				tempDetails.push({
					date: row.Date,
					items: rowChanges
				});
			}
		});

		if (tempRequests.length === 0) {
			uni.showToast({
				title: '没有检测到数据修改',
				icon: 'none'
			});
			return;
		}

		console.log(`====== 准备提交 ${tempRequests.length} 条记录 ======`);
		tempRequests.forEach(req => {
			console.log(`[${req.data.time}] 参数预览:`);
			console.log(JSON.stringify(req.data, null, 2));
		});

		requestsToSend.value = tempRequests;
		changeDetails.value = tempDetails;
		showConfirm.value = true;
	};

	const onConfirmSave = () => {
		// 1. 关闭弹窗
		showConfirm.value = false;

		// 2. 执行真正的保存逻辑 (复用之前的 executeSave)
		executeSave();
	};


	// --- 真正的保存逻辑 ---
	const normalize = (val) => {
		if (val === null || val === undefined) return "";
		return String(val).trim();
	};

	// --- 修改后的保存逻辑：只提交变更项 ---
	const executeSave = () => {
		if (requestsToSend.value.length === 0) return;

		uni.showLoading({
			title: '正在提交...',
			mask: true
		});

		const requestPromises = requestsToSend.value.map(req => {
			return new Promise((resolve, reject) => {
				uni.request({
					url: req.url,
					method: 'POST',
					header: {
						'content-type': 'application/json'
					},
					data: req.data,
					success: (res) => resolve(res.data),
					fail: (err) => reject(err)
				});
			});
		});

		Promise.all(requestPromises)
			.then(() => {
				uni.hideLoading();
				uni.showToast({
					title: '保存成功',
					icon: 'success'
				});
				// 清空待发送列表
				requestsToSend.value = [];
				// 延迟刷新
				setTimeout(() => {
					getData();
				}, 1000);
			})
			.catch((err) => {
				uni.hideLoading();
				console.error(err);
				uni.showToast({
					title: '保存失败',
					icon: 'none'
				});
			});
	};

	const getColName = (key) => {
		if (key === 'Date') return '日期';
		if (key === 'TOTAL') return '小计';
		return key;
	};

	// 格式化显示：如果是输入框，null显示为空；如果是文本，null显示-
	const formatValue = (val) => {
		if (val === null || val === undefined) return '-';
		return val;
	};

	const isNumber = (val) => {
		return typeof val === 'number';
	};
</script>

<style lang="scss" scoped>
	$date-width: 180rpx;
	$total-width: 140rpx;
	$normal-width: 140rpx;
	$check-width: 80rpx;
	/* 新增：复选框列宽度 */

	/* 1. 页面容器 */
	.container {
		height: 100vh;
		background-color: $bg-color-page;
		display: flex;
		flex-direction: column;
		overflow: hidden;
	}

	/* 2. 顶部标题 */
	.page-title {
		padding: 20rpx 30rpx;
		text-align: center;
		flex-shrink: 0;
		@include flex-col;
		background-color: $bg-color-page;
		z-index: 10;

		text:first-child {
			font-size: 34rpx;
			font-weight: bold;
			color: $text-color-main;
		}

		.sub-text {
			font-size: 22rpx;
			color: $text-color-sub;
			margin-top: 10rpx;
		}
	}

	/* 3. 表格 Wrapper */
	.table-wrapper {
		flex: 1;
		min-height: 0;
		background-color: #fff;
		border-top: 1rpx solid #eee;
		border-bottom: 1rpx solid #eee;
		position: relative;
	}

	/* 4. Scroll View */
	.scroll-view {
		width: 100%;
		height: 100%;
		white-space: nowrap;
	}

	.table-scroller {
		width: 100%;
		height: 100%;
		overflow: auto;
		/* 关键：开启原生滚动，支持 x 和 y 轴 */
		position: relative;
		/* 为 sticky 提供定位上下文 */
	}

	/* 表格行 */
	.header-row,
	.data-row {
		display: flex;
		min-width: 100%;
		width: max-content;
	}

	/* --- 表头吸顶 --- */
	.header-row {
		position: sticky;
		top: 0;
		z-index: 100;
		background-color: #eef2f9;
		border-bottom: 1rpx solid #e1e5eb;

		/* 关键：确保宽度撑开，否则横向滚动时背景会断掉 */
		width: max-content;
		min-width: 100%;
		display: flex;
	}

	.data-row {
		width: max-content;
		min-width: 100%;
		display: flex;
		border-bottom: 1rpx solid #f0f0f0;

		&.stripe {
			background-color: #fafafa;
		}
	}

	.th,
	.td {
		display: inline-block;
		width: $normal-width;
		padding: 24rpx 10rpx;
		text-align: center;
		vertical-align: middle;
	}

	/* --- [新增] 复选框列样式 --- */
	.col-check {
		width: $check-width;
		position: sticky;
		left: 0;
		z-index: 120;
		/* 最高层级 */
		background-color: #fff;
		border-right: 1rpx solid #eee;

		display: inline-flex !important;
		align-items: center;
		justify-content: center;
	}

	.header-row .col-check {
		background-color: #eef2f9;
	}

	/* --- 冻结列样式 (日期 & 小计) --- */
	/* 默认情况 */
	.col-date {
		position: sticky;
		left: 0;
		width: $date-width;
		border-right: 1rpx solid #eee;
	}

	.col-total {
		position: sticky;
		left: $date-width;
		width: $total-width;
		border-right: 1rpx dashed #ccc;
		font-weight: bold;
	}

	/* --- [关键] 删除模式下的偏移处理 --- */
	/* 当 container 有 mode-delete 类名时，覆盖 left 值 */
	.mode-delete {
		.col-date {
			left: $check-width !important;
		}

		.col-total {
			left: calc(#{$date-width} + #{$check-width}) !important;
		}
	}

	/* 层级处理 */
	.header-row .col-date,
	.header-row .col-total {
		background-color: #eef2f9;
		z-index: 110;
	}

	.data-row .col-date,
	.data-row .col-total {
		z-index: 20;
	}

	.data-row:not(.stripe) .col-date,
	.data-row:not(.stripe) .col-total {
		background-color: #fff;
	}

	.data-row.stripe .col-date,
	.data-row.stripe .col-total {
		background-color: #fafafa;
	}

	/* 复选框样式 */
	.checkbox {
		width: 40rpx;
		height: 40rpx;
		border: 2rpx solid #ccc;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 24rpx;
		color: #fff;

		&.checked {
			background-color: #ff3b30;
			border-color: #ff3b30;
		}
	}

	/* 文本颜色 */
	.text-blue {
		color: $text-color-blue !important;
		font-weight: bold;
	}

	.text-red {
		color: $text-color-red !important;
		font-weight: bold;
	}

	/* 加载空状态 */
	.loading-box,
	.empty-box {
		text-align: center;
		padding: 100rpx;
		color: $text-color-sub;
	}

	/* --- 底部区域 (双按钮) --- */
	.footer-area {
		flex-shrink: 0;
		background-color: #fff;
		padding: 20rpx 40rpx;
		box-shadow: 0 -4rpx 10rpx rgba(0, 0, 0, 0.05);
		padding-bottom: calc(20rpx + constant(safe-area-inset-bottom));
		padding-bottom: calc(20rpx + env(safe-area-inset-bottom));
	}

	.footer-btns {
		display: flex;
		gap: 20rpx;
	}

	.action-btn {
		flex: 1;
		height: 88rpx;
		line-height: 88rpx;
		border-radius: 44rpx;
		font-size: 30rpx;
		font-weight: bold;
		border: none;
		color: #fff;

		&:active {
			opacity: 0.9;
		}

		&.disabled {
			background-color: #e0e0e0 !important;
			color: #999;
			pointer-events: none;
		}
	}

	/* 删除按钮 */
	.btn-delete {
		background-color: #ff3b30;

		&.active {
			box-shadow: 0 4rpx 12rpx rgba(255, 59, 48, 0.4);
		}
	}

	/* 修改/取消按钮 */
	.btn-modify {
		background-color: #1a1a1a;

		&.cancel-mode {
			background-color: #999;
		}
	}

	/* 输入框样式 */
	.table-input {
		width: 100%;
		height: 100%;
		min-height: 60rpx;
		text-align: center;
		font-size: 26rpx;
		color: #333;
		background-color: transparent;

		&:focus {
			background-color: #e6f7ff;
			border-radius: 4rpx;
		}
	}

	.input-ph {
		color: #ccc;
	}

	.td {
		padding: 10rpx;
		height: 80rpx;
		line-height: 80rpx;
	}

	/* 弹窗样式 */
	.modal-content-box {
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

	.change-list-scroll {
		max-height: 500rpx;
		background-color: #f8f8f8;
		border-radius: 12rpx;
		padding: 20rpx;
		box-sizing: border-box;
		margin-bottom: 20rpx;
	}

	.change-group {
		margin-bottom: 30rpx;
		border-bottom: 1rpx dashed #e0e0e0;
		padding-bottom: 20rpx;

		&:last-child {
			border-bottom: none;
			margin-bottom: 0;
			padding-bottom: 0;
		}
	}

	.change-date-row {
		display: flex;
		align-items: center;
		margin-bottom: 14rpx;

		.icon-date {
			margin-right: 8rpx;
			font-size: 24rpx;
		}

		.date-text {
			font-weight: bold;
			font-size: 28rpx;
			color: #333;
		}
	}

	.change-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		font-size: 26rpx;
		margin-bottom: 8rpx;
		padding-left: 36rpx;
	}

	.type-name {
		color: #555;
		font-weight: bold;
	}

	.val-box {
		display: flex;
		align-items: center;
	}

	.old-val {
		color: #999;
		text-decoration: line-through;
		margin-right: 10rpx;
	}

	.arrow {
		color: #ccc;
		margin-right: 10rpx;
		font-size: 20rpx;
	}

	.new-val {
		color: #ff3b30;
		font-weight: bold;
	}

	.modal-summary {
		text-align: right;
		font-size: 24rpx;
		color: #666;
		margin-top: 10rpx;
	}

	.highlight-num {
		color: #ff3b30;
		font-weight: bold;
		margin: 0 6rpx;
	}

	.date-picker-cell {
		width: 100%;
		height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 26rpx;
		color: #333;

		.edit-icon {
			font-size: 20rpx;
			color: #999;
			margin-left: 6rpx;
		}
	}

	/* 小计行特殊样式 */
	.summary-row {
		border-top: 1rpx dashed #eee;
		margin-top: 10rpx;
		padding-top: 10rpx;

		.type-name {
			color: #000;
			font-size: 28rpx;
			/* 字号稍微大一点 */
		}
	}

	.summary-val {
		font-weight: 900;
		font-size: 30rpx;
	}

	.delete-modal-content {
		display: flex;
		flex-direction: column;
		align-items: center;
		padding: 10rpx 0;
	}

	.warning-icon {
		width: 90rpx;
		height: 90rpx;
		background-color: #fff2f1;
		color: #ff3b30;
		font-size: 56rpx;
		font-weight: bold;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		margin-bottom: 30rpx;
		border: 4rpx solid #ffcccc;
	}

	.modal-text {
		font-size: 32rpx;
		color: #333;
		text-align: center;
		margin-bottom: 12rpx;
		font-weight: bold;
	}

	.sub-text {
		font-size: 24rpx;
		color: #999;
		margin-bottom: 30rpx;
	}

	.delete-list-scroll {
		width: 100%;
		max-height: 400rpx;
		/* 稍微增加高度 */
		background-color: #f5f7fa;
		border-radius: 16rpx;
		padding: 20rpx;
		box-sizing: border-box;
		margin-top: 20rpx;
		border: 1rpx solid #eee;
	}

	.delete-list-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 16rpx 0;
		border-bottom: 1rpx dashed #e0e0e0;

		&:last-child {
			border-bottom: none;
			padding-bottom: 0;
		}

		&:first-child {
			padding-top: 0;
		}
	}

	.del-left {
		display: flex;
		align-items: center;
	}

	.del-icon {
		font-size: 24rpx;
		margin-right: 10rpx;
	}

	.del-date {
		font-size: 28rpx;
		color: #333;
		font-weight: bold;
	}

	.del-right {
		display: flex;
		align-items: center;
		font-size: 26rpx;
	}

	.del-label {
		color: #999;
		margin-right: 10rpx;
	}

	.del-val {
		font-weight: 900;
		font-size: 30rpx;
		min-width: 60rpx;
		/* 防止数字太短对不齐 */
		text-align: right;
		color: #333;
		/* 默认颜色，会被 text-red/text-blue 覆盖 */
	}

	/* 复用颜色类 (如果之前没定义在全局，这里补充) */
	.text-red {
		color: #ff3b30 !important;
	}

	.text-blue {
		color: #007aff !important;
	}
</style>