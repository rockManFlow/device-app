<template>
  <view class="page">
    <!-- 自定义导航栏：顶部留出状态栏安全区 -->
    <view
      class="custom-nav-bar"
      :style="{ paddingTop: statusBarHeight + 'px' }"
    >
      <!-- 回退图标 -->
      <view class="back-icon" @tap="handleBack">
        <text class="back-text">&lt;</text>
      </view>
      <!-- 页面标题 -->
      <view class="nav-title">设备详情</view>
      <!-- 删除图标 -->
      <view class="delete-icon" @tap="handleDelete">✕</view>
    </view>

    <view v-if="detailLoading" class="page-loading">
      <text class="page-loading-text">加载中...</text>
    </view>
    <view v-else class="content-wrap">
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
          <text class="label">当前状态</text>
          <text class="value status-text" :class="statusChecked ? 'status-text-on' : 'status-text-off'">
            {{ statusChecked ? '开启' : '关闭' }}
          </text>
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
              :class="{ active: rangeDays === 1, disabled: runtimeLoading }"
              @tap="setRangeDays(1)"
            >
              1天
            </view>
            <view
              class="range-tab"
              :class="{ active: rangeDays === 7, disabled: runtimeLoading }"
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
            :ontouch="runtimeChartScrollEnabled"
            :disable-scroll="runtimeChartScrollEnabled"
          />
        </view>

        <!-- 灯类设备：开关状态占比扇形图（与上方 1 天 / 7 天 Tab 一致） -->
        <view
          v-if="runtimeKind === 'light' && !runtimeLoading && !runtimeError && hasRuntimePoints"
          class="light-pie-section"
        >
          <view class="light-pie-title">开关状态占比</view>
          <text class="light-pie-range-hint">{{ lightPieRangeHint }}</text>
          <view v-if="hasLightPieData" class="light-pie-body">
            <view class="light-pie-chart-box">
              <qiun-data-charts
                type="pie"
                :opts="lightPieOpts"
                :chartData="lightPieChartData"
                canvasId="deviceLightPieChart"
                tooltip-format="lightPieTip"
                :tooltip-show="true"
                :ontouch="true"
              />
            </view>
            <view class="light-pie-stats">
              <view class="light-pie-stat light-pie-stat-on">
                <view class="light-pie-stat-dot" />
                <view class="light-pie-stat-text">
                  <text class="light-pie-stat-name">开启</text>
                  <text class="light-pie-stat-pct">{{ lightSwitchStats.onPctText }}</text>
                  <text class="light-pie-stat-tip">上报为「开」的数据点占比</text>
                </view>
              </view>
              <view class="light-pie-stat light-pie-stat-off">
                <view class="light-pie-stat-dot" />
                <view class="light-pie-stat-text">
                  <text class="light-pie-stat-name">关闭</text>
                  <text class="light-pie-stat-pct">{{ lightSwitchStats.offPctText }}</text>
                  <text class="light-pie-stat-tip">上报为「关」的数据点占比</text>
                </view>
              </view>
            </view>
          </view>
          <view v-else class="chart-hint light-pie-empty">暂无有效开关采样数据</view>
          <text v-if="hasLightPieData" class="light-pie-footnote">说明：占比为有效采样点统计，与实际上电时长可能略有差异。</text>
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
      detailLoading: false,
      statusBarHeight: 0,
      statusChecked: false,
      updatingStatus: false,
      uid: configs.USER_INFO.UID,
      rangeDays: 1,
      runtimeLoading: false,
      runtimeError: '',
      runtimeChartData: {
        categories: [],
        series: []
      },
      /** 原始运行序列，用于灯类开关占比扇形图 */
      runtimeRawPoints: []
    };
  },
  computed: {
    deviceNameText() {
      const d = this.detail || {};
      return d.deviceName || '-';
    },
    /** 温度 / 灯 类型才展示运行图（兼容大小写） */
    runtimeKind() {
      const d = this.detail || {}
      const t = String(d.deviceType || '').toLowerCase()
      if (t === 'temperature') return 'temperature'
      if (t === 'light') return 'light'
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
    /** 数据点多于 8 个时开启横向滚动；需配合 qiun 的 ontouch，否则小程序端无法滑动 */
    runtimeChartScrollEnabled() {
      const c = this.runtimeChartData.categories
      return !!(c && c.length > 8)
    },
    chartOpts() {
		//滚动展示标识
      const scroll = this.runtimeChartScrollEnabled
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
    },
    lightPieRangeHint() {
      return this.rangeDays <= 1 ? '统计范围：近 1 天（与上方 Tab 一致）' : '统计范围：近 7 天（与上方 Tab 一致）'
    },
    lightSwitchStats() {
      return this.computeLightSwitchStats(this.runtimeRawPoints)
    },
    hasLightPieData() {
      return this.lightSwitchStats.total > 0
    },
    lightPieChartData() {
      const s = this.lightSwitchStats
      if (!s.total) {
        return { series: [{ data: [] }] }
      }
      const data = []
      if (s.onCount > 0) {
        data.push({
          name: '开启',
          value: s.onCount,
          tooltipText: `开启：占比 ${s.onPctText}（${s.onCount} 个上报点）`
        })
      }
      if (s.offCount > 0) {
        data.push({
          name: '关闭',
          value: s.offCount,
          tooltipText: `关闭：占比 ${s.offPctText}（${s.offCount} 个上报点）`
        })
      }
      return { series: [{ data }] }
    },
    lightPieOpts() {
      return {
        color: ['#3b82f6', '#9ca3af'],
        fontSize: 12,
        padding: [10, 10, 10, 10],
        dataLabel: false,
        legend: { show: false },
        extra: {
          pie: {
            activeOpacity: 0.88,
            activeRadius: 12,
            offsetAngle: 0,
            border: true,
            borderWidth: 2,
            borderColor: '#FFFFFF'
          },
          tooltip: {
            showBox: true,
            showArrow: true,
            bgColor: '#111827',
            bgOpacity: 0.92,
            fontColor: '#FFFFFF',
            fontSize: 13,
            lineHeight: 22,
            legendShow: false,
            legendShape: 'circle',
            boxPadding: 4
          }
        }
      }
    }
  },
  onLoad(options) {
    try {
      const sys = uni.getSystemInfoSync()
      this.statusBarHeight = (sys && sys.statusBarHeight) ? sys.statusBarHeight : 0
    } catch (e) {
      this.statusBarHeight = 0
    }
    if (options && options.id) {
      this.id = options.id;
      this.fetchDetail();
    } else {
      uni.showToast({ title: '缺少设备ID', icon: 'none' });
    }
  },
  methods: {
    /** 将单点 value 解析为开 true / 关 false / 无法识别 null（与折线图 0/1 约定一致） */
    normalizeSwitchState(val) {
      if (val === true || val === 1) return true
      if (val === false || val === 0) return false
      if (val == null || val === '') return null
      if (typeof val === 'number') {
        if (val > 0.5) return true
        if (val <= 0.5) return false
      }
      const s = String(val).toLowerCase().trim()
      if (s === 'on' || s === '1' || s === 'true' || s === '开') return true
      if (s === 'off' || s === '0' || s === 'false' || s === '关') return false
      return null
    },
    computeLightSwitchStats(points) {
      let onCount = 0
      let offCount = 0
      if (!Array.isArray(points)) {
        return { onCount: 0, offCount: 0, total: 0, onPctText: '0%', offPctText: '0%' }
      }
      for (let i = 0; i < points.length; i++) {
        const st = this.normalizeSwitchState(points[i].value)
        if (st === true) onCount++
        else if (st === false) offCount++
      }
      const total = onCount + offCount
      const fmt = (n, d) => {
        if (!d) return '0%'
        const p = Math.round((n / d) * 1000) / 10
        return `${p.toFixed(1)}%`
      }
      return {
        onCount,
        offCount,
        total,
        onPctText: fmt(onCount, total),
        offPctText: fmt(offCount, total)
      }
    },
    handleBack() {
		uni.redirectTo({ url: '/pages/index/index' })
      // uni.navigateBack({
      //   delta: 1,
      //   fail: () => {
      //     uni.redirectTo({ url: '/pages/index/index' })
      //   }
      // })
    },

    fetchDetail() {
      this.detailLoading = true
      uni.request({
        url: `${BASE_URL}/device/detail`,
        method: 'POST',
        header: { 'Content-Type': 'application/json' },
        data: { sn: this.id, uid: this.uid },
        success: (res) => {
          if (res.statusCode === 200 && res.data.data) {
            this.detail = res.data.data;
            this.statusChecked = this.detail.deviceStatus === 'on';
            if (!this.runtimeKind) {
              this.runtimeRawPoints = [];
            }
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
        },
        complete: () => {
          this.detailLoading = false
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

    /** 将接口返回解析为 { ts, value }[]，ts 统一为毫秒时间戳（兼容秒级时间戳） */
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
        const n = Number(ts);
        if (Number.isFinite(n) && n > 0 && n < 1e12) {
          ts = n * 1000;
        }
        const val = p.value;
        out.push({ ts, value: val });
      }
      out.sort((a, b) => a.ts - b.ts);
      return out;
    },

    formatTick(ts, days) {
      const d = new Date(ts);
      if (Number.isNaN(d.getTime())) return '-';
      const h = String(d.getHours()).padStart(2, '0');
      const m = String(d.getMinutes()).padStart(2, '0');
      const s = String(d.getSeconds()).padStart(2, '0');
      const mo = String(d.getMonth() + 1).padStart(2, '0');
      const day = String(d.getDate()).padStart(2, '0');
      if (days <= 1) return `${h}:${m}:${s}`;
      return `${mo}-${day} ${h}:${m}`;
    },

    buildRuntimeChart(points) {
      const cats = points.map((p) => this.formatTick(p.ts, this.rangeDays));
      const seriesName =
        this.runtimeKind === 'temperature' ? '温度' : '开关';
      const seriesData = points.map((p) => {
        if (this.runtimeKind !== 'light') return p.value;
        const st = this.normalizeSwitchState(p.value);
        if (st === true) return 1;
        if (st === false) return 0;
        const n = Number(p.value);
        return Number.isFinite(n) ? n : 0;
      });
      this.runtimeChartData = {
        categories: cats,
        series: [{ name: seriesName, data: seriesData }]
      };
    },

	//从后端接口获取图表数据
    fetchRuntimeSeries() {
      if (!this.id || !this.runtimeKind) return;
      // const metric = this.runtimeKind === 'temperature' ? 'temperature' : 'switch';
      this.runtimeLoading = true;
      this.runtimeError = '';
      this.runtimeRawPoints = [];
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
            this.runtimeRawPoints = [];
            return;
          }
          const payload = res.data && (res.data.data !== undefined ? res.data.data : res.data);
          const points = this.normalizeSeriesPayload(payload);
          if (!points.length) {
            this.runtimeChartData = { categories: [], series: [] };
            this.runtimeRawPoints = [];
            return;
          }
          this.runtimeRawPoints = points;
          this.buildRuntimeChart(points);
        },
        fail: () => {
          this.runtimeError = '网络异常，无法加载运行图';
          this.runtimeChartData = { categories: [], series: [] };
          this.runtimeRawPoints = [];
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
            if (this.runtimeKind && this.id) {
              this.$nextTick(() => this.fetchRuntimeSeries());
            }
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
  padding-bottom: constant(safe-area-inset-bottom);
  padding-bottom: env(safe-area-inset-bottom);
}

/* 自定义导航栏：内容区 44px + 动态 paddingTop 状态栏 */
.custom-nav-bar {
  min-height: 44px;
  line-height: 44px;
  padding-left: 24rpx;
  padding-right: 24rpx;
  padding-bottom: 0;
  box-sizing: content-box;
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

.page-loading {
  padding: 120rpx 24rpx;
  display: flex;
  align-items: center;
  justify-content: center;
}

.page-loading-text {
  font-size: 28rpx;
  color: #9ca3af;
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

.status-text-on {
  color: #3b82f6;
  font-weight: 500;
}

.status-text-off {
  color: #6b7280;
  font-weight: 500;
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

.range-tab.disabled {
  opacity: 0.45;
  pointer-events: none;
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

.light-pie-section {
  margin-top: 28rpx;
  padding-top: 24rpx;
  border-top: 1px solid #f3f4f6;
}

.light-pie-title {
  font-size: 28rpx;
  font-weight: 500;
  color: #111827;
  margin-bottom: 8rpx;
  text-align: center;
}

.light-pie-range-hint {
  display: block;
  font-size: 22rpx;
  color: #9ca3af;
  margin-bottom: 20rpx;
  line-height: 1.5;
  text-align: center;
}

/* 扇形图在左、说明在右同一行（窄屏过窄时允许换行，仍居中） */
.light-pie-body {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  gap: 20rpx 28rpx;
  width: 100%;
  box-sizing: border-box;
}

.light-pie-chart-box {
  width: 480rpx;
  height: 480rpx;
  flex-shrink: 0;
  max-width: 100%;
  box-sizing: border-box;
}

.light-pie-stats {
  flex: 0 1 210rpx;
  min-width: 180rpx;
  max-width: 240rpx;
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 24rpx;
}

.light-pie-stat {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  justify-content: flex-start;
  width: 100%;
}

.light-pie-stat-dot {
  width: 16rpx;
  height: 16rpx;
  border-radius: 50%;
  margin-top: 10rpx;
  margin-right: 16rpx;
  flex-shrink: 0;
}

.light-pie-stat-on .light-pie-stat-dot {
  background-color: #3b82f6;
}

.light-pie-stat-off .light-pie-stat-dot {
  background-color: #9ca3af;
}

.light-pie-stat-text {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  text-align: left;
  flex: 1;
  min-width: 0;
}

.light-pie-stat-name {
  font-size: 28rpx;
  font-weight: 500;
  color: #111827;
  margin-bottom: 6rpx;
}

.light-pie-stat-pct {
  font-size: 36rpx;
  font-weight: 600;
  color: #1f2937;
  margin-bottom: 8rpx;
}

.light-pie-stat-tip {
  font-size: 22rpx;
  color: #6b7280;
  line-height: 1.45;
}

.light-pie-footnote {
  display: block;
  margin-top: 20rpx;
  font-size: 22rpx;
  color: #9ca3af;
  line-height: 1.5;
  text-align: center;
}

.light-pie-empty {
  padding: 32rpx 0 8rpx;
  text-align: center;
}
</style>