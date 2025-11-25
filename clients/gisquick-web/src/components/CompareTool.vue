<template>
  <div class="f-row-ac">
    <v-btn class="icon p-1" @click="switchSides">
      <v-icon name="swap"/>
      <v-tooltip slot="tooltip">
        <translate>Switch sides</translate>
      </v-tooltip>
    </v-btn>
    <portal v-if="layer" to="map-overlay">
      <div
        class="swipe-handler"
        :style="handlerStyle"
        @mousedown="resizeHandler"
      >
        <div class="line"/>
      </div>
    </portal>
  </div>
</template>

<script>
import { mapState, mapGetters } from 'vuex'

import clamp from 'lodash/clamp'
import { getRenderPixel } from 'ol/render.js'
import { unByKey } from 'ol/Observable'

import { eventCoord, DragHandler } from '@/events'

export default {
  props: {
    layer: String
  },
  data () {
    return {
      position: 50,
      flip: false
    }
  },
  computed: {
    ...mapState(['project']),
    ...mapGetters(['visibleBaseLayers']),
    handlerStyle () {
      return {
        left: this.position + '%'
      }
    },
    tr () {
      return {
        Layer: this.$gettext('Layer'),
      }
    }
  },
  beforeDestroy () {
    this.removeLayer()
    this.$map.render()
  },
  watch: {
    layer: {
      immediate: true,
      handler (layer) {
        if (layer) {
          this.setupLayer(layer)
        } else {
          this.removeLayer()
        }
      }
    }
  },
  methods: {
    switchSides () {
      this.flip = !this.flip
      this.$map.render()
    },
    removeLayer () {
      if (this._ol) {
        unByKey(this._ol.listeners)
        this._ol = null
      }
    },
    getClipArea (evt) {
      const mapSize = this.$map.getSize()
      const x = mapSize[0] * (this.position / 100)
      if (this.flip) {
        const tl = getRenderPixel(evt, [0, 0])
        const tr = getRenderPixel(evt, [x, 0])
        const bl = getRenderPixel(evt, [0, mapSize[1]])
        const br = getRenderPixel(evt, [x, mapSize[1]])
        return [ tl, tr, bl, br ]
      } else {
        const tl = getRenderPixel(evt, [x, 0])
        const tr = getRenderPixel(evt, [mapSize[0], 0])
        const bl = getRenderPixel(evt, [x, mapSize[1]])
        const br = getRenderPixel(evt, mapSize)
        return [ tl, tr, bl, br ]
      }
    },
    async setupLayer (layer) {
      const map = this.$map
      this.removeLayer()

      const ol = map.getLayers().getArray().find(l => l.get('name') === layer)
      const key1 = ol.on('prerender', evt => {
        const ctx = evt.context
        const [ tl, tr, bl, br ] = this.getClipArea(evt)

        ctx.save()
        ctx.beginPath()
        ctx.moveTo(tl[0], tl[1])
        ctx.lineTo(bl[0], bl[1])
        ctx.lineTo(br[0], br[1])
        ctx.lineTo(tr[0], tr[1])
        ctx.closePath()
        ctx.clip()
      })
      const key2 = ol.on('postrender', evt => evt.context.restore())
      // map.addLayer(ol)
      map.render()

      this._ol = {
        layer: ol,
        listeners: [key1, key2]
      }
    },
    resizeHandler (e) {
      const width = this.$map.getSize()[0]
      DragHandler(e, {
        onStart: () => {
          this.resizing = true
        },
        onMove: e => {
          const x = eventCoord(e)[0]
          const pos = 100 * (x / width)
          this.position = clamp(0, pos, 100)
          this.$map.render()
        },
        onEnd: () => {
          this.resizing = false
        }
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.swipe-handler {
  position: fixed;
  pointer-events: auto;
  width: 6px;
  cursor: ew-resize;
  top: 0;
  bottom: 0;
  user-select: none;
  display: flex;
  flex-direction: column;
  margin-left: -2px;
  .line {
    background-color: rgba(#222, 0.75);
    // border-right: 1px solid #eee;
    height: 100%;
    width: 2px;
    top: 0;
    left: 0;
    margin-left: 2px;
  }
}
</style>