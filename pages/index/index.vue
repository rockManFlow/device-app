<template>
  <view class="page">
    <!-- 顶部：用户信息 + 添加按钮 -->
    <view class="header">
      <view class="user-info">
        <image
          class="avatar"
          :src="user.avatar"
          mode="aspectFill"
        />
        <view class="user-name">
          {{ user.name }}
        </view>
      </view>
      <view
        class="add-btn"
        @tap="handleAddDevice"
      >
        +
      </view>
    </view>

    <!-- 中部：图片占位 -->
    <view class="banner">
      <image
        class="banner-image"
        src="@/static/home-placeholder.jpeg"
        mode="aspectFill"
      />
      <text class="banner-text">智能家居 · 简洁生活</text>
    </view>

    <!-- 底部：设备列表 -->
    <view class="device-section">
      <view class="section-title">
        已添加设备
      </view>
      <view
        v-if="loading"
        class="hint-text"
      >
        加载中...
      </view>
      <view
        v-else-if="devices.length === 0"
        class="hint-text"
      >
        暂无设备，请点击右上角“+”添加。
      </view>
      <scroll-view
        v-else
        scroll-y
        class="device-list"
      >
        <view
          v-for="item in devices"
          :key="item.sn"
          class="device-item"
          @tap="goDetail(item)"
        >
          <image
            class="device-icon"
            :src="item.icon || defaultDeviceIcon"
            mode="aspectFill"
          />
          <view class="device-main">
            <view class="device-name">
              {{ item.deviceName }}
            </view>
            <view class="device-desc">
              {{ item.deviceType || '未设置位置' }}
            </view>
          </view>
          <view
            class="device-status"
            :class="item.status === 'on' ? 'status-on' : 'status-off'"
          >
            {{ item.status === 'on' ? '开' : '关' }}
          </view>
        </view>
      </scroll-view>
    </view>
  </view>
</template>

<script>
const BASE_URL = 'http://127.0.0.1:8820';

export default {
  data() {
    return {
      user: {
        name: '用户名',
        avatar: '/static/head-photo.png'
      },
      devices: [],
      loading: false,
      defaultDeviceIcon: '/static/device-default.png'
    };
  },
  onShow() {
    this.fetchDevices();
  },
  methods: {
    fetchDevices() {
      this.loading = true;
      uni.request({
        url: `${BASE_URL}/devices`,
        method: 'POST',
		header: {
		          'Content-Type': 'application/json' 
		          // 可选：如果接口需要token，添加请求头
		          // 'Authorization': 'Bearer ' + uni.getStorageSync('token')
		        },
		data:{uid:'asccde11111'},
        success: (res) => {
			console.info(res);
          if (res.statusCode === 200 && Array.isArray(res.data)) {
            this.devices = res.data;
          } else {
            uni.showToast({
              title: '设备数据异常',
              icon: 'none'
            });
          }
        },
        fail: (err) => {
		  console.error('请求失败：', err);
		  uni.showToast({
			title: '获取设备失败',
			icon: 'none'
		  });
        },
        complete: () => {
          this.loading = false;
        }
      });
    },
    handleAddDevice() {
      uni.scanCode({
        onlyFromCamera: true,
		// 扫码类型：qrCode（二维码）、barCode（条形码），默认都支持
		scanType:['qrCode'],
        success: (res) => {
		  console.log('扫码成功：', res);
		  //设备id xxxx
          const result = res.result;// 存储扫码结果（二维码内容）
          uni.request({
            url: `${BASE_URL}/devices/bind`,
            method: 'POST',
            data: { uid:'',sn:result },
            success: (resp) => {			
			  console.log('扫码成功：', resp);
              if (resp.statusCode === 200) {
                uni.showToast({
                  title: '添加成功',
                  icon: 'success'
                });
                this.fetchDevices();
              } else {
                uni.showToast({
                  title: '添加失败:'+(resp.getMsg||'fail'),
                  icon: 'none'
                });
              }
            },
            fail: (err) => {
              uni.showToast({
                title: '扫码失败：' + (err.errMsg || '取消扫码'),
                icon: 'none'
              });
            }
          });
        },
		// 扫码失败的回调（如用户取消、权限不足）
        fail: (err) => {
		  console.error('扫码失败：', err);
          uni.showToast({
                title: '扫码失败：' + (err.errMsg || '取消扫码'),
                icon: 'none'
              });
        }
      });
    },
    goDetail(item) {
      uni.navigateTo({
        url: `/pages/device_detail/device_detail?id=${item.id}`
      });
    }
  }
};
</script>

<style scoped>
.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #f5f7fb;
  padding: 16rpx 24rpx 24rpx;
  box-sizing: border-box;
}

.header {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  margin-top: 24rpx;
}

.user-info {
  display: flex;
  flex-direction: row;
  align-items: center;
}

.avatar {
  width: 80rpx;
  height: 80rpx;
  border-radius: 50%;
  background-color: #e0e3ee;
}

.user-name {
  margin-left: 16rpx;
  font-size: 30rpx;
  font-weight: 500;
  color: #222;
}

.add-btn {
  width: 64rpx;
  height: 64rpx;
  border-radius: 50%;
  border-width: 2rpx;
  border-style: solid;
  border-color: #3b82f6;
  color: #3b82f6;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 40rpx;
}

.banner {
  margin-top: 32rpx;
  padding: 24rpx;
  border-radius: 24rpx;
  background: linear-gradient(135deg, #e0f2fe, #f5f7fb);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.banner-image {
  width: 100%;
  height: 260rpx;
  border-radius: 20rpx;
  background-color: #d4dbe9;
}

.banner-text {
  margin-top: 16rpx;
  font-size: 26rpx;
  color: #555;
}

.device-section {
  flex: 1;
  margin-top: 32rpx;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.section-title {
  font-size: 28rpx;
  font-weight: 500;
  color: #222;
  margin-bottom: 16rpx;
}

.hint-text {
  font-size: 26rpx;
  color: #888;
  margin-top: 24rpx;
  text-align: center;
}

.device-list {
  flex: 1;
}

.device-item {
  flex-direction: row;
  display: flex;
  align-items: center;
  padding: 20rpx 16rpx;
  border-radius: 20rpx;
  background-color: #ffffff;
  margin-bottom: 16rpx;
}

.device-icon {
  width: 72rpx;
  height: 72rpx;
  border-radius: 20rpx;
  background-color: #e0e3ee;
}

.device-main {
  flex: 1;
  margin-left: 20rpx;
}

.device-name {
  font-size: 28rpx;
  color: #222;
}

.device-desc {
  margin-top: 6rpx;
  font-size: 24rpx;
  color: #999;
}

.device-status {
  font-size: 26rpx;
  padding: 6rpx 18rpx;
  border-radius: 999rpx;
}

.status-on {
  background-color: #dcfce7;
  color: #16a34a;
}

.status-off {
  background-color: #e5e7eb;
  color: #6b7280;
}
</style>

