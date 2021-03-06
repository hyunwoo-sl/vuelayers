<script>
  import { createTileUrlFunctionFromTemplates } from 'ol-tilecache'
  import { VectorTile as VectorTileSource } from 'ol/source'
  import { createXYZ } from 'ol/tilegrid'
  import { urlTileSource } from '../../mixins'
  import { createMvtFmt, roundExtent, extentFromProjection } from '../../ol-ext'
  import { isArray, isFunction, isNumber, sealFactory, makeWatchers, isEqual, sequential } from '../../utils'

  export default {
    name: 'VlSourceVectorTile',
    mixins: [
      urlTileSource,
    ],
    props: {
      // ol/source/Tile
      cacheSize: {
        type: Number,
        default: 128,
      },
      zDirection: {
        type: Number,
        default: 1,
      },
      // ol/source/VectorTile
      extent: {
        type: Array,
        validator: value => value.length === 4 && value.every(isNumber),
      },
      formatFactory: {
        type: Function,
        default: createMvtFmt,
      },
      overlaps: {
        type: Boolean,
        default: true,
      },
      tileClass: Function,
      maxZoom: {
        type: Number,
        default: 22,
      },
      minZoom: {
        type: Number,
        default: 0,
        validator: value => value >= 0,
      },
      maxResolution: Number,
      tileSize: {
        type: [Number, Array],
        default: () => [512, 512],
        validator: value => isNumber(value) ||
          (isArray(value) && value.length === 2 && value.every(isNumber)),
      },
    },
    data () {
      return {
        format: null,
      }
    },
    computed: {
      tileSizeArr () {
        return isArray(this.tileSize) ? this.tileSize : [this.tileSize, this.tileSize]
      },
      derivedTileGridFactory () {
        if (isFunction(this.tileGridFactory)) {
          return this.tileGridFactory
        }

        const extent = this.extentDataProj || extentFromProjection(this.projection)
        const maxZoom = this.maxZoom
        const minZoom = this.minZoom
        const maxResolution = this.maxResolution
        const tileSize = this.tileSizeArr

        return () => createXYZ({ extent, maxZoom, minZoom, maxResolution, tileSize })
      },
      extentDataProj () {
        return roundExtent(this.extent)
      },
      extentViewProj () {
        return this.extentToViewProj(this.extent)
      },
      formatIdent () {
        if (!this.olObjIdent) return

        return this.makeIdent(this.olObjIdent, 'format')
      },
      sealFormatFactory () {
        return sealFactory(::this.formatFactory)
      },
      resolvedTileUrlFunc () {
        if (isFunction(this.tileUrlFunc)) {
          return this.tileUrlFunc
        }
        if (this.expandedUrls.length === 0) return

        return createTileUrlFunctionFromTemplates(this.expandedUrls, this.tileGrid)
      },
    },
    watch: {
      formatIdent: /*#__PURE__*/sequential(function (value, prevValue) {
        if (value && prevValue) {
          this.moveInstance(value, prevValue)
        } else if (value && !prevValue && this.format) {
          this.setInstance(value, this.format)
        } else if (!value && prevValue) {
          this.unsetInstance(prevValue)
        }
      }),
      sealFormatFactory: /*#__PURE__*/sequential(async function (value) {
        while (this.hasInstance(this.formatIdent)) {
          this.unsetInstance(this.formatIdent)
        }

        if (isFunction(value)) {
          this.format = this.instanceFactoryCall(this.formatIdent, this::value)
        } else {
          this.format = undefined
        }

        if (process.env.VUELAYERS_DEBUG) {
          this.$logger.log('sealDataFormatFactory changed, scheduling recreate...')
        }

        await this.scheduleRecreate()
      }),
      overlaps: /*#__PURE__*/sequential(async function (value) {
        if (value === await this.getOverlaps()) return

        if (process.env.VUELAYERS_DEBUG) {
          this.$logger.log('overlaps changed, scheduling recreate...')
        }

        await this.scheduleRecreate()
      }),
      .../*#__PURE__*/makeWatchers([
        'extentViewProj',
        'tileClass',
      ], prop => /*#__PURE__*/sequential(async function (val, prev) {
        if (isEqual(val, prev)) return

        if (process.env.VUELAYERS_DEBUG) {
          this.$logger.log(`${prop} changed, scheduling recreate...`)
        }

        await this.scheduleRecreate()
      })),
    },
    created () {
      if (isFunction(this.sealFormatFactory)) {
        this.format = this.instanceFactoryCall(this.formatIdent, ::this.sealFormatFactory)
        // this.$watch('format', async () => {
        //   if (process.env.VUELAYERS_DEBUG) {
        //     this.$logger.log('format changed, scheduling recreate...')
        //   }
        //
        //   await this.scheduleRecreate()
        // })
      }
    },
    methods: {
      /**
       * @return {VectorTileSource}
       */
      createSource () {
        return new VectorTileSource({
          // ol/source/Source
          attributions: this.currentAttributions,
          attributionsCollapsible: this.attributionsCollapsible,
          projection: this.resolvedDataProjection,
          state: this.currentState,
          wrapX: this.wrapX,
          // ol/source/Tile
          cacheSize: this.cacheSize,
          tileGrid: this.tileGrid,
          transition: this.transition,
          zDirection: this.zDirection,
          // ol/source/UrlTile
          tileLoadFunction: this.resolvedTileLoadFunc,
          tileUrlFunction: this.resolvedTileUrlFunc,
          // ol/source/VectorTile
          format: this.format,
          extent: this.extentViewProj,
          overlaps: this.overlaps,
          tileClass: this.tileClass,
        })
      },
      async getFeaturesInExtent (extent, viewProj = false) {
        if (!viewProj) {
          extent = this.extentToViewProj(extent)
        }

        return (await this.resolveSource()).getFeaturesInExtent(extent)
      },
      async getOverlaps () {
        return (await this.resolveSource()).getOverlaps()
      },
      async clear () {
        (await this.resolveSource()).clear()
      },
    },
  }
</script>
