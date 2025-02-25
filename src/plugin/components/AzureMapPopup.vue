<template>
  <component
    :is="tag"
  >
    <slot/>
  </component>
</template>

<script lang="ts">
import { getMapInjection } from '@/plugin/utils/dependency-injection'
import getOptionsFromProps from '@/plugin/utils/get-options-from-props'
import { atlas } from 'types'
import Vue, { PropType } from 'vue'

enum AzureMapPopupEvent {
  Created = 'created',
  Update = 'update',
  Open = 'open',
  Close = 'close',
}

/**
 * An information window anchored at a specified position on a map.
 */
export default Vue.extend({
  name: 'AzureMapPopup',

  /**
   * Inject the `getMap` function to get the `atlas.Map` instance
   */
  inject: ['getMap'],

  model: {
    prop: 'open',
    event: AzureMapPopupEvent.Update,
  },

  props: {
    /**
     * Specifies the tag used for the popup content
     * default `div`
     * @default "div"
     */
    tag: {
      type: String,
      default: 'div',
    },

    /**
     * Opens or closes the popup,
     * compatible with `v-model` directive
     */
    open: {
      type: Boolean,
      default: false,
    },

    /**
     * Specifies if the close button should be displayed in the popup or not.
     * default `true`
     * @default true
     */
    closeButton: {
      type: Boolean as PropType<boolean | null>,
      default: null,
    },

    /**
     * Specifies the fill color of the popup.
     * default `"#FFFFFF"`
     * @default "#FFFFFF"
     */
    fillColor: {
      type: String as PropType<string | null>,
      default: null,
    },

    /**
     * An array of [pixelsRight, pixelsDown] for how many pixels to the right and down the anchor of the popup should be
     * offset. Negative numbers can be used to offset the popup left and up.
     * default `[0, 0]`
     * @default [0, 0]
     */
    pixelOffset: {
      type: Array as PropType<atlas.Pixel | null>,
      default: null,
    },

    /**
     * The position on the map where the popup should be anchored.
     * default `[0, 0]`
     * @default [0, 0]
     */
    position: {
      type: Array as PropType<atlas.data.Position | null>,
      default: null,
    },

    /**
     * Specifies if the pointer should be displayed in the popup or not.
     * default `true`
     * @default true
     */
    showPointer: {
      type: Boolean as PropType<boolean | null>,
      default: null,
    },
  },

  computed: {
    popupOptionsProps(): Record<string, any> {
      let { closeButton, fillColor, pixelOffset, position, showPointer } = this

      return {
        closeButton,
        fillColor,
        pixelOffset,
        position,
        showPointer,
      }
    },
  },

  created() {
    // Look for the injected function that retreives the map instance
    const getMap = getMapInjection(this)

    if (!getMap) return

    // Retrieve the map instance from the injected function
    const map = getMap()

    // If the content option is passed as an attribute,
    // warn the user that they should use slots instead of a raw html string
    if (this.$attrs.content && process.env.NODE_ENV !== 'production') {
      return console.warn(
        `Invalid <AzureMapPopup> content prop.\nThe content option is not supported, please use the default slot instead to prevent XSS attacks.`
      )
    }

    // Create a popup with selected component props as options
    const popup = new this.$_azureMaps.atlas.Popup(
      getOptionsFromProps({ props: this.popupOptionsProps })
    )

    this.$emit(AzureMapPopupEvent.Created, popup)

    // Set the popup content when the component is mounted
    this.$on('hook:mounted', () => {
      popup.setOptions({
        content: this.$el as HTMLElement,
      })

      // Emit update event when popup opens
      map.events.add(AzureMapPopupEvent.Open, popup, this.openEventHandler)

      // Emit update event when popup closes
      map.events.add(AzureMapPopupEvent.Close, popup, this.closeEventHandler)

      // Watch the open prop to show and hide the popup
      this.$watch(
        'open',
        (newVal: boolean) => {
          if (newVal) {
            popup.open(map)
          } else if (popup.isOpen()) {
            popup.close()
          }
        },
        {
          immediate: true,
        }
      )
    })

    // Watch for all options props changes
    this.$watch(
      () => {
        let values = ''
        for (const value of Object.values(this.popupOptionsProps)) {
          values += value
        }
        return values
      },
      () => {
        popup.setOptions(
          getOptionsFromProps({
            props: this.popupOptionsProps,
          })
        )
      }
    )

    // When the component is destroyed
    this.$once('hook:destroyed', () => {
      // Close and remove the popup
      popup.remove()

      // Remove the popup events attached to the map
      map.events.remove(AzureMapPopupEvent.Open, popup, this.openEventHandler)
      map.events.remove(AzureMapPopupEvent.Close, popup, this.closeEventHandler)
    })
  },

  methods: {
    openEventHandler(targetedEvent: atlas.TargetedEvent): void {
      this.$emit(AzureMapPopupEvent.Update, true)
      this.$emit(AzureMapPopupEvent.Open, targetedEvent)
    },

    closeEventHandler(targetedEvent: atlas.TargetedEvent): void {
      this.$emit(AzureMapPopupEvent.Update, false)
      this.$emit(AzureMapPopupEvent.Close, targetedEvent)
    },
  },
})
</script>
