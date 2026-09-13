<template>
  <div class="screen">
    <!-- 视频区：原生模块按 (0,0,800,192) 直写屏幕，这里只放透明点击层 -->
    <div class="videoarea" @click="togglePlay">
      <div class="errbox" v-if="errorText">
        <text class="errtext">{{ errorText }}</text>
      </div>
    </div>

    <!-- 控制条（视频区之外，避免被视频帧覆盖） -->
    <div class="controls">
      <div class="row1">
        <text class="time">{{ fmt.formatTime(positionMs) }}</text>
        <slider class="seekbar" :min="0" :max="seekMax" :step="1" v-model="seekVal"
          active-color="#4fd6c3" background-color="#263340"
          @moving="onSeekMoving" @change="onSeekChange"></slider>
        <text class="time right">{{ durText }}</text>
      </div>
      <div class="row2">
        <div class="btn press" @click="togglePlay"><text class="btntext">{{ playing ? '暂停' : '播放' }}</text></div>
        <div class="btn press" @click="seekBy(-30)"><text class="btntext">-30s</text></div>
        <div class="btn press" @click="seekBy(30)"><text class="btntext">+30s</text></div>
        <div class="btn accent press" @click="cycleRate"><text class="btntextacc">{{ rateLabel }}</text></div>
        <text class="qtag" v-if="qualityLabel">{{ qualityLabel }}</text>
        <div class="volbox">
          <text class="volicon">音</text>
          <slider class="volbar" :min="0" :max="100" :step="5" v-model="volume"
            active-color="#f5b85c" background-color="#263340" @change="onVolumeChange"></slider>
          <text class="voltext">{{ volume }}</text>
        </div>
        <div class="btn back press" @click="goBack"><text class="btntext">返回</text></div>
      </div>
    </div>

    <app-toast></app-toast>
  </div>
</template>

<script>
import bridge from '../../utils/player-bridge.js'
import quality from '../../utils/quality.js'
import history from '../../utils/history.js'
import device from '../../utils/device.js'
import * as fmt from '../../utils/fmt.js'
import appToast from '../../components/app-toast.vue'

export default {
  components: { 'app-toast': appToast },
  data() {
    return {
      path: '',
      name: '',
      positionMs: 0,
      durationMs: 0,
      seekVal: 0,
      seekMax: 1000,
      scrubbing: false,
      playing: false,
      ended: false,
      rate: 1,
      volume: 70,
      screen: { w: 800, h: 254 },
      qualityLabel: '',
      errorText: ''
    }
  },
  computed: {
    rateLabel() {
      const r = Number(this.rate) || 1
      return (r === Math.floor(r) ? r.toFixed(0) : String(r)) + 'x'
    },
    durText() {
      return this.durationMs > 0 ? fmt.formatTime(this.durationMs) : '--:--'
    }
  },
  methods: {
    toast(text, ms) {
      $falcon.trigger('bpv-toast', { text, ms })
    },
    goBack() {
      this.$page.finish()
    },
    // 播放入口：先按屏幕尺寸决定是否降画质，再交给原生模块
    async start(path, resumeMs) {
      const r = await bridge.play({
        path,
        screenW: this.screen.w,
        screenH: this.screen.h,
        maxW: this.screen.w,
        maxH: this.screen.h,
        rate: this.rate,
        volume: this.volume,
      })
      if (!r || !r.ok) {
        this.playing = false
        this.errorText = '无法播放\n' + ((r && r.error) || '未知错误')
        this.toast('无法播放：' + ((r && r.error) || '未知错误'), 3200)
        return
      }
      this.errorText = ''
      this.playing = true
      this.ended = false
      if (r.durationMs > 0) this.durationMs = r.durationMs
      const q = quality.decideQuality({
        videoW: r.videoW, videoH: r.videoH,
        screenW: this.screen.w, screenH: this.screen.h,
        name: this.name, mode: 'auto',
      })
      this.qualityLabel = q.label
      if (q.action === 'downscale') this.toast(q.label)
      if (resumeMs > 3000) {
        await bridge.seek(resumeMs)
        this.positionMs = resumeMs
      }
    },
    togglePlay() {
      if (this.ended) {
        this.ended = false
        bridge.seek(0)
        bridge.resume()
        this.playing = true
        return
      }
      if (this.playing) {
        bridge.pause()
        this.playing = false
      } else {
        bridge.resume()
        this.playing = true
      }
    },
    seekBy(sec) {
      const target = Math.max(0, this.positionMs + sec * 1000)
      this.seekVal = this.durationMs > 0 ? Math.round(target / this.durationMs * this.seekMax) : 0
      bridge.seek(target)
      this.positionMs = target
    },
    onSeekMoving(val) {
      this.scrubbing = true
      if (this.durationMs > 0) this.positionMs = Math.round(val / this.seekMax * this.durationMs)
    },
    onSeekChange(val) {
      this.scrubbing = false
      if (this.durationMs > 0) {
        const target = Math.round(val / this.seekMax * this.durationMs)
        bridge.seek(target)
        this.positionMs = target
      }
    },
    async cycleRate() {
      const list = quality.RATES
      let i = list.indexOf(Number(this.rate) || 1)
      if (i === -1) i = 2
      this.rate = list[(i + 1) % list.length]
      await bridge.setRate(this.rate)
      this.toast(this.rateLabel)
    },
    async onVolumeChange(val) {
      this.volume = val
      await bridge.setVolume(val)
    },
    // 500ms 轮询：进度 + 状态
    async tick() {
      if (!this.path) return
      const pos = await bridge.position()
      const st = await bridge.status()
      if (st && st.durationMs > 0) this.durationMs = st.durationMs
      if (!this.scrubbing && this.durationMs > 0) {
        this.seekVal = Math.min(this.seekMax, Math.round(pos / this.durationMs * this.seekMax))
      }
      this.positionMs = pos
      if (st && st.ok) {
        this.playing = st.playing
        if (st.eos) {
          this.ended = true
          this.playing = false
        }
      }
      this._tickCount = (this._tickCount || 0) + 1
      if (this._tickCount % 6 === 0) {
        history.saveProgress(this.path, pos, this.durationMs).catch(() => {})
      }
    }
  },
  async onLoad(options) {
    const o = options || {}
    this.path = String(o.path || '')
    this.name = String(o.name || '')
    this.rate = Number(o.rate || 1) || 1
    this.screen = await device.detectScreen()
    // 断点续播
    let resumeMs = 0
    try {
      const hist = await history.load()
      const it = hist.entries.find(e => e.path === this.path)
      if (it) {
        resumeMs = Number(it.positionMs || 0)
        if (it.volume != null) this.volume = it.volume
        if (it.rate) this.rate = it.rate
      }
    } catch (e) { /* 历史缺失不阻塞播放 */ }
    if (this.path) await this.start(this.path, resumeMs)
    this._pollTimer = this.setInterval(() => { this.tick().catch(() => {}) }, 500)
  },
  async onHide() {
    bridge.pause()
    this.playing = false
    if (this.path) await history.saveProgress(this.path, this.positionMs, this.durationMs).catch(() => {})
  },
  async onUnload() {
    bridge.stop()
    this.playing = false
    if (this._pollTimer) this.clearInterval(this._pollTimer)
    if (this.path) {
      try {
        await history.record({ path: this.path, name: this.name, positionMs: this.positionMs, durationMs: this.durationMs, rate: this.rate, volume: this.volume })
      } catch (e) { /* 忽略 */ }
    }
  }
}
</script>

<style lang="less" scoped>
@import "../../styles/common.less";
.screen {
  width: 100vw;
  height: 100vh;
  background-color: #000000;
}
.videoarea {
  position: absolute;
  left: 0vw;
  top: 0vh;
  width: 100vw;
  height: 75.59vh; /* 192px：控制条之外的区域，原生视频直写 */
}
.controls {
  position: absolute;
  left: 0vw;
  top: 75.59vh;
  width: 100vw;
  height: 24.41vh; /* 62px */
  background-color: #121922;
}
.row1 {
  display: flex;
  flex-direction: row;
  align-items: center;
  padding: 0 1.5vw;
  height: 12vh;
}
.time {
  width: 10vw;
  color: #8ca0ad;
  font-size: 4.2vh;
  text-align: center;
}
.seekbar {
  flex: 1;
  height: 8vh;
}
.row2 {
  display: flex;
  flex-direction: row;
  align-items: center;
  padding: 0 1.5vw;
  height: 12vh;
}
.btn {
  width: 11vw;
  height: 10.5vh;
  background-color: #19242f;
  border-radius: 1vw;
  margin-right: 1vw;
  display: flex;
  align-items: center;
  justify-content: center;
}
.accent {
  background-color: #123a37;
}
.back {
  margin-left: auto;
  margin-right: 0;
}
.btntext {
  color: #e8eef2;
  font-size: 4.4vh;
}
.qtag {
  color: #8ca0ad;
  font-size: 3.4vh;
  lines: 1;
  margin-right: 1vw;
}
.errbox {
  position: absolute;
  left: 4vw;
  top: 30vh;
  width: 60vw;
}
.errtext {
  color: #ff6b72;
  font-size: 4.4vh;
  line-height: 6.5vh;
}
.btntextacc {
  color: #4fd6c3;
  font-size: 4.4vh;
}
.volbox {
  display: flex;
  flex-direction: row;
  align-items: center;
  margin-right: 1vw;
}
.volicon {
  color: #f5b85c;
  font-size: 4.4vh;
  width: 4.5vw;
}
.volbar {
  width: 14vw;
  height: 8vh;
}
.voltext {
  color: #8ca0ad;
  font-size: 3.8vh;
  width: 5vw;
  text-align: right;
}
</style>
