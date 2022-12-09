<template>
  <div>
    <div class="player-app"></div>
  </div>
</template>

<script>
import OPlayer from '@oplayer/core'
import ui from '@oplayer/ui'
import hls from '@oplayer/hls'

export default {
  name: 'PlayerTv',
  props: ['url'],
  data() {
    return {
      player: null
    }
  },
  methods: {
    init() {
      this.player = OPlayer.make('.player-app', {
        source: {
          src: this.url
        },
        autoplay: true,
        isLive: true
      })
        .use([
          ui({
            pictureInPicture: true,
            fullscreen: true
          }),
          hls()
        ])
        .create()
    }
  },
  beforeDestroy() {
    if (this.player) {
      this.player.destroy()
    }
  },
  watch: {
    url: function () {
      this.$nextTick(() => {
        this.init()
      })
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
