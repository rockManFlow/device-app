<template>
  <view class="page">
    <!-- 自定义导航栏（新增左侧回退图标） -->
    <view class="custom-nav-bar">
      <!-- 回退图标 -->
      <view class="back-icon" @tap="handleBack">
        <text class="back-text">&lt;</text>
      </view>
      <!-- 页面标题 -->
      <view class="nav-title">设备详情</view>
      <!-- 删除图标 -->
      <view class="delete-icon" @tap="handleDelete">✕</view>
    </view>

    <!-- 原有设备详情内容 -->
    <view class="content-wrap">
      <view class="card">
        <view class="row main-info">
          <view class="left">
            <view class="name">
              <text class="name-label">设备名称: </text>
              <text class="name-value">{{ deviceNameText }}</text>
            </view>
          </view>
          <view class="status-switch" @tap="handleStatusToggleTap">
            <view class="toggle" :class="statusChecked ? 'toggle-on' : 'toggle-off'">
              <view class="toggle-knob"></view>
            </view>
          </view>
        </view>

        <view class="row">
          <text class="label">序列号</text>
          <text class="value">{{ detail.sn || '-' }}</text>
        </view>
        <view class="row">
          <text class="label">设备类型</text>
          <text class="value">{{ detail.deviceType || '-' }}</text>
        </view>
        <view class="row">
          <text class="label">型号</text>
          <text class="value">{{ detail.deviceIcon || '-' }}</text>
        </view>
      </view>

      <view class="card">
        <view class="section-title">设备说明</view>
        <text class="desc">{{ detail.description || '暂无设备说明。' }}</text>
      </view>

      <!-- 设备运行图：温度 / 灯类设备展示折线图，右侧 1 天 / 7 天 -->
      <view v-if="showRuntimeChart" class="card chart-card">
        <view class="chart-header">
          <view class="section-title chart-title">设备运行图</view>
          <view class="range-tabs">
            <view
              class="range-tab"
              :class="{ active: rangeDays === 1 }"
              @tap="setRangeDays(1)"
            >
              1天
            </view>
            <view
              class="range-tab"
              :class="{ active: rangeDays === 7 }"
              @tap="setRangeDays(7)"
            >
              7天
            </view>
          </view>
        </view>
        <view class="chart-y-label">{{ runtimeYAxisLabel }}</view>
        <view v-if="runtimeLoading" class="chart-hint">加载中...</view>
        <view v-else-if="runtimeError" class="chart-hint err">{{ runtimeError }}</view>
        <view v-else-if="!hasRuntimePoints" class="chart-hint">暂无运行数据</view>
        <view v-else class="chart-box">
          <qiun-data-charts
            type="line"
            :opts="chartOpts"
            :chartData="runtimeChartData"
            canvasId="deviceRuntimeChart"
          />
        </view>
      </view>
    </view>
  </view>
</template>

<script>
// 引入常量文件
import configs from '@/common/global-config.js'
import qiunDataCharts from '@qiun/uni-ucharts'

const BASE_URL = configs.BASE_URL
/** 与后端约定：POST，body: { sn, uid, days: 1|7, metric: 'temperature'|'switch' }，返回见 fetchRuntimeSeries 内解析 */
const RUNTIME_SERIES_PATH = '/device/runtime/series'

export default {
  components: {
    qiunDataCharts
  },
  data() {
    return {
      id: null,
      detail: {},
      statusChecked: false,
      updatingStatus: false,
      uid: configs.USER_INFO.UID,
      rangeDays: 1,
      runtimeLoading: false,
      runtimeError: '',
      runtimeChartData: {
        categories: [],
        series: []
      }
    };
  },
  computed: {
    deviceNameText() {
      const d = this.detail || {};
      return d.deviceName || '-';
    },
    /** 温度 / 灯 类型才展示运行图（按 deviceType 文案匹配，可按后端约定调整） */
    runtimeKind() {
      const d = this.detail || {}
      const t = String(d.deviceType || '')
      if (d.deviceType==='temperature') return 'temperature'
      if (d.deviceType==='LIGHT') return 'light'
      return null
    },
    showRuntimeChart() {
      return !!this.runtimeKind
    },
	//返回Y坐标信息
    runtimeYAxisLabel() {
      if (this.runtimeKind === 'temperature') return '温度 (℃)'
      if (this.runtimeKind === 'light') return '开关 (0关 / 1开)'
      return ''
    },
	//是否有数据方法
    hasRuntimePoints() {
      const s = this.runtimeChartData.series && this.runtimeChartData.series[0]
      return s && Array.isArray(s.data) && s.data.length > 0
    },
    chartOpts() {
		//滚动展示标识
      const scroll = (this.runtimeChartData.categories && this.runtimeChartData.categories.length > 8)
      const light = this.runtimeKind === 'light'
      const xAxis = {
        disableGrid: true,
        fontColor: '#9ca3af',
        rotateLabel: scroll
      }
      if (scroll) {
        xAxis.scrollShow = true
        xAxis.itemCount = 6
      }
      return {
        color: ['#3b82f6'],
        padding: [12, 12, 0, 8],
        enableScroll: scroll,
        legend: { show: false },
        xAxis,
        yAxis: {
          gridType: 'dash',
          dashLength: 2,
          data: light
            ? [{ min: 0, max: 1, splitNumber: 1 }]
            : [{}]
        },
        extra: {
          line: {
            type: 'straight',
            width: 2
          }
        }
      }
    }
  },
  onLoad(options) {
    if (options && options.id) {
      this.id = options.id;
      this.fetchDetail();
    } else {
      uni.showToast({ title: '缺少设备ID', icon: 'none' });
    }
  },
  methods: {
    // 新增：回退到上一页
    handleBack() {
		//直接跳转到首页
		uni.redirectTo({ url: '/pages/index/index' });
      // uni.navigateBack 是uni-app原生返回上一页的API
      // uni.navigateBack({
      //   delta: 1, // 返回1级页面（上一页）
      //   fail: () => {
      //     // 兼容：如果没有上一页，跳转到首页
      //     uni.redirectTo({ url: '/pages/index/index' });
      //   }
      // });
    },

    // 获取设备详情（原有逻辑）
    fetchDetail() {
      uni.request({
        url: `${BASE_URL}/device/detail`,
        method: 'POST',
        header: { 'Content-Type': 'application/json' },
        data: { sn: this.id, uid: this.uid },
        success: (res) => {
          if (res.statusCode === 200 && res.data.data) {
            this.detail = res.data.data;
            this.statusChecked = this.detail.deviceStatus === 'on';
            if (this.runtimeKind) {
              this.runtimeLoading = true;
              this.runtimeError = '';
            }
            this.$nextTick(() => {
              if (this.runtimeKind) this.fetchRuntimeSeries();
            });
          } else {
            uni.showToast({ title: '详情数据异常', icon: 'none' });
          }
        },
        fail: () => {
          uni.showToast({ title: '获取详情失败', icon: 'none' });
        }
      });
    },

    setRangeDays(days) {
      if (this.rangeDays === days || this.runtimeLoading) 
		return;
      this.rangeDays = days;
      if (this.runtimeKind && this.id) 
		this.fetchRuntimeSeries();
    },

    /** 将接口返回解析为 { ts, value }[]，ts 统一为毫秒时间戳 */
    normalizeSeriesPayload(raw) {
      let list = [];
      if (Array.isArray(raw)) 
		list = raw;
	  else
		return [];
      const out = [];
      for (let i = 0; i < list.length; i++) {
        const p = list[i];
        let ts = p.ts;
        let val = p.value;
		//{ ts: ts, value: val }，属于标准的 JavaScript 普通对象
        out.push({ ts, value: val });//数组添加数据
      }
      out.sort((a, b) => a.ts - b.ts);
      return out;
    },

	//格式化时间格式
    formatTick(ts, days) {
      const d = new Date(ts);
      const h = String(d.getHours()).padStart(2, '0');//padStart确保小时字符串的长度至少为 2 位，不足的话在开头补 0
      const m = String(d.getMinutes()).padStart(2, '0');
	  const s = String(d.getSeconds()).padStart(2, '0');
      const mo = String(d.getMonth() + 1).padStart(2, '0');
      const day = String(d.getDate()).padStart(2, '0');
      if (days <= 1) return `${h}:${m}:${s}`;
      return `${mo}-${day} ${h}:${m}`;
    },

	//处理响应数据设置给图标参数
    buildRuntimeChart(points) {
      const cats = points.map((p) => this.formatTick(p.ts, this.rangeDays));
      const seriesName =
        this.runtimeKind === 'temperature' ? '温度' : '开关';
      this.runtimeChartData = {
		  //横坐标时间格式
        categories: cats,
        series: [{ name: seriesName, data: points.map((p) => p.value) }]
      };
    },

	//从后端接口获取图表数据
    fetchRuntimeSeries() {
      if (!this.id || !this.runtimeKind) return;
      // const metric = this.runtimeKind === 'temperature' ? 'temperature' : 'switch';
      this.runtimeLoading = true;
      this.runtimeError = '';
      uni.request({
        url: `${BASE_URL}${RUNTIME_SERIES_PATH}`,
        method: 'POST',
        header: { 'Content-Type': 'application/json' },
        data: {
          sn: this.id,
          uid: this.uid,
          days: this.rangeDays
        },
        success: (res) => {
          if (res.statusCode !== 200) {
            this.runtimeError = '加载运行数据失败';
            this.runtimeChartData = { categories: [], series: [] };
            return;
          }
          const payload = res.data && (res.data.data !== undefined ? res.data.data : res.data);
          const points = this.normalizeSeriesPayload(payload);
          if (!points.length) {
            this.runtimeChartData = { categories: [], series: [] };
            return;
          }
          this.buildRuntimeChart(points);
        },
        fail: () => {
          this.runtimeError = '网络异常，无法加载运行图';
          this.runtimeChartData = { categories: [], series: [] };
        },
        complete: () => {
          this.runtimeLoading = false;
        }
      });
    },

    // 开关状态切换（原有逻辑）
    handleStatusToggleTap() {
      if (this.updatingStatus) return;
      const nextChecked = !this.statusChecked;
      const nextStatus = nextChecked ? 'on' : 'off';
      const actionText = nextChecked ? '开启' : '关闭';

      uni.showModal({
        title: '确认操作',
        content: `是否确认【${actionText}】该设备？`,
        confirmText: '确定',
        cancelText: '取消',
        success: (r) => {
          if (r.confirm) this.updateDeviceStatus(nextStatus);
        }
      });
    },

    // 更新设备状态（原有逻辑）
    updateDeviceStatus(nextStatus) {
      if (!this.id) return;
      this.updatingStatus = true;
      uni.showLoading({ title: '处理中...' });
      uni.request({
        url: `${BASE_URL}/device/alter/status`,
        method: 'POST',
        header: { 'Content-Type': 'application/json' },
        data: { sn: this.id, uid: this.uid, deviceStatus: nextStatus },
        success: (res) => {
          if (res.statusCode === 200) {
            this.detail = { ...this.detail, deviceStatus: nextStatus };
            this.statusChecked = nextStatus === 'on';
            uni.showToast({ title: '操作成功', icon: 'success' });
          } else {
            uni.showToast({ title: '操作失败', icon: 'none' });
          }
        },
        fail: () => {
          uni.showToast({ title: '操作失败', icon: 'none' });
        },
        complete: () => {
          uni.hideLoading();
          this.updatingStatus = false;
        }
      });
    },

    // 点击删除图标触发确认弹窗（原有逻辑）
    handleDelete() {
      if (!this.id) return;
      uni.showModal({
        title: '删除设备',
        content: '确认【删除】该设备吗？',
        confirmText: '确认',
        cancelText: '取消',
        success: (r) => {
          if (r.confirm) this.deleteDevice();
        }
      });
    },

    // 调用删除接口（原有逻辑）
    deleteDevice() {
      uni.showLoading({ title: '删除中...' });
      uni.request({
        url: `${BASE_URL}/device/delete`,
        method: 'POST',
        header: { 'Content-Type': 'application/json' },
        data: {
          sn: this.id,
          uid: this.uid
        },
        success: (res) => {
          if (res.statusCode === 200) {
            uni.showToast({ title: '删除成功', icon: 'success' });
            setTimeout(() => {
              uni.reLaunch({ url: '/pages/index/index' });
            }, 600);
          } else {
            uni.showToast({ title: '删除失败', icon: 'none' });
          }
        },
        fail: () => {
          uni.showToast({ title: '删除失败', icon: 'none' });
        },
        complete: () => {
          uni.hideLoading();
        }
      });
    }
  }
};
</script>

<style scoped>
/* 页面基础样式 */
.page {
  background-color: #f5f7fb;
  min-height: 100vh;
  box-sizing: border-box;
}

/* 自定义导航栏样式（新增回退图标布局） */
.custom-nav-bar {
  height: 44px; /* 适配小程序导航栏高度 */
  line-height: 44px;
  padding: 0 24rpx;
  background-color: #F8F8F8;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  border-bottom: 1px solid #eee;
}

/* 回退图标样式 */
.back-icon {
  position: absolute;
  left: 24rpx;
  top: 50%;
  transform: translateY(-50%);
  width: 48rpx;
  height: 48rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #000;
  font-size: 32rpx;
  cursor: pointer;
}

.back-icon:active {
  opacity: 0.8; /* 点击反馈 */
}

.back-text {
  font-weight: 500;
}

/* 导航标题（居中） */
.nav-title {
  font-size: 36rpx;
  font-weight: 500;
  color: #000;
}

/* 删除图标样式（不变） */
.delete-icon {
  position: absolute;
  right: 24rpx;
  top: 50%;
  transform: translateY(-50%);
  width: 48rpx;
  height: 48rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #ff4d4f; /* 红色 */
  font-size: 32rpx;
  cursor: pointer;
}

.delete-icon:active {
  opacity: 0.8;
}

/* 内容区域（避开导航栏） */
.content-wrap {
  padding: 24rpx;
}

/* 原有样式保留 */
.card {
  background-color: #ffffff;
  border-radius: 24rpx;
  padding: 24rpx 24rpx 18rpx;
  margin-bottom: 24rpx;
}

.main-info {
  margin-bottom: 24rpx;
}

.row {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  padding: 10rpx 0;
}

.left {
  flex: 1;
}

.name {
  font-size: 32rpx;
  font-weight: 600;
  color: #111827;
}

.name-label {
  color: #6b7280;
  font-size: 26rpx;
  font-weight: 500;
  margin-right: 12rpx;
}

.name-value {
  color: #111827;
  font-size: 32rpx;
  font-weight: 600;
}

.status-switch {
  margin-left: 16rpx;
  flex-shrink: 0;
}

.toggle {
  width: 96rpx;
  height: 54rpx;
  border-radius: 999rpx;
  padding: 6rpx;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  background-color: #e5e7eb;
  transition: background-color 0.18s ease;
}

.toggle-on {
  background-color: #3b82f6;
  justify-content: flex-end;
}

.toggle-off {
  background-color: #e5e7eb;
  justify-content: flex-start;
}

.toggle-knob {
  width: 42rpx;
  height: 42rpx;
  border-radius: 50%;
  background-color: #ffffff;
  box-shadow: 0 6rpx 14rpx rgba(17, 24, 39, 0.18);
}

.label {
  font-size: 26rpx;
  color: #6b7280;
}

.value {
  font-size: 26rpx;
  color: #111827;
  margin-left: 40rpx;
}

.section-title {
  font-size: 28rpx;
  font-weight: 500;
  color: #111827;
  margin-bottom: 12rpx;
}

.desc {
  font-size: 26rpx;
  color: #4b5563;
  line-height: 1.6;
}

.chart-card {
  padding-top: 20rpx;
}

.chart-header {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8rpx;
}

.chart-title {
  margin-bottom: 0;
  flex-shrink: 0;
}

.range-tabs {
  display: flex;
  flex-direction: row;
  align-items: center;
  border-radius: 12rpx;
  background: #f3f4f6;
  padding: 4rpx;
}

.range-tab {
  padding: 10rpx 28rpx;
  font-size: 24rpx;
  color: #6b7280;
  border-radius: 10rpx;
}

.range-tab.active {
  background: #ffffff;
  color: #3b82f6;
  font-weight: 500;
  box-shadow: 0 2rpx 8rpx rgba(59, 130, 246, 0.15);
}

.chart-y-label {
  font-size: 22rpx;
  color: #9ca3af;
  margin-bottom: 12rpx;
}

.chart-hint {
  font-size: 26rpx;
  color: #9ca3af;
  text-align: center;
  padding: 48rpx 0;
}

.chart-hint.err {
  color: #ef4444;
}

.chart-box {
  width: 100%;
  height: 420rpx;
}
</style>