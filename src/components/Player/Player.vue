<template>
  <div id="player" class="player-app"></div>
</template>

<script>
import Global from '@/components/Global'
import { getRealUrl } from '@/api/liveList'
import Danmaku from 'danmaku'
import OPlayer from '@oplayer/core'
import ui from '@oplayer/ui'
import hls from '@oplayer/hls'
import mpegts from '@oplayer/mpegts'
import danmaku from '@oplayer/danmaku'

export default {
  name: 'Player',
  props: [
    'platform',
    'roomId',
    'isLive',
    'banActive',
    'banLevel',
    'banContentList',
    'checkedContentList',
    'danmuStyle',
    'danmuSpeed',
    'danmuArea',
    'danmuNum'
  ],
  data() {
    return {
      player: null,
      quality: [],
      ws: null,
      huyaAyyuid: '',
      eGameToken: '',
      danmaku: null,
      danmuShow: true,
      danmuNumStep: 0
    }
  },
  methods: {
    init() {
      let _this = this
      if (this.isLive) {
        getRealUrl(this.platform, this.roomId).then((response) => {
          let data = response.data.data
          let qualityTemp = []
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('token')) {
            this.eGameToken = data.token
          }
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('ayyuid')) {
            this.huyaAyyuid = data.ayyuid
          }

          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('FD')) {
            qualityTemp.push({ name: '流畅', url: data.FD })
          }
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('LD')) {
            qualityTemp.push({ name: '清晰', url: data.LD })
          }
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('SD')) {
            qualityTemp.push({ name: '高清', url: data.SD })
          }
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('HD')) {
            qualityTemp.push({ name: '超清', url: data.HD })
          }
          // eslint-disable-next-line no-prototype-builtins
          if (data.hasOwnProperty('OD')) {
            qualityTemp.push({ name: '原画', url: data.OD })
          }

          qualityTemp[qualityTemp.length - 1].default = true
          this.quality = qualityTemp

          const player = OPlayer.make('.player-app', {
            source: {
              src: this.quality[this.quality.length - 1].url
            },
            autoplay: true,
            isLive: true
          })
            .use([
              ui({
                pictureInPicture: true,
                fullscreen: true,
                menu: [
                  {
                    name: `画质`,
                    children: this.quality,
                    onChange({ url }) {
                      player.changeSource({ src: url })
                    }
                  }
                ]
              }),
              hls(),
              mpegts(),
              danmaku({ enable: true })
            ])
            .create()

          this.danmaku = player.plugins.danmaku
          this.danmaku.speed = ((this.danmuSpeed + 25) / 100) * 200

          this.player = player
          if (this.platform == 'bilibili') {
            this.initBilibiliWs()
          } else if (this.platform == 'douyu') {
            this.initDouyuWs()
          } else if (this.platform == 'huya') {
            this.initHuyaWs()
          } else if (this.platform == 'egame') {
            this.initEgameWs()
          } else {
            _this.$emit('notSupport')
          }
        })
      }
    },
    initBilibiliWs() {
      const ws = new WebSocket('wss://broadcastlv.chat.bilibili.com:2245/sub')
      this.ws = ws
      let _this = this
      ws.onopen = function () {
        let sendInfo = Global.encode(
          JSON.stringify({
            roomid: Number(_this.roomId)
          }),
          7
        )
        ws.send(sendInfo)
      }
      this.interval = setInterval(function () {
        ws.send(Global.encode('', 2))
      }, 30000)
      ws.onmessage = async function (msgEvent) {
        const packet = await Global.decode(msgEvent.data)
        switch (packet.op) {
          case 8:
            console.log('获取直播间弹幕成功')
            break
          case 5:
            packet.body.forEach((body) => {
              switch (body.cmd) {
                case 'DANMU_MSG':
                  if (_this.isBanned(body.info[4][0], `${body.info[1]}`)) {
                    _this.emitDanmu(`${body.info[1]}`, `${body.info[2][1]}`)
                  }
                  break
              }
            })
            break
        }
      }
    },
    initDouyuWs() {
      const ws = new WebSocket('wss://danmuproxy.douyu.com:8503/')
      let roomId = this.roomId
      let _this = this
      this.ws = ws
      ws.onopen = function () {
        let loginMsg = Global.douyuEncode('type@=loginreq/roomid@=' + roomId + '/')
        let joinGroupMsg = Global.douyuEncode('type@=joingroup/rid@=' + roomId + '/gid@=1/')
        let heartBeatMsg = Global.douyuEncode('type@=mrkl/')
        ws.send(heartBeatMsg)
        ws.send(loginMsg)
        ws.send(joinGroupMsg)
      }
      this.interval = setInterval(function () {
        let heartBeatMsg = Global.douyuEncode('type@=mrkl/')
        ws.send(heartBeatMsg)
      }, 45000)
      ws.onmessage = async function (msg) {
        const packet = await Global.douyuDecode(msg.data)
        switch (packet.body.type) {
          case 'loginres':
            console.log('获取直播间弹幕成功')
            break
          case 'chatmsg':
            if (_this.isBanned(packet.body.level, packet.body.txt)) {
              _this.emitDanmu(packet.body.txt, packet.body.nn)
            }
            break
        }
      }
    },
    initHuyaWs() {
      const ws = new WebSocket('wss://cdnws.api.huya.com/')
      let roomId = this.roomId
      let _this = this
      this.ws = ws
      ws.onopen = function () {
        let inRoomMsg = Global._bind_ws_info(_this.huyaAyyuid)
        let loginMsg = Global.huyaSendPingReq()
        ws.send(inRoomMsg)
        ws.send(loginMsg)
      }
      this.interval = setInterval(function () {
        let heartBeatMsg = Global.huyaSendPingReq()
        ws.send(heartBeatMsg)
      }, 30 * 1000)

      ws.onmessage = async function (msg) {
        var reader = new FileReader()
        reader.readAsArrayBuffer(msg.data)
        reader.onload = function () {
          let msg_obj = Global._on_mes(this.result)
          if (msg_obj.type == 'chat') {
            if (_this.isBanned('999', msg_obj.content)) {
              _this.emitDanmu(msg_obj.content, msg_obj.name)
            }
          }
        }
      }
    },
    initEgameWs() {
      const ws = new WebSocket('wss://barragepush.egame.qq.com/sub')
      this.ws = ws
      let _this = this
      ws.onopen = function () {
        let sendInfo = Global.eGameEncode(_this.eGameToken)
        ws.send(sendInfo)
      }
      this.interval = setInterval(function () {
        ws.send(Global.eGameHeart())
      }, 30000)
      ws.onmessage = async function (msgEvent) {
        const packet = await Global.eGameDecode(msgEvent.data)
        let data = packet.body[0].bin_data
        for (let i = 0; i < data.length; i++) {
          let msgData = data[i]
          let type = msgData.type
          if (type == 0 || type == 3 || type == 9) {
            if (_this.isBanned(msgData.ext.lvnew, msgData.content)) {
              _this.emitDanmu(msgData.content, msgData.nick)
            }
          }
        }
      }
    },

    testDanmuBan(text) {
      for (let i = 0; i < this.checkedContentList.length; i++) {
        let banContent = this.checkedContentList[i]
        let reg = new RegExp(banContent)
        if (reg.test(text)) {
          return true
        }
      }
      return false
    },
    isReg(reg) {
      let isReg
      try {
        isReg = eval(reg) instanceof RegExp
      } catch (e) {
        isReg = false
      }
      return isReg
    },
    isBanned(level, danmuContent) {
      let _this = this
      if (!_this.banActive || (Number(level) > Number(_this.banLevel) && !_this.testDanmuBan(danmuContent))) {
        return true
      }
      return false
    },
    emitDanmu(text, from) {
      let _this = this
      var someDanmakuAObj = {
        text: text, // Danmu text
        style: {
          fontSize: (this.danmuStyle.fontSize / 100) * 40 + 'px',
          color: this.danmuStyle.color,
          textShadow: this.danmuStyle.textShadow ? '-1px -1px #000, -1px 1px #000, 1px -1px #000, 1px 1px #000' : '',
          opacity: this.danmuStyle.opacity / 100,
          fontWeight: this.weightChange(this.danmuStyle.fontWeight)
        }
      }
      var newDanmu = {
        fromName: from,
        msg: text
      }
      _this.$emit('newDanmuSend', newDanmu)
      if (this.danmuNumStep > 0) {
        this.danmuNumStep--
        return
      } else {
        this.danmaku.emit(someDanmakuAObj)
        this.danmuNumStep = (100 - this.danmuNum) / 10
      }
    },
    weightChange(value) {
      switch (value) {
        case 0:
          return 'lighter'
        case 50:
          return 'normal'
        case 100:
          return 'bolder'
      }
    }
  },
  beforeDestroy() {
    if (this.player) {
      this.player.destroy()
    }
    clearInterval(this.interval)
    if (this.ws) {
      this.ws.close()
    }
    if (this.danmaku) {
      this.danmaku.destroy()
    }
  },
  watch: {
    platform: function () {
      this.$nextTick(() => {
        this.init()
      })
    },
    danmuSpeed: function () {
      if (this.danmaku) {
        let speed = ((this.danmuSpeed + 20) / 100) * 300
        this.danmaku.speed = speed
      }
    },
    danmuArea: function () {
      if (this.danmaku) {
        this.danmaku.clear()
        document.getElementsByClassName('danmuku')[0].style.height = this.danmuArea + '%'
        this.danmaku.resize()
      }
    },
    danmuNum: function () {
      this.danmuNumStep = (100 - this.danmuNum) / 10
    }
  }
}
</script>

<style scoped>
.player-app {
  width: 100%;
  height: 100%;
  /*pointer-events: none;*/
}
</style>
