<template>
  <div>
    <p>{{ $t('modalTextDataEntry') }}</p>
    <b-form @submit.prevent="onSubmit" :validated="formValidated">
      <b-form-group v-for="(mapping, index) in visibleTraitMapping"
                    :key="`trait-${index}`"
                    :label-for="`trait-${index}`">
        <!-- Show the trait name along with the type and its color as the label -->
        <template v-slot:label>
          <span :style="{ color: storeTraitColors[mapping.index % storeTraitColors.length] }"><BIconCircleFill /> {{ mapping.trait.name }}<b-badge variant="light" class="ml-1">{{ getTraitTypeText(mapping.trait) }}</b-badge></span>
        </template>

        <b-input-group v-if="mapping.trait.type === 'date'">
          <!-- For date types, show a datepicker -->
          <b-form-input :id="`trait-${index}`"
                            :ref="`trait-${index}`"
                            :state="formState[index]"
                            @keyup.enter="handleDateInput(index)"
                            :value="values[index]"
                            type="date"
                            :key="values[index]"
                            @keyup.exact="(event) => handleDateInputChar(index, event)"
                            @change="(event) => onDateChanged(event, index)" />
          <b-input-group-append>
            <b-button v-b-tooltip="$t('tooltipDataEntryDateMinusOne')" @click="setDateMinusOne(index)"><BIconCaretLeftFill /></b-button>
            <b-button v-b-tooltip="$t('tooltipDataEntryDateToday')" @click="setDateToday(index)"><BIconCalendar3 /></b-button>
            <b-button v-b-tooltip="$t('tooltipDataEntryDatePlusOne')" @click="setDatePlusOne(index)"><BIconCaretRightFill /></b-button>
            <b-button v-b-tooltip="$t('tooltipDataEntryDateReset')" variant="danger" @click="resetDate(index)"><BIconSlashCircle /></b-button>
          </b-input-group-append>
        </b-input-group>
        <!-- For int types, show a number input, apply restrictions -->
        <b-form-input      v-else-if="mapping.trait.type === 'int'"
                           :id="`trait-${index}`"
                           :ref="`trait-${index}`"
                           :state="formState[index]"
                           @keyup.enter="traverseForm(index + 1)"
                           v-model.number="values[index]"
                           type="number"
                           :min="(mapping.trait.restrictions && mapping.trait.restrictions.min !== null && mapping.trait.restrictions.min !== undefined) ? mapping.trait.restrictions.min : null"
                           :max="(mapping.trait.restrictions && mapping.trait.restrictions.max !== null && mapping.trait.restrictions.max !== undefined) ? mapping.trait.restrictions.max : null" />
        <!-- For float types, show a number input, apply restrictions -->
        <b-form-input      v-else-if="mapping.trait.type === 'float'"
                           :id="`trait-${index}`"
                           :ref="`trait-${index}`"
                           :state="formState[index]"
                           @keyup.enter="traverseForm(index + 1)"
                           v-model.number="values[index]"
                           type="number"
                           :min="(mapping.trait.restrictions && mapping.trait.restrictions.min !== null && mapping.trait.restrictions.min !== undefined) ? mapping.trait.restrictions.min : null"
                           :max="(mapping.trait.restrictions && mapping.trait.restrictions.max !== null && mapping.trait.restrictions.max !== undefined) ? mapping.trait.restrictions.max : null"
                           :step="0.02" />
        <!-- For text types, show a simple input field -->
        <b-form-input      v-else-if="mapping.trait.type === 'text'"
                           :id="`trait-${index}`"
                           :ref="`trait-${index}`"
                           :state="formState[index]"
                           @keyup.enter="traverseForm(index + 1)"
                           v-model="values[index]"
                           type="text" />
        <!-- For categorical traits -->
        <template v-if="mapping.trait.type === 'categorical' && mapping.trait.restrictions && mapping.trait.restrictions.categories">
          <!-- If there are more than 3 options, show a dropdown select -->
          <b-form-select   v-if="mapping.trait.restrictions.categories.length > 3"
                           :id="`trait-${index}`"
                           :ref="`trait-${index}`"
                           :state="formState[index]"
                           @keyup.enter="traverseForm(index + 1)"
                           v-model="values[index]"
                           :options="[{ value: null, text: $t('formSelectCategory') }, ...mapping.trait.restrictions.categories]" />
          <!-- Else show a button group for easier selection -->
          <b-form-radio-group v-else
                           :id="`trait-${index}`"
                           :ref="`trait-${index}`"
                           :state="formState[index]"
                           buttons
                           button-variant="outline-secondary"
                           @keyup.enter="traverseForm(index + 1)"
                           v-model="values[index]"
                           :options="[...mapping.trait.restrictions.categories, { value: null, text: '⦸' }]" />
        </template>
      </b-form-group>

      <!-- User comments -->
      <b-form-group :label="$t('formLabelComment')" label-for="comment">
        <b-input-group>
          <b-form-input v-model="comment" id="comment" />
          <b-input-group-addon append v-if="supportsSpeechRecognition">
            <b-button @click="toggleRecording" :variant="speechRecognition ? 'danger' : null" v-b-tooltip="$t('tooltipDataEntryCommentMicrophone')"><BIconMic /></b-button>
          </b-input-group-addon>
        </b-input-group>
      </b-form-group>
    </b-form>
    <!-- Show a button for image tagging -->
    <b-button @click="$refs.imageModal.show()"><BIconCameraFill/> {{ $t('buttonTakePhoto') }}</b-button>
    <!-- Image tagging modal -->
    <ImageModal :name="name" ref="imageModal" />
  </div>
</template>

<script>
import Vue from 'vue'
import ImageModal from '@/components/modals/ImageModal'
import { mapGetters } from 'vuex'
import { EventBus } from '@/plugins/event-bus.js'
import { BIconCameraFill, BIconCircleFill, BIconMic, BIconCaretLeftFill, BIconCaretRightFill, BIconCalendar3, BIconSlashCircle } from 'bootstrap-vue'

export default {
  components: {
    BIconCameraFill,
    BIconCircleFill,
    BIconCaretLeftFill,
    BIconCaretRightFill,
    BIconCalendar3,
    BIconMic,
    BIconSlashCircle,
    ImageModal
  },
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
    },
    isMarked: {
      type: Boolean,
      default: false
    }
  },
  data: function () {
    return {
      values: {
      },
      dates: {
      },
      name: null,
      comment: null,
      imageFile: null,
      imageData: null,
      formValidated: false,
      formState: [],
      textSynth: null,
      speechRecognition: null,
      dateInput: '',
      dateInputIndex: null
    }
  },
  computed: {
     /** Mapgetters exposing the store configuration */
    ...mapGetters([
      'storeData',
      'storeGeolocation',
      'storeHideToggledTraits',
      'storeTraitColors',
      'storeTraits',
      'storeUseGps',
      'storeUseSpeech',
      'storeVisibleTraits'
    ]),
    supportsSpeechRecognition: function () {
      return ('webkitSpeechRecognition' in window)
    },
    visibleTraitMapping: function () {
      const result = []

      this.storeTraits.forEach((t, index) => {
        if (!this.storeHideToggledTraits || this.storeVisibleTraits[index]) {
          result.push({
            trait: t,
            index: index
          })
        }
      })

      return result
    }
  },
  watch: {
    storeUseSpeech: function (newValue) {
      if (newValue) {
        this.textSynth = window.speechSynthesis
      } else {
        this.textSynth = null
      }
    }
  },
  methods: {
    toggleRecording: function () {
      if (this.supportsSpeechRecognition) {
        if (this.speechRecognition) {
          this.disableSpeechRecognition()
        } else {
          // eslint-disable-next-line new-cap
          this.speechRecognition = new window.webkitSpeechRecognition()
          this.speechRecognition.continuous = true
          this.speechRecognition.interimResults = true

          this.speechRecognition.start()
          this.speechRecognition.onresult = event => {
            let result = ''

            for (var i = event.resultIndex; i < event.results.length; ++i) {
              result += event.results[i][0].transcript
            }
            this.comment = result
          }
        }
      }
    },
    handleDateInput: function (index) {
      if (this.dateInput.length > 0 && !isNaN(this.dateInput)) {
        const number = +this.dateInput

        const current = this.getToday()
        current.setDate(current.getDate() + number)

        Vue.set(this.values, index, current.toISOString().split('T')[0])

        this.dateInput = ''
        this.dateInputIndex = null
      }

      this.traverseForm(index + 1)
    },
    handleDateInputChar: function (index, event) {
      if (this.dateInputIndex !== index) {
        this.dateInputIndex = index
        this.dateInput = ''
      }

      // If this could be part of a number
      if (event.key === '-' || event.key === '+' || !isNaN(event.key)) {
        this.dateInput += event.key
      }
    },
    disableSpeechRecognition: function () {
      if (this.speechRecognition) {
        this.speechRecognition.stop()
        this.speechRecognition = null
      }
    },
    setDateMinusOne: function (index) {
      let current = this.values[index]

      if (current === undefined || current === null || current === '') {
        current = this.getToday()
      } else {
        current = new Date(current)
      }

      current.setDate(current.getDate() - 1)

      Vue.set(this.values, index, current.toISOString().split('T')[0])
    },
    setDatePlusOne: function (index) {
      let current = this.values[index]

      if (current === undefined || current === null || current === '') {
        current = this.getToday()
      } else {
        current = new Date(current)
      }

      current.setDate(current.getDate() + 1)

      Vue.set(this.values, index, current.toISOString().split('T')[0])
    },
    resetDate: function (index) {
      Vue.set(this.values, index, null)
    },
    setDateToday: function (index) {
      Vue.set(this.values, index, this.getTodayString())
    },
    getToday: function () {
      let today = new Date()
      const offset = today.getTimezoneOffset()
      return new Date(today.getTime() + (offset * 60 * 1000))
    },
    getTodayString: function () {
      return this.getToday().toISOString().split('T')[0]
    },
    reset: function () {
      this.formValidated = false
      this.formState = this.visibleTraitMapping.map(t => null)
      this.imageFile = null
      this.imageData = null
      if (this.col !== null && this.row !== null) {
        const dp = this.storeData.get(`${this.row}-${this.col}`)
        this.values = this.visibleTraitMapping.map(t => dp.values[t.index])
        this.dates = this.visibleTraitMapping.map(t => dp.dates[t.index])
        this.name = dp.name
        // this.isMarked = dp.isMarked || false
        this.comment = dp.comment

        this.speak(this.name)
        this.speak(this.visibleTraitMapping[0].trait.name)
        this.setFocus()
      }
    },
    speak: function (text) {
      if (this.textSynth) {
        const utterance = new SpeechSynthesisUtterance(text)
        utterance.rate = 1.2
        this.textSynth.speak(utterance)
      }
    },
    /**
     * Sets focus to the first trait input
     */
    setFocus: function () {
      this.$nextTick(() => {
        if (this.$refs['trait-0'][0] && this.$refs['trait-0'][0].focus) {
          this.$refs['trait-0'][0].focus()
        }
        if (this.$refs['trait-0'][0] && this.$refs['trait-0'][0].select) {
          this.$refs['trait-0'][0].select()
        }
      })
    },
    /**
     * Traverses the form and focusses the next input field in turn based on the given index
     * @param newIndex The index of the next field (the method will internally modulo this to stay in range)
     */
    traverseForm: function (newIndex) {
      const oldIndex = newIndex - 1

      this.speak(this.values[oldIndex])

      // If the next ref exists and it has a focus method, call it
      if (newIndex < this.visibleTraitMapping.length) {
        if (this.$refs[`trait-${newIndex}`][0]) {
          if (this.$refs[`trait-${newIndex}`][0].focus) {
            this.$refs[`trait-${newIndex}`][0].focus()
          }
          if (this.$refs[`trait-${newIndex}`][0].select) {
            this.$refs[`trait-${newIndex}`][0].select()
          }

          this.speak(this.visibleTraitMapping[newIndex].trait.name)
        }
      } else {
        this.onSubmit()
      }
    },
    /**
     * Handle date change events
     * @param event The native event
     * @param index The trait index
     */
    onDateChanged: function (event, index) {
      // If the input is something that would be considered "empty", reset the value and date
      if (event === null || event === undefined || event === '') {
        this.values[index] = null
        this.dates[index] = null
      } else {
        Vue.set(this.values, index, event)
        this.traverseForm(index + 1)
      }
    },
    verify: function (verifyCallback) {
      // TODO
      this.formState = this.visibleTraitMapping.map((t, i) => {
        if (this.values[i] === '' || this.values[i] === null) {
          return true
        } else if (t.restrictions) {
          if (t.type === 'categorical') {
            // Check whether the value is one of the pre-defined categories
            return t.restrictions.categories.indexOf(this.values[i]) !== -1
          } else if (t.type === 'int' || t.type === 'float') {
            // Check whether the value lies between the required min and max
            return t.restrictions.min <= this.values[i] && this.values[i] <= t.restrictions.max
          }
        } else {
          return true
        }
      })

      // If the form is invalid, return
      this.formValidated = true
      for (let i = 0; i < this.formState.length; i++) {
        if (this.formState[i] === false) {
          verifyCallback(false)
          return
        }
      }

      this.values.forEach((v, i) => {
        // Ignore empty values, set them to null
        if (this.values[i] === '' || this.values[i] === null) {
          this.values[i] = null
          this.dates[i] = null
          verifyCallback(false)
          return
        }

        if (this.visibleTraitMapping[i].trait.type === 'date') {
          // If the trait a date, simply set the date to the value
          this.dates[i] = this.values[i]
        } else if (this.values[i] !== null && this.dates[i] === null) {
          // Else, if there is a value and no date, set the date as now
          this.dates[i] = this.getTodayString()
        }
      })

      // Only store the comment if it's not empty
      const comment = (this.comment !== undefined && this.comment !== null && this.comment.length > 0) ? this.comment : null

      const dp = this.storeData.get(`${this.row}-${this.col}`)
      const finalValues = JSON.parse(JSON.stringify(dp.values))
      const finalDates = JSON.parse(JSON.stringify(dp.dates))

      this.visibleTraitMapping.forEach((t, index) => {
        finalValues[t.index] = this.values[index]
        finalDates[t.index] = this.dates[index]
      })

      // Update the store with the newly set data point
      this.$store.commit('ON_DATA_POINT_CHANGED_MUTATION', {
        row: this.row,
        col: this.col,
        name: this.name,
        isMarked: this.isMarked,
        values: finalValues,
        dates: finalDates,
        geolocation: this.storeUseGps ? this.storeGeolocation : null,
        comment: comment
      })

      EventBus.$emit('data-point-changed', this.row, this.col)
      verifyCallback(true)
    }
  },
  created: function () {
    if (this.storeUseSpeech && window.speechSynthesis) {
      this.textSynth = window.speechSynthesis
    }

    this.reset()
  }
}
</script>

<style>

</style>
