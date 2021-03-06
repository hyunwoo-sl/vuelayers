<script>
  import { Fill as FillStyle } from 'ol/style'
  import { style } from '../../mixins'
  import { normalizeColor } from '../../ol-ext'
  import { isEqual, sequential } from '../../utils'

  export default {
    name: 'VlStyleFill',
    mixins: [
      style,
    ],
    props: {
      color: {
        type: [String, Array],
        default: () => [255, 255, 255, 0.4],
      },
    },
    computed: {
      parsedColor () {
        return normalizeColor(this.color)
      },
    },
    watch: {
      color: /*#__PURE__*/sequential(async function (value) {
        await this.setColor(value)
      }),
    },
    created () {
      this::defineServices()
    },
    methods: {
      /**
       * @return {FillStyle}
       * @protected
       */
      createStyle () {
        return new FillStyle({
          color: this.parsedColor,
        })
      },
      /**
       * @return {Promise<void>}
       * @protected
       */
      async mount () {
        if (this.$fillStyleContainer) {
          await this.$fillStyleContainer.setFill(this)
        }

        return this::style.methods.mount()
      },
      /**
       * @return {Promise<void>}
       * @protected
       */
      async unmount () {
        if (this.$fillStyleContainer && this.$fillStyleContainer.getFillVm() === this) {
          await this.$fillStyleContainer.setFill(null)
        }

        return this::style.methods.unmount()
      },
      /**
       * @return {Promise}
       */
      async refresh () {
        this::style.methods.refresh()

        if (this.$fillStyleContainer) {
          this.$fillStyleContainer.refresh()
        }
      },
      async getColor () {
        return normalizeColor((await this.resolveStyle()).getColor())
      },
      async setColor (color) {
        color = normalizeColor(color)
        if (isEqual(color, await this.getColor())) return

        (await this.resolveStyle()).setColor(color)
        await this.scheduleRefresh()
      },
    },
  }

  function defineServices () {
    Object.defineProperties(this, {
      $fillStyleContainer: {
        enumerable: true,
        get: () => this.$services?.fillStyleContainer,
      },
    })
  }
</script>
