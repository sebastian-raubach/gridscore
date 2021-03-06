<template>
  <h4 id="user-position" class="position-absolute no-user-select" :style="{ left: userLeft, top: userTop, transform: userRotate, opacity: 0.75, margin: 0, padding: 0 }" v-show="userLeft && userTop">
    <template v-if="position !== null && position.heading !== undefined && position.heading !== null"><BIconCursorFill /></template>
    <template v-else><BIconBullseye /></template>
  </h4>
</template>

<script>
import { mapGetters } from 'vuex'
import { BIconCursorFill, BIconBullseye } from 'bootstrap-vue'

export default {
  components: {
    BIconCursorFill,
    BIconBullseye
  },
  data: function () {
    return {
      userLeft: null,
      userTop: null
    }
  },
  props: {
    tableId: {
      type: String,
      default: ''
    },
    /** The current user position in the field, x and y are scaled to 0-100 */
    position: {
      type: Object,
      default: () => null
    }
  },
  watch: {
    position: function () {
      this.updateUserPosition()
    }
  },
  computed: {
    /** Mapgetters exposing the store configuration */
    ...mapGetters([
      'storeCornerPoints',
      'storeUseGps'
    ]),
    userRotate: function () {
      if (this.position && this.position !== null && this.position.heading !== null && this.position.heading >= 0 && this.position.heading <= 360) {
        // Calculate the angles between a vertical line and the right hand edge of the field layout
        const cp = this.storeCornerPoints
        const angleOne = Math.atan2(100, 0)
        const angleTwo = Math.atan2(cp[2][0] - cp[1][0], cp[2][1] - cp[1][1])
        // Convert it from radians to degrees
        const radians = (Math.abs(angleOne) - Math.abs(angleTwo)) * (180 / Math.PI)
        // Substract that angle from the heading minus 45 because the icon we're using is rotated
        const heading = this.position.heading - radians - 45

        return `rotate(${heading}deg)`
      } else {
        return null
      }
    }
  },
  methods: {
    /**
     * Updates the location of the user position indicator.
     */
    updateUserPosition: function () {
      if (this.storeUseGps && this.position && this.position.x >= 0 && this.position.x <= 100 && this.position.y >= 0 && this.position.y <= 100) {
        if (!document.querySelector(`#${this.tableId} .table`)) {
          return null
        }
        // Get the widths of the table, the top left cell and the bottom right cell
        const tableWidth = document.querySelector(`#${this.tableId} .table`).offsetWidth
        const firstColWidth = document.querySelector(`#${this.tableId} .table thead tr th:first-child`).offsetWidth
        const lastColWidth = document.querySelector(`#${this.tableId} .table tfoot tr th:last-child`).offsetWidth
        const iconWidth = document.querySelector(`#${this.tableId} #user-position`).offsetWidth

        // Calculate the percentage position within just the body of the table (without first and last column), then add the first column width and subtract half the height of the icon to center it
        this.userLeft = `${firstColWidth + (tableWidth - firstColWidth - lastColWidth) * this.position.x / 100 - iconWidth / 2}px`

        // Get the heights of the table, the top left cell and the bottom right cell
        const tableHeight = document.querySelector(`#${this.tableId} .table`).offsetHeight
        const headerHeight = document.querySelector(`#${this.tableId} .table thead tr th:first-child`).offsetHeight
        const footerHeight = document.querySelector(`#${this.tableId} .table tfoot tr th:last-child`).offsetHeight
        const iconHeight = document.querySelector(`#${this.tableId} #user-position`).offsetHeight

        // Calculate the percentage position within just the body of the table (without first and last row), then add the first row height and substract half the width of the icon to center it
        this.userTop = `${headerHeight + (tableHeight - headerHeight - footerHeight) * (100 - this.position.y) / 100 - iconHeight / 2}px`
      } else {
        this.userLeft = null
        this.userTop = null
      }
    }
  },
  mounted: function () {
    window.addEventListener('resize', this.updateUserPosition)

    this.$nextTick(() => this.updateUserPosition())
  },
  beforeDestroy: function () {
    window.removeEventListener('resize', this.updateUserPosition)
  }
}
</script>

<style>
#user-position {
  z-index: 1;
  transition: top .1s ease-in-out, left .1s ease-in-out;
}
</style>
