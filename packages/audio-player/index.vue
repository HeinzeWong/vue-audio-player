<template>
  <section class="audio-conatiner">
    <div class="audio-btn-container">
      <img class="play-previous-btn"
           src="./icon-play-previous.png"
           @click="playPrev">
      <img class="play-start-btn"
           src="./icon-play-start.png"
           v-if="!isPlaying"
           @click="play">
      <img class="play-start-btn"
           src="./icon-play-pause.png"
           v-else
           @click="pause">
      <img class="play-next-btn"
           src="./icon-play-next.png"
           @click="playNext">
    </div>
    <div class="audio-progress-container"
         ref="audioProgressContainer">
      <div class="audio-progress"
           ref="audioProgress" />
      <img class="audio-progress-point"
           src="./icon-play-progress-point.png"
           ref="audioProgressPoint">
    </div>
    <div class="audio-time-container">
      <div class="audio-current-time">
        {{ currentTimeAfterFormat }}
      </div>
      <div class="audio-duration">
        {{ formatTime(duration) }}
      </div>
    </div>
    <audio ref="audio"
           :src="audioList[currentPlayIndex]"
           @ended="onEnded"
           @timeupdate="onTimeUpdate"
           @loadedmetadata="onLoadedmetadata">
      浏览器太老咯，请升级浏览器吧~
    </audio>
  </section>
</template>

<script>
export default {
  name: 'AudioPlayer',
  props: {
    // 音频播放列表
    audioList: {
      default: null,
      type: Array
    },
    // 播放前的回调函数
    beforePlay: {
      default: null,
      type: Function
    },
    // 上一首前的回调函数
    beforePrev: {
      default: null,
      type: Function
    },
    // 下一首前的回调函数
    beforeNext: {
      default: null,
      type: Function
    },
    // 是否列表循环播放
    isLoop: {
      default: true,
      type: Boolean
    }
  },
  data() {
    return {
      currentPlayIndex: 0, // 当前播放的音频位置索引
      isPlaying: false, // 音频是否正在播放
      duration: '', // 音频持续时间
      currentTime: '', // 音频当前播放时间
      currentTimeAfterFormat: '', // 音频播放当时时间（格式化后）
      isDragging: false // 是否正在拖拽音频进度
    }
  },
  methods: {
    // 当媒介元素的持续时间以及其它媒介已加载数据时运行脚本
    onLoadedmetadata() {
      this.duration = this.$refs.audio.duration
      this.initProgressDrag()
    },
    // 正在播放音频中
    onTimeUpdate(val) {
      this.progressValue = this.$refs.audio.currentTime / (this.$refs.audio.duration / 100)
      this.currentTimeAfterFormat = this.formatTime(this.$refs.audio.currentTime)

      // 正在拖拽进度
      if (this.isDragging) {
        return
      }

      // 设置播放进度条
      this.$refs.audioProgress.style.width =
        this.$refs.audio.currentTime / this.$refs.audio.duration * this.$refs.audioProgressContainer.offsetWidth + 'px'
      // 设置播放进度拖拽点
      this.$refs.audioProgressPoint.style.left =
        this.$refs.audio.currentTime /
        this.$refs.audio.duration *
        (this.$refs.audioProgressContainer.offsetWidth - this.$refs.audioProgressPoint.offsetWidth / 2) +
        'px'
    },
    // 格式化秒为分
    formatTime(second) {
      return [parseInt((second / 60) % 60), parseInt(second % 60)].join(':').replace(/\b(\d)\b/g, '0$1')
    },
    // 音频播放完毕
    onEnded() {
      this.pause()
      this.$emit('ended')
    },
    // 初始化音频进度的拖拽逻辑
    initProgressDrag() {
      this.$refs.audioProgressPoint.addEventListener(
        'touchstart',
        event => {
          // 设置拖拽中
          this.isDragging = true
        },
        false
      )
      this.$refs.audioProgressPoint.addEventListener(
        'touchmove',
        event => {
          let touch = event.touches[0]

          // 超出范围
          if (
            touch.pageX < this.$refs.audioProgressContainer.offsetLeft ||
            touch.pageX > this.$refs.audioProgressContainer.offsetLeft + this.$refs.audioProgressContainer.offsetWidth
          ) {
            return
          }
          // 设置点点
          this.$refs.audioProgressPoint.style.left =
            touch.pageX - this.$refs.audioProgressPoint.offsetWidth / 2 - this.$refs.audioProgressContainer.offsetLeft + 'px'
          // 设置进度条
          this.$refs.audioProgress.style.width = touch.pageX - this.$refs.audioProgressContainer.offsetLeft + 'px'
          // 设置当前时间
          this.currentTime =
            this.$refs.audioProgress.offsetWidth / this.$refs.audioProgressContainer.offsetWidth * this.duration
          // 设置当前时间（格式化后）
          this.currentTimeAfterFormat = this.formatTime(this.currentTime)
        },
        false
      )
      this.$refs.audioProgressPoint.addEventListener(
        'touchend',
        event => {
          // 设置播放位置
          this.$refs.audio.currentTime = this.currentTime
          // 设置未拖拽
          this.isDragging = false
        },
        false
      )

      this.initProgressClick()
    },
    // 初始化音频进度的点击逻辑
    initProgressClick() {
      this.$refs.audioProgressContainer.addEventListener('click', event => {
        let touch = event

        // 设置点点
        this.$refs.audioProgressPoint.style.left =
          touch.pageX - this.$refs.audioProgressPoint.offsetWidth / 2 - this.$refs.audioProgressContainer.offsetLeft + 'px'
        // 设置进度条
        this.$refs.audioProgress.style.width = touch.pageX - this.$refs.audioProgressContainer.offsetLeft + 'px'
        // 设置当前时间
        this.currentTime =
          this.$refs.audioProgress.offsetWidth / this.$refs.audioProgressContainer.offsetWidth * this.duration
        // 设置当前时间（格式化后）
        this.currentTimeAfterFormat = this.formatTime(this.currentTime)
        // 设置播放位置
        this.$refs.audio.currentTime = this.currentTime
      })
    },
    // 开始播放
    play() {
      if (this.beforePlay) {
        this.beforePlay((state) => {
          if (state !== false) {
            this.$refs.audio.play()
            this.isPlaying = true
          }
        })
        return
      }
      this.$refs.audio.play()
      this.isPlaying = true
    },
    // 暂停播放
    pause() {
      this.$refs.audio.pause()
      this.isPlaying = false
    },
    // 播放上一首
    playPrev() {
      if (this.currentPlayIndex <= 0 && !this.isLoop) {
        // 无上一首了
        alert('已是第一首！')
        return
      }

      let prev = () => {
        if (this.currentPlayIndex <= 0 && this.isLoop) {
          // 列表循环
          this.currentPlayIndex = this.audioList.length - 1
        } else {
          this.currentPlayIndex--
        }

        this.$nextTick(() => {
          this.$refs.audio.play()
          this.isPlaying = true
        })
      }

      if (this.beforePrev) {
        this.beforePrev((state) => {
          if (state !== false) {
            prev()
          }
        })
        return
      }
      prev()
    },
    // 播放下一首
    playNext() {
      if (this.currentPlayIndex + 1 >= this.audioList.length && !this.isLoop) {
        // 无下一首了
        alert('已是最后一首！')
        return
      }

      let next = () => {
        if (this.currentPlayIndex + 1 >= this.audioList.length && this.isLoop) {
          // 列表循环
          this.currentPlayIndex = 0
        } else {
          this.currentPlayIndex++
        }
        this.$nextTick(() => {
          this.$refs.audio.play()
          this.isPlaying = true
        })
      }

      if (this.beforeNext) {
        this.beforeNext((state) => {
          if (state !== false) {
            next()
          }
        })
        return
      }
      next()
    }
  }
}
</script>

<style lang="less">
section.audio-conatiner {
  margin: 0 15px;
  .audio-btn-container {
    display: flex;
    align-items: center;
    justify-content: center;
    .play-previous-btn {
      width: 21px;
    }
    .play-start-btn {
      width: 42px;
      margin: 0 40px;
    }
    .play-next-btn {
      width: 21px;
    }
  }
  .audio-progress-container {
    position: relative;
    background: #eee;
    height: 2px;
    margin-top: 22.5px;
    .audio-progress {
      position: absolute;
      left: 0;
      top: 0;
      bottom: 0;
      width: 0;
      background: #e35924;
    }
    .audio-progress-point {
      position: absolute;
      left: 0;
      top: 50%;
      width: 12px;
      margin-top: -6px;
    }
  }
  .audio-time-container {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    margin-top: 7px;
    .audio-current-time {
      font-size: 10px;
      color: #888;
    }
    .audio-duration {
      font-size: 10px;
      color: #888;
    }
  }
  audio {
    display: block;
    margin: 0 auto;
  }
}
</style>