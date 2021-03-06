<template>
  <b-modal :ok-title="$t('buttonOk')"
          :cancel-title="$t('buttonCancel')"
          :no-close-on-backdrop="isGuidedWalk"
          :no-close-on-esc="isGuidedWalk"
          @close="beforeClose"
          @ok.prevent="onSubmit"
          @hide="onHide"
          @shown="onShown"
          size="lg"
          no-fade
          ref="dataPointModal">
    <template v-slot:modal-header="{ close }">
      <h5 class="modal-title">{{ name }}</h5>

      <b-button :pressed.sync="isMarked">
        <template v-if="isMarked">
          <BIconBookmarkCheckFill /> {{ $t('buttonUnbookmarkCell') }}
        </template>
        <template v-else>
          <BIconBookmark /> {{ $t('buttonBookmarkCell') }}
        </template>
      </b-button>

      <button class="close ml-0" @click="close()">×</button>
    </template>
    <DataPointEntry :isMarked="isMarked" :row="localRow" :col="localCol" :key="`${localRow}-${localCol}`" ref="dataPointEntry" />

    <template v-if="!isGuidedWalk">
      <b-button @click="guidedWalkVisible = !guidedWalkVisible" class="my-3"><BIconSignpost /> {{ $t('buttonStartGuidedWalkToggle') }}</b-button>
      <b-collapse id="data-point-guide-collapse" class="mb-3" v-model="guidedWalkVisible">
        <p class="text-info">{{ $t('widgetGuideOrderText') }}</p>
        <GuideOrderSelector :row="localRow" :col="localCol" :visible="guidedWalkVisible" @order-selected="orderSelected"/>
        <b-button :disabled="!selectedOrder" @click="isGuidedWalk = true"><BIconPlay /> {{ $t('buttonStartGuidedWalk') }}</b-button>
      </b-collapse>
    </template>

    <template v-slot:modal-footer="{ ok, cancel }">
      <template v-if="isGuidedWalk">
        <b-button @click="prev" :disabled="walkingOrderIndex <= 0"><BIconChevronLeft /> {{ $t('buttonBack') }}</b-button>
        <b-button @click="finish" v-if="walkingOrderIndex >= walkingOrder.length - 1">{{ $t('buttonFinish') }} <BIconCheck2All /></b-button>
        <b-button @click="next" v-else>{{ $t('buttonNext') }} <BIconChevronRight /></b-button>
      </template>
      <template v-else>
        <b-button variant="secondary" @click="cancel">{{ $t('buttonCancel') }}</b-button>
        <b-button variant="primary" @click="ok">{{ $t('buttonOk') }}</b-button>
      </template>
    </template>
  </b-modal>
</template>

<script>
import DataPointEntry from '@/components/DataPointEntry'
import GuideOrderSelector from '@/components/GuideOrderSelector'
import { BIconBookmarkCheckFill, BIconBookmark, BIconCheck2All, BIconPlay, BIconSignpost, BIconChevronLeft, BIconChevronRight } from 'bootstrap-vue'
import types from '@/mixin/types'

/**
 * Shows a modal used to enter the data into GridScore. Each trait is shown and based on its type a different method for data input is show.
 */
export default {
  props: {
    /** The row of the current data point */
    row: {
      type: Number,
      default: null
    },
    /** The col of the current data point */
    col: {
      type: Number,
      default: null
    }
  },
  data: function () {
    return {
      guidedWalkVisible: false,
      isMarked: false,
      name: null,
      isGuidedWalk: true,
      walkingOrder: null,
      walkingOrderIndex: 0,
      localRow: null,
      localCol: null,
      selectedOrder: null,
      current: null
    }
  },
  components: {
    BIconBookmarkCheckFill,
    BIconSignpost,
    BIconPlay,
    BIconBookmark,
    BIconCheck2All,
    BIconChevronLeft,
    BIconChevronRight,
    DataPointEntry,
    GuideOrderSelector
  },
  mixins: [ types ],
  methods: {
    /**
     * Shows the modal and resets it to its initial state
     */
    show: function () {
      this.guidedWalkVisible = false
      this.isGuidedWalk = false
      this.walkingOrder = null
      this.walkingOrderIndex = 0
      this.localRow = null
      this.localCol = null
      this.$nextTick(() => this.$refs.dataPointModal.show())
    },
    orderSelected: function (order) {
      this.selectedOrder = order
      const orderObject = this.guideOrderTypes.filter(o => o.name === order)[0]
      this.walkingOrder = orderObject.cellSequence({ x: this.localCol, y: this.localRow, direction: orderObject.initialDirection })
      this.walkingOrderIndex = 0
    },
    prev: function () {
      this.$refs.dataPointEntry.verify(valid => {
        if (valid) {
          this.walkingOrderIndex = Math.min(this.walkingOrder.length - 1, this.walkingOrderIndex - 1)
          const current = this.walkingOrder[this.walkingOrderIndex]

          if (current.speak) {
            this.$refs.dataPointEntry.speak(this.$t(current.speak))
          }
          if (current.finished) {
            // TODO
          } else {
            this.localCol = current.x
            this.localRow = current.y
            this.update()
          }
        } else {
          // TODO
        }
      })
    },
    next: function () {
      this.$refs.dataPointEntry.verify(valid => {
        if (valid) {
          this.walkingOrderIndex = Math.max(0, this.walkingOrderIndex + 1)
          const current = this.walkingOrder[this.walkingOrderIndex]

          if (current.speak) {
            this.$refs.dataPointEntry.speak(this.$t(current.speak))
          }
          if (current.finished) {
            // TODO
          } else {
            this.localCol = current.x
            this.localRow = current.y
            this.update()
          }
        } else {
          // TODO
        }
      })
    },
    finish: function () {
      this.$refs.dataPointEntry.verify(valid => {
        if (valid) {
          this.hide()
        }
      })
    },
    update: function () {
      if (this.localRow !== null && this.localCol !== null) {
        const cell = this.$store.getters.storeData.get(`${this.localRow}-${this.localCol}`)
        this.isMarked = cell.isMarked || false
        this.name = cell.name
      } else {
        this.isMarked = false
        this.name = null
      }
    },
    /**
     * Hides the modal
     */
    hide: function () {
      this.$nextTick(() => this.$refs.dataPointModal.hide())
    },
    onFormVerified: function (valid) {
      if (valid) {
        this.hide()
      }
    },
    /**
     * Handle the submit event. Checks restrictions before accepting the data.
     */
    onSubmit: function () {
      this.$refs.dataPointEntry.verify(valid => {
        if (valid) {
          this.hide()
        }
      })
    },
    onHide: function () {
      this.$nextTick(() => this.$refs.dataPointEntry.disableSpeechRecognition())
    },
    onShown: function () {
      this.localRow = this.row
      this.localCol = this.col
      this.$nextTick(() => {
        this.update()
        this.$refs.dataPointEntry.setFocus()
      })
    },
    beforeClose: function (event) {
      if (this.isGuidedWalk) {
        // Prevent the default close event
        event.preventDefault()
        // Ask the user if they want to exit the guided walk mode
        this.$bvModal.msgBoxConfirm(this.$t('modalTextGuidedWalkExit'), {
          title: this.$t('modalTitleGuidedWalkExit'),
          okVariant: 'danger',
          okTitle: this.$t('buttonYes'),
          cancelTitle: this.$t('buttonNo')
        })
          .then(value => {
            if (value) {
              // If so, check the validity of the current form
              this.$refs.dataPointEntry.verify(valid => {
                if (valid) {
                  // If it's valid, hide
                  this.hide()
                }
              })
            }
          })
      }
    }
  }
}
</script>

<style>

</style>
