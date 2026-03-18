<template>
  <view class="page">
    <view class="card">
      <view class="row main-info">
        <view class="left">
          <view class="name">
            {{ detail.deviceName || '设备名称' }}
          </view>
        </view>
        <view
          class="status-tag"
          :class="detail.deviceStatus === 'on' ? 'on' : 'off'"
        >
          {{ detail.deviceStatus === 'on' ? '开' : '关' }}
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
      detail: {}
    };
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
		data:{sn:this.id，uid:'testuid'},
        success: (res) => {
          if (res.statusCode === 200 && res.data.data) {
            this.detail = res.data.data;
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

.sub {
  margin-top: 8rpx;
  font-size: 26rpx;
  color: #6b7280;
}

.status-tag {
  padding: 8rpx 20rpx;
  border-radius: 999rpx;
  font-size: 26rpx;
}

.status-tag.on {
  background-color: #dcfce7;
  color: #16a34a;
}

.status-tag.off {
  background-color: #e5e7eb;
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

