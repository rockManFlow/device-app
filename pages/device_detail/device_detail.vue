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
    </view>
  </view>
</template>

<script>
const BASE_URL = 'http://127.0.0.1:8820';

export default {
  data() {
    return {
      id: null,
      detail: {},
      statusChecked: false,
      updatingStatus: false,
      uid: 'testuid'
    };
  },
  computed: {
    deviceNameText() {
      const d = this.detail || {};
      return d.deviceName || d.name || d.device_name || d.deviceTitle || '-';
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
      // uni.navigateBack 是uni-app原生返回上一页的API
      uni.navigateBack({
        delta: 1, // 返回1级页面（上一页）
        fail: () => {
          // 兼容：如果没有上一页，跳转到首页
          uni.redirectTo({ url: '/pages/index/index' });
        }
      });
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
          } else {
            uni.showToast({ title: '详情数据异常', icon: 'none' });
          }
        },
        fail: () => {
          uni.showToast({ title: '获取详情失败', icon: 'none' });
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
        content: `是否确认${actionText}该设备？`,
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
        url: `${BASE_URL}/device/status`,
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
        content: '确认删除该设备吗？删除后不可恢复。',
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
</style>