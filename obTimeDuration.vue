<template>
  <div
    v-click-outside="handleOutsideClick"
    class="time-duration"
    :class="{ disabled: disabled }"
  >
    <div
      class="time-duration__container"
      @click="toggle"
    >
      <div class="time-duration__title">
        <div class="time-duration__label">
          {{ label }}
        </div>
        <div
          class="time-duration__value"
          :class="{ 'is-placeholder': !valueLabel }"
        >
          {{ valueLabel || placeholder }}
        </div>
      </div>
      <div class="time-duration__controls">
        <i
          class="icon chevron-down"
          :class="isOpen ? 'chevron-up' : 'chevron-down'"
        />
      </div>
    </div>
    <div
      v-if="isOpen"
      class="time-duration__form"
    >
      <ob-input
        v-if="hasDays"
        ref="days"
        v-model.number="days"
        label="Дни"
        placeholder="00"
        type="number:strict"
        class="time-duration__input mr10"
        @blur="handleBlur($event, 'days')"
        @keydown.native="handleKeydown($event, 'days', 3)"
      />
      <ob-input
        ref="hours"
        v-model.number="hours"
        label="Часы"
        placeholder="00"
        type="number:strict"
        class="time-duration__input mr10"
        @blur="handleBlur($event, 'hours')"
        @keydown.native="handleKeydown($event, 'hours', hasDays ? 2 : 3)"
      />
      <ob-input
        ref="minutes"
        v-model.number="minutes"
        label="Минуты"
        placeholder="00"
        type="number:strict"
        class="time-duration__input"
        @blur="handleBlur($event, 'minutes', true)"
        @keydown.native="handleKeydown($event, 'minutes', 2)"
      />
    </div>
  </div>
</template>

<script>
import ClickOutside from 'vue-click-outside'
import obInput from '../obInput'

const MINUTES_IN_HOUR = 60
const HOURS_IN_DAY = 24

const KEY_CODE_TAB = 9
const KEY_CODE_ENTER = 13
const KEY_CODE_ESC = 27
const KEY_CODE_LEFT = 37
const KEY_CODE_RIGHT = 39
const DIGIT_PREFIX = 'Digit'

const declOfNum = n => {
  const titles = 'день | дня | дней'.split('|')
  return titles[n % 10 === 1 && n % 100 !== 11 ? 0 : n % 10 >= 2 && n % 10 <= 4 && (n % 100 < 10 || n % 100 >= 20) ? 1 : 2]
}

const getNextRef = ref => {
  if (ref === 'days') return 'hours'
  if (ref === 'hours') return 'minutes'
  return false
}

const getPrevRef = (ref, hasDays) => {
  if (ref === 'hours' && hasDays) return 'days'
  if (ref === 'minutes') return 'hours'
  return false
}

export default {
  components: {
    obInput
  },
  directives: {
    ClickOutside
  },
  props: {
    hasDays: {
      type: Boolean,
      default: true
    },
    value: {
      type: [String, Number]
    },
    label: {
      type: String,
      default: 'Длительность'
    },
    placeholder: {
      type: String,
      default: 'Укажите'
    },
    disabled: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      days: 0,
      hours: 0,
      minutes: 0,
      prevDays: 0,
      prevHours: 0,
      prevMinutes: 0,
      isOpen: false,
      valueLabel: null,
      willReset: {
        days: true,
        hours: true,
        minutes: true
      }
    }
  },
  computed: {
    totalMinutes () {
      const days = this.days || 0
      const hours = this.hours || 0
      const minutes = this.minutes || 0
      let totalMinutes = 0

      totalMinutes += days * HOURS_IN_DAY * MINUTES_IN_HOUR
      totalMinutes += hours * MINUTES_IN_HOUR
      totalMinutes += minutes

      return totalMinutes
    }
  },
  mounted () {
    if (!this.value) return
    let currentMinutes = +this.value

    if (this.hasDays) {
      this.days = Math.trunc(currentMinutes / (MINUTES_IN_HOUR * HOURS_IN_DAY))
      currentMinutes = currentMinutes % (MINUTES_IN_HOUR * HOURS_IN_DAY)
    }

    this.hours = Math.trunc(currentMinutes / MINUTES_IN_HOUR)
    this.minutes = currentMinutes % MINUTES_IN_HOUR

    this.updateDuration(false)
  },
  methods: {
    handleOutsideClick () {
      if (!this.isOpen) return false

      this.updateDuration()
    },
    handleBlur (e, ref) {
      const value = e.target.value

      let maxValue
      if (ref === 'minutes') maxValue = 59
      if (ref === 'hours' && this.hasDays) maxValue = 23

      if (maxValue && +value > maxValue) {
        this[ref] = maxValue
      }
    },
    async handleKeydown (e, ref, maxLength) {
      const {
        key, code, keyCode,
        target: {
          selectionStart,
          selectionEnd,
          value
        }
      } = e

      if (code.startsWith(DIGIT_PREFIX) && this.willReset[ref]) {
        e.preventDefault()
        this[ref] = +key
        this.willReset[ref] = false

        return false
      }

      if (
        code.startsWith(DIGIT_PREFIX) &&
        value.length >= maxLength &&
        selectionStart === selectionEnd
      ) {
        e.preventDefault()
      }

      if ([KEY_CODE_TAB, KEY_CODE_ENTER].includes(keyCode)) {
        e.preventDefault()

        if (!value) this[ref] = 0
        if (getNextRef(ref)) this.$refs[getNextRef(ref)].focus()
        else {
          this.$refs.hours.focus()
          await this.$nextTick()
          this.updateDuration()
        }

        return false
      }

      if (keyCode === KEY_CODE_LEFT) {
        if (
          selectionStart === 0 &&
          selectionEnd === 0 &&
          getPrevRef(ref, this.hasDays)
        ) {
          this.$refs[getPrevRef(ref, this.hasDays)].focus()
        }
      }

      if (keyCode === KEY_CODE_RIGHT) {
        if (
          selectionStart === value.length &&
          selectionEnd === value.length &&
          getNextRef(ref)
        ) {
          this.$refs[getNextRef(ref)].focus()
        }
      }

      if (keyCode === KEY_CODE_ESC) {
        this.toggle()

        this.days = this.prevDays
        this.hours = this.prevHours
        this.minutes = this.prevMinutes
      }
    },
    async toggle () {
      if (this.disabled) return false

      const focusRef = this.hasDays ? 'days' : 'hours'
      this.isOpen = !this.isOpen

      await this.$nextTick()

      if (this.isOpen) this.$refs[focusRef].focus()
    },
    updateDuration (toggle) {
      if (toggle !== false) this.toggle()

      this.prevDays = this.days
      this.prevHours = this.hours
      this.prevMinutes = this.minutes
      this.willReset.days = true
      this.willReset.hours = true
      this.willReset.minutes = true

      if (!this.days && !this.hours && !this.minutes) {
        this.days = 0
        this.hours = 0
        this.minutes = 0
        this.valueLabel = null
      }

      this.updateTitle()
      this.$emit('input', this.totalMinutes)
    },
    updateTitle () {
      let textDays, textHours, textMinutes
      let days, hours, minutes

      if (this.hasDays) {
        days = this.days
        hours = this.hours
        minutes = this.minutes
      } else {
        let currentMinutes = +this.totalMinutes
        days = Math.trunc(currentMinutes / (MINUTES_IN_HOUR * HOURS_IN_DAY))
        currentMinutes = currentMinutes % (MINUTES_IN_HOUR * HOURS_IN_DAY)

        hours = Math.trunc(currentMinutes / MINUTES_IN_HOUR)
        minutes = currentMinutes % MINUTES_IN_HOUR
      }

      textDays = days ? `${days} ${declOfNum(days)}` : ''
      textHours = hours ? `${hours} ч` : ''
      textMinutes = minutes ? `${minutes} мин` : ''

      if (textDays) {
        if ((!textHours && textMinutes) || textHours) textDays += ' '
      }
      if (textHours && textMinutes) textHours += ' '

      this.valueLabel = `${textDays}${textHours}${textMinutes}`
    }
  }

}
</script>

<style lang="scss" scoped>
@import "../../static/css/mixins";
@import "../../static/css/variables";

.time-duration {
  position: relative;

  &__container {
    display: flex;
    justify-content: space-between;
    background: $background-white;
    border: 1px solid $color-grey-border;
    box-sizing: border-box;
    border-radius: 3px;
    min-height: 52px;
    width: 100%;
    cursor: pointer;
    outline: none;
    &:hover, &:focus {
      border: 1px $cornflower-craiola solid;
    }
    &_open {
      border: 1px solid $blue-700 !important;
    }
    .disabled &, .readonly &, .form-disabled & {
      cursor: default;
      background: $color-white-lilac;
      border-color: $color-white-lilac !important;
    }
    .not-disabled & {
      background: $background-white;
    }
    .invalid & {
      border: 1px solid $color-error !important;
    }
  }

  &__title {
    padding-top: 6px;
    padding-bottom: 7px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    overflow: hidden;
    width: 100%;

    &_only-label {
      justify-content: center;
    }
  }
  &__controls {
    display: flex;
    flex-direction: column;
    justify-content: center;
    margin-right: 12px;
    .icon {
      display: flex;
      justify-content: center;
      align-items: center;
      transform: translateY(-1px);
      width: 22px;
      height: 12px;
      font-size: 22px;
      color: $color-blue-navy-blue;
    }
    .disabled &, .readonly &, .form-disabled & {
      .icon::before {
        color: $color-grey-border;
      }
    }
    .not-disabled & {
      .icon::before {
        color: $color-blue-navy-blue;
      }
    }
    .invalid & {
      .icon::before {
        color: $color-error;
      }
    }
  }
  &__label {
    @include font-mixin(12px, inherit, 400, $gray-slate, 0.4px);
    margin-top: 0px;
    margin-left: 11px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    display: flex;
    align-items: center;
    pointer-events: none;

    .disabled &, .form-disabled & {
      color: $color-grey-span;
    }
    .not-disabled & {
      color: $gray-slate;
    }
  }

  &__value {
    @include font-mixin(16px, 20px, 400, $color-text-base);
    display: flex;
    align-items: center;
    margin-left: 11px;
    white-space: nowrap;
    text-overflow: ellipsis;
    pointer-events: none;

    .disabled &, .form-disabled & {
      color: $color-grey-span;
    }
    .not-disabled & {
      color: $color-text-base;
    }
    &.is-placeholder {
      color: $gray-window;
    }
    &-icon {
      position: relative;
      left: -1px;
      width: 14px;
      height: 14px;
      margin-right: 7px;
      font-size: 14px;
      color: $blue-700;
      &[class*='risk'] {
        width: 16px;
        height: 16px;
        font-size: 16px;
      }
    }
    &-html {
      margin-right: 8px;
    }
  }

  &__form {
    position: absolute;
    left: 0;
    bottom: -72px;
    display: inline-flex;
    padding: 10px;
    background-color: $background-white;
    box-shadow: 0px 4px 16px 0px #004BC833;
  }

  &__input {
    width: 80px;
  }
}

.mr10 {
  margin-right: 10px;
}
</style>
