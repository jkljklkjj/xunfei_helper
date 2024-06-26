<template>
  <view class="recorder">
    <view class="re-top" v-if="showTop">
      <view class="re-cancel" @click="cancel">取消</view>
      <view class="re-confirm" :style="{color: theme}" @click="confirm">{{ confirmText }}</view>
    </view>
    <text class="title">{{ finish ? '点击播放' : '点击录制语音' }}</text>
    <view class="recorder-box" 
      v-if="!finish"
      @click="handle">
      <u-circle-progress :active-color="theme" :duration="0" :percent="calcProgress">
        <view class="u-progress-content">
          <image src="/static/sound-recording/voice.png" mode="aspectFit" :style="{
            width: width,
            height: height
          }"></image>
        </view>
      </u-circle-progress>
    </view>
    <view class="recorder-box" 
      v-else
      @click="playVoice">
      <u-circle-progress :active-color="theme" :duration="0" :percent="playProgress">
        <view class="u-progress-content">
          <image src="/static/sound-recording/play.png" mode="aspectFit" :style="{
            width: width,
            height: height
          }" v-if="!playStatus"></image>
          <image src="/static/sound-recording/pause.png" mode="aspectFit" :style="{
            width: width,
            height: height
          }" v-else></image>
        </view>
      </u-circle-progress>
    </view>
    <text class="now-date">{{ reDate }}</text>
    <view @click="reset">重新录制</view>
  </view>
</template>

<script>
  import uCircleProgress from '../u-circle-progress/u-circle-progress.vue'
  const recorderManager = uni.getRecorderManager();
  const innerAudioContext = uni.createInnerAudioContext();
	export default {
    components: {
      uCircleProgress
    },
    props: {
      width: {
        type: String,
        default: '60rpx'
      },
      height: {
        type: String,
        default: '60rpx'
      },
      showTop: {
        type: Boolean,
        default: true
      },
      maximum: {
        type: [Number, String],
        default: 15
      },
      duration: {
        type: Number,
        default: 20
      },
      theme: {
        type: String,
        default: '#32b99d'
      },
      confirmText: {
        type: String,
        default: '完成'
      }
    },
		data() {
			return {
        reDate: '00:00',
        sec: 0,
        min: 0,
        finish: false,
        voicePath: '',
        playProgress: 100,
        playTimer: null,
        timer: null,
        playStatus: false,
        isRecording: false // 录音状态变量
			};
		},
    created () {
      this.onMonitorEvents()
    },
    computed: {
      calcProgress () {
        return (this.sec + (this.min * 60)) / this.maximum * 100
      }
    },
    methods: {
      confirm () {
        if (!innerAudioContext.paused) {
          innerAudioContext.stop()
        }
        this.$emit('confirm', this.voicePath)
      },
      cancel () {
        if (!innerAudioContext.paused) {
          innerAudioContext.stop()
        }
        this.$emit('cancel')
      },
      handle () {
        if (!this.isRecording) { // 如果不在录音状态，开始录音
          this.onStartRecoder();
          this.isRecording = true;
        } else { // 如果在录音状态，停止录音
          this.onEndRecoder();
          this.isRecording = false;
        }
      },
      reset () {
        this.voicePath = ''
        this.min = 0
        this.sec = 0
        this.reDate = '00:00'
        this.playProgress = 100
        this.finish = false
        this.$emit('reset')
      },
      playVoice() {
        innerAudioContext.src = this.voicePath;
        
        if (innerAudioContext.paused) {
          innerAudioContext.play()
          this.playStatus = true
        } else {
          innerAudioContext.stop();
        }
        this.$emit('playVoice', innerAudioContext.paused)
      },
      onEndRecoder () {
        recorderManager.stop()
      },
      onStartRecoder () {
        recorderManager.start({
          duration: this.maximum * 1000
        })
      },
      onMonitorEvents () {
        recorderManager.onStart(() => {
          uni.showLoading({
            title: '录制中...'
          })
          this.startDate()
          this.$emit('start')
        })
        recorderManager.onStop(({ tempFilePath }) => {
          this.voicePath = tempFilePath
          clearInterval(this.timer)
          uni.hideLoading()
          this.finish = true
          this.$emit('end')
        })
        innerAudioContext.onTimeUpdate(() => {
          let totalDate = innerAudioContext.duration
          let nowTime = innerAudioContext.currentTime
          let surplus = totalDate - nowTime
          this.playProgress = surplus / totalDate * 100
          
          let _min = Math.floor(surplus / 60)
          if (_min < 10) _min = '0' + _min;
          let _sec = Math.floor(surplus%60)
          if (_sec < 10) _sec = '0' + _sec;
          this.reDate = _min + ':' + _sec
        })
        innerAudioContext.onPause(() => {
          this.resetDate()
          this.playProgress = 100
          this.playStatus = false
          console.log('播放暂停')
        })
        innerAudioContext.onStop(() => {
          this.resetDate()
          this.playProgress = 100
          this.playStatus = false
          console.log('播放停止')
          this.$emit('stop')
        })
      },
      startDate () {
        clearInterval(this.timer)
        this.sec = 0
        this.min = 0
        this.timer = setInterval(() => {
          this.sec += this.duration / 1000
          if (this.sec >= 60) {
            this.min ++
            this.sec = 0
          }
          this.resetDate()
        }, this.duration)
      },
      resetDate () {
        let _s = this.sec < 10 ? '0' + parseInt(this.sec) : parseInt(this.sec)
        let _m = this.min < 10 ? '0' + this.min : this.min
        this.reDate = _m + ':' + _s
      }
    }
	}
</script>

<style lang="scss">
.recorder {
  position: relative;
  display: flex;
  align-items: center;
  flex-direction: column;
  background-color: #fff;
  font-size: 24rpx;
  width: 100%;
  .re-top {
    display: flex;
    justify-content: space-between;
    padding: 10rpx 20rpx;
    width: 100%;
    font-size: 28rpx;
    box-sizing: border-box;
  }
  .title {
    font-size: 36rpx;
    color: #333;
    padding: 20rpx 0 30rpx;
  }
  .recorder-box {
    position: relative;
  }
  .now-date {
    font-size: 28rpx;
    color: #666;
    padding: 20rpx 0;
  }
}
</style>
