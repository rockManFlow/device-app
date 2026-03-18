<template>
  <view class="page">
    <view class="top-actions">
      <view class="delete-icon" @tap="handleDelete">✕</view>
    </view>
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
      <view class="section-title">
        设备说明
      </view>
      <text class="desc">
        {{ detail.description || '暂无设备说明。' }}
      </text>
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
      return (
        d.deviceName ||
        d.name ||
        d.device_name ||
        d.deviceTitle ||
        '-'
      );
    }
  },
  onLoad(options) {
    if (options && options.id) {
      this.id = options.id;
      this.fetchDetail();
    } else {
      uni.showToast({
        title: '缺少设备ID',
        icon: 'none'
      });
    }
  },
  methods: {
    fetchDetail() {
      uni.request({
        url: `${BASE_URL}/device/detail`,
        method: 'POST',
		header: {
		          'Content-Type': 'application/json' 
		          // 可选：如果接口需要token，添加请求头
		          // 'Authorization': 'Bearer ' + uni.getStorageSync('token')
		        },
		data:{sn:this.id,uid:this.uid},
        success: (res) => {
          if (res.statusCode === 200 && res.data.data) {
            this.detail = res.data.data;
            this.statusChecked = (this.detail.deviceStatus === 'on');
          } else {
            uni.showToast({
              title: '详情数据异常',
              icon: 'none'
            });
          }
        },
        fail: () => {
          uni.showToast({
            title: '获取详情失败',
            icon: 'none'
          });
        }
      });
    },
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
    updateDeviceStatus(nextStatus) {
      if (!this.id) return;
      this.updatingStatus = true;
      uni.showLoading({ title: '处理中...' });
      uni.request({
        url: `${BASE_URL}/device/status`,
        method: 'POST',
        header: {
          'Content-Type': 'application/json'
        },
        data: {
          sn: this.id,
          uid: this.uid,
          deviceStatus: nextStatus
        },
        success: (res) => {
          if (res.statusCode === 200) {
            this.detail = {
              ...(this.detail || {}),
              deviceStatus: nextStatus
            };
            this.statusChecked = (nextStatus === 'on');
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
    deleteDevice() {
      uni.showLoading({ title: '删除中...' });
      uni.request({
        url: `${BASE_URL}/device/delete`,
        method: 'POST',
        header: {
          'Content-Type': 'application/json'
        },
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
.page {
  padding: 24rpx;
  background-color: #f5f7fb;
  min-height: 100%;
  box-sizing: border-box;
}

.top-actions {
  position: fixed;
  top: 18rpx;
  right: 24rpx;
  z-index: 20;
}

.delete-icon {
  width: 48rpx;
  height: 48rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fca5a5;           /* 浅淡的红色 */
  font-size: 32rpx;
  opacity: 0.6;             /* 弱化透明度 */
  cursor: pointer;
  transition: opacity 0.2s ease;
}

.delete-icon:hover {
  opacity: 0.8;             /* 悬停时增加透明度 */
}

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

.sub {
  margin-top: 8rpx;
  font-size: 26rpx;
  color: #6b7280;
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

