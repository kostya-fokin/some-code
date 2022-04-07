<template>
  <div class="ob-input-container">
    <div
      v-click-outside="onOutsideClick"
      class="ob-input-wrap"
      :class="wrapperClass"
      :data-qa="dataQa"
      @click="focus"
    >
      <div class="col-left">
        <label
          v-if="label && !mini"
          class="label"
        >
          <div
            class="label__text ellipsis"
            :title="label"
          >
            {{ label }}
          </div>
          <template
            v-if="labelInfo"
          >
            <ob-tooltip
              class="label__info"
              :content="labelInfo"
              position="top-start"
              :open-delay="500"
              :popper-class="popperClass"
            >
              <i class="icon question" />
            </ob-tooltip>
          </template>
          <span
            v-if="required"
            class="label__required -light-raspberry-red"
          >
            *
          </span>
        </label>
        <div class="input-group">
          <span
            v-if="(preIcon || preIconHtml) && !mini"
            class="pre-icon"
          >
            <ob-tooltip
              :content="preIconInfo"
              :disabled="!preIconInfo"
              :open-delay="500"
              :popper-class="popperClass"
              position="top-start"
            >
              <template
                v-if="preIconHtml"
              >
                <div
                  class="pre-icon__html"
                  v-html="preIconHtml"
                />
              </template>
              <template
                v-if="preIcon"
              >
                <i :class="['icon', preIcon]" />
              </template>
            </ob-tooltip>
          </span>
          <div class="input-block">
            <template v-if="isShowLink">
              <a
                :href="computedLink"
                class="input-block__link"
                target="_blank"
              >
                {{ localValue }}
              </a>
            </template>
            <template
              v-else-if="mask"
            >
              <ob-mask
                ref="input"
                :value="localValue"
                data-attr-el="ob-input-container"
                class="input"
                :mask="mask"
                :is-masked="true"
                :disabled="disabled"
                :readonly="readonly"
                :tokens="maskTokens"
                :mask-handler="maskHandler"
                :placeholder="placeholder"
                :data-qa="dataQa"
                @input="inputEmitter"
                @on-blur="onBlurHandler"
                @on-enter="onKeyDownHandler"
                @on-focus="onFocusNative"
                @on-change="onChangeHandler"
              />
            </template>
            <template
              v-else
            >
              <!--
                Added 'step' attribute for inputs with the type 'number'
                to remove native browser tips like "Please enter a valid value..."
              -->
              <input
                :id="id"
                ref="input"
                :value="localValue"
                :tabindex="allowAction ? 0 : -1"
                data-attr="ob-input-container"
                class="input"
                :placeholder="placeholder"
                :disabled="disabled"
                :readonly="readonly"
                :type="nativeType"
                :data-qa="dataQa"
                @input="inputEmitter"
                @blur="onBlurHandler"
                @keydown="onKeyDownHandler"
                @focus="onFocusNative"
                @change="onChangeHandler"
              >
            </template>
          </div>
        </div>
      </div>
      <div
        v-if="isShowColRight"
        ref="col-right"
        class="col-right"
      >
        <div
          v-if="isShowClear"
          class="clear-icon"
          @click="onClearHandler"
        >
          <i class="icon close" />
        </div>
        <div
          v-if="subTitle && !mini"
          :class="[ 'units', { 'active': !disabled && (focused || localValue) } ]"
        >
          <template
            v-if="subTitleItems.length"
          >
            <ob-inline-select
              v-model="localSubTitle"
              :items="subTitleItems"
              :label="subTitleLabel"
              :disabled="disabled"
              :readonly="readonly"
              :data-qa="dataQa"
              @input="inputEmitter"
            />
          </template>
          <template
            v-else
          >
            {{ subTitle }}
          </template>
        </div>
        <div
          v-if="isShowMainIcon"
          class="main-icon"
        >
          <template v-if="isShowLink">
            <a
              :href="computedLink"
              class="main-icon__link"
              target="_blank"
            >
              <i class="icon external-link" />
            </a>
          </template>
          <template v-else>
            <ob-tooltip
              :disabled="!isShowIconTooltip"
              :type="iconTooltipType"
              :content="tooltipContent"
              position="top-end"
              :open-delay="500"
              :popper-class="popperClass"
            >
              <i
                class="icon"
                :class="iconClasses"
                :data-qa="dataQa"
                @click="onClickIcon"
              />
            </ob-tooltip>
          </template>
        </div>
      </div>
    </div>
    <transition name="slideInDown">
      <div
        v-if="remind && localValue"
        class="ob-input__reminder"
      >
        <router-link
          :to="remindTo"
          class="ob-input__reminder-text"
        >
          {{ remind }}
        </router-link>
      </div>
    </transition>
  </div>
</template>
<script>
import FormItemMixin from '../../obComponents/obForm/FormItemMixin'
import ObTooltip from '../../obComponents/obTooltip'
import ClickOutside from 'vue-click-outside'
import ObMask from '../../obComponents/obMask/component'
import ObInlineSelect from '../obInlineSelect'
import masker from '../obMask/utils/masker'
import { formatNumber } from '../../helpers/common'
import UniqueIdMixin from '../../mixins/unique-id'
import { divide, multiply, bignumber } from 'mathjs'

export default {
  name: 'ObInput',
  directives: {
    ClickOutside
  },
  components: {
    ObTooltip,
    ObMask,
    ObInlineSelect
  },
  mixins: [
    FormItemMixin,
    UniqueIdMixin
  ],
  props: {
    value: {
      type: [String, Number]
    },
    placeholder: {
      type: String,
      default: null
    },
    label: {
      type: String,
      default: null
    },
    labelInfo: {
      type: String,
      default: null
    },
    preIcon: {
      type: String,
      default: null
    },
    preIconInfo: {
      type: String,
      default: null
    },
    preIconHtml: {
      type: String,
      default: null
    },
    icon: {
      type: String,
      default: null
    },
    iconClass: {
      type: String,
      default: null
    },
    type: {
      type: String,
      default: 'text'
    },
    required: {
      type: Boolean,
      default: false
    },
    readonly: {
      type: Boolean,
      default: false
    },
    disabled: {
      type: Boolean,
      default: false
    },
    remind: {
      type: String,
      default: null
    },
    remindTo: {
      type: String,
      default: null
    },
    isValid: {
      type: Boolean,
      default: null
    },
    validationMessage: {
      type: String,
      default: null
    },
    tooltip: {
      type: String,
      default: null
    },
    subTitle: {
      type: String,
      default: null
    },
    subTitleLabel: {
      type: String,
      default: 'Label'
    },
    subTitleItems: {
      type: Array,
      default: () => []
    },
    mask: {
      type: [String, Array],
      default: null
    },
    isMasked: {
      type: Boolean,
      default: false
    },
    isValidationMask: {
      type: Boolean,
      default: false
    },
    isDirtyForce: {
      type: Boolean,
      default: false
    },
    theme: {
      type: String,
      default: 'default'
    },
    large: {
      type: Boolean,
      default: false
    },
    mini: {
      type: Boolean,
      default: false
    },
    isConvertMoney: {
      type: Boolean,
      default: true
    },
    dataQa: {
      type: String
    },
    isTrimOnBlur: {
      type: Boolean,
      default: true
    },
    popperClass: {
      type: String,
      default: null
    },
    isClear: {
      type: Boolean,
      default: false
    },
    maskTokens: {
      type: Object
    },
    minValue: {
      type: Number,
      default: null
    },
    link: {
      type: [ String, Object ],
      default: null
    },
    maskHandler: {
      type: [ Function, String, null ],
      default: null
    }
  },
  data () {
    return {
      localValue: '',
      focused: false,
      dirty: false,
      localSubTitle: ''
    }
  },
  computed: {
    nativeType () {
      return ['password'].includes(this.type)
        ? this.type
        : 'text'
    },
    wrapperClass () {
      return {
        '--disabled': this.disabled,
        '--readonly': this.readonly,
        '--focused': this.focused,
        '--success': this.isStateValid,
        '--error': this.isInvalid || (this.isNotNullValid && !this.isValid),
        '--main-tooltip': this.isShowIconTooltip,
        [`--theme-${this.theme}`]: this.theme,
        '--large': this.large,
        '--mini': this.mini
      }
    },
    isInvalid () {
      return this.isInvalidValidator || (this.isNotNullValid && ((!this.isValid && !this.isValidationMask) || (!this.isValidMask)))
    },
    isInvalidValidator () {
      return Boolean(this.mainError)
    },
    isStateValid () {
      return this.isNotNullValid && ((this.isValid && !this.isValidationMask) || (this.isValidationMask && this.isValidMask)) && !this.isInvalid
    },
    isNotNullValid () {
      return (this.isInvalidValidator || this.isValid !== null || this.isValidationMask) && (this.dirty || this.isDirtyForce)
    },
    isValidMask () {
      return this.mask
        ? this.maskedValue.length === this.mask.length
        : true
    },
    isShowMainIcon () {
      if (this.mini) {
        return false
      }

      return (this.isNotNullValid && (this.isStateValid || this.isInvalid || !this.isValidMask)) || this.icon || this.tooltip || this.isShowLink
    },
    isShowClear () {
      return this.isClear && Boolean(this.localValue) && !this.disabled
    },
    isShowIconTooltip () {
      return ((this.isInvalid && ((this.mainError && this.mainError.text) || this.validationMessage)) || (!this.isStateValid && this.tooltip))
    },
    isShowColRight () {
      return this.isShowMainIcon || this.subTitle
    },
    currentTypeModificator () {
      return this.type && this.type.split(':')[1]
    },
    currentType () {
      return this.type && this.type.split(':')[0]
    },
    allowAction () {
      return !this.disabled && !this.readonly
    },
    iconTooltipType () {
      return this.isInvalid
        ? 'error'
        : 'primary'
    },
    iconClasses () {
      return [
        this.iconClass,
        this.iconByState,
        this.iconColorByState,
        this.isOnClickIconListener && this.allowAction && '--clickable'
      ]
    },
    iconByState () {
      if (this.icon) return this.icon

      if (this.isInvalid) return 'alert'
      if (this.isStateValid) return 'checked'

      return 'question'
    },
    iconColorByState () {
      if (this.disabled) return ''
      if (this.isInvalid) return '-red700'
      if (this.isStateValid) return '-green700'

      return '-blue700'
    },
    tooltipContent () {
      if (this.isInvalid) {
        return (this.mainError && this.mainError.text) || this.validationMessage
      }

      return this.tooltip
    },
    unmaskedValue () {
      return this.mask
        ? masker(this.localValue, this.mask, false, this.maskTokens)
        : this.localValue
    },
    maskedValue () {
      return this.mask
        ? masker(this.localValue, this.mask, true, this.maskTokens)
        : ''
    },
    isOnClickIconListener () {
      return Boolean(this.$listeners['on-click-icon'])
    },
    computedLink () {
      if (!this.link) return null

      const isExternalLink = typeof this.link === 'string' && this.link.includes('http')

      return this.$router && !isExternalLink ? this.$router.resolve(this.link).href : this.link
    },
    isShowLink () {
      return this.link && this.computedLink && (this.readonly || this.disabled)
    }
  },
  watch: {
    value (to) {
      let nextValue = this.convertValue(to, 'from')
      nextValue = this.checkValueByType(nextValue)
      const isChangedValue = this.checkChangeValue(this.localValue, nextValue)
      if (isChangedValue) {
        this.localValue = nextValue
        if (this.isMasked) {
          this.$emit('input', this.maskedValue)
        }
      }
    },
    focused (val) {
      if (!val) {
        this.dirty = true
        this.touchField()
      }
    },
    isValidMask (val) {
      if (val) {
        this.$emit('on-mask-valid')
      }
    },
    maskedValue () {
      if (this.isValidMask) {
        this.$emit('on-change', this.maskedValue)
      }
    }
  },
  created () {
    let value = this.convertValue(this.value, 'from')
    value = this.checkValueByType(value)
    this.localValue = value
    this.localSubTitle = this.subTitle

    if (this.isDirtyForce) {
      setTimeout(() => {
        this.dirty = true
        this.touchField()
      }, 0)
    }
  },
  methods: {
    checkChangeValue (current, next) {
      if (this.currentType === 'money') {
        current = current ? current.toString().replace(/\s+/g, '').replace(/[.|,]/g, '.') : null
        next = next ? next.toString().replace(/\s+/g, '').replace(/[.|,]/g, '.') : null
      } else if (this.mask) {
        current = masker(current, this.mask, false, this.maskTokens)
      }
      return current !== next
    },
    inputEmitter (e) {
      let value = e.target ? e.target.value : e
      let computedValue = this.checkValueByType(value)
      this.localValue = computedValue
      if (!this.mask) {
        this.$refs.input.value = computedValue
      }
      this.emitValue(computedValue)
    },
    onChangeHandler (e) {
      let value = e.target ? e.target.value : e
      let emitValue = this.convertValue(value, 'to')
      if (emitValue !== null) {
        emitValue = this.checkValueByType(emitValue, true)
      }
      this.$emit('change', emitValue)
    },
    emitValue (value) {
      let emitValue = this.convertValue(value, 'to')
      if (emitValue !== null) {
        emitValue = this.checkValueByType(emitValue, true)
      }

      if (this.subTitleItems.length) {
        this.$emit('update:subTitle', this.localSubTitle)
      }

      if (emitValue !== null) {
        this.$emit('input', emitValue)
      }
    },
    focusHandler () {
      this.focused = true
      this.$emit('focus')
      this.$refs.input && this.$refs.input.focus()
    },
    focus (event) {
      if (this.allowAction && !this.focused) {
        this.focusHandler(event)
      } else {
        this.$refs.input && this.$refs.input.focus()
      }
    },
    onFocusNative () {
      if (this.allowAction && !this.focused) {
        this.$emit('focus')
        this.focused = true
      }
    },
    onOutsideClick (e) {
      this.onBlurHandler(e)
    },
    onBlurHandler (e) {
      let isFocusMask = e && this.mask && (e.target === this.$refs.input.$refs.input) && !e.relatedTarget
      if (this.allowAction && this.focused && !this.$el.contains(document.activeElement) && !isFocusMask) {
        if (this.localValue && typeof this.localValue === 'string' && this.isTrimOnBlur && ['money', 'number'].includes(this.currentType)) {
          this.localValue = this.localValue.trim().replace(/\s{2,}/g, ' ')
          this.inputEmitter(e)
        }

        this.$nextTick(() => {
          this.$emit('blur', e)
          this.focused = false
        })
      }
    },
    onKeyDownHandler (e) {
      let keyCode = e.keyCode
      if (keyCode === 13) {
        this.$emit('on-enter')
        if (this.form && this.form.isOnPressEnterSubmit) {
          this.form.onKeyPressEnter()
        }
      }
    },
    checkValueByType (value, emit = false) {
      let returnValue

      switch (this.currentType) {
        case 'number':
          returnValue = this.checkNumber(value)
          break
        case 'money':
          returnValue = this.checkMoney(value)
          if (emit && returnValue && typeof returnValue === 'string' && !this.isMasked) {
            returnValue = returnValue.replace(/\s/g, '').replace(',', '.')
          }
          break
        case 'word':
          returnValue = this.checkWord(value)
          break
        default:
          returnValue = value
      }

      return this.mask && (this.isMasked || emit)
        ? masker(returnValue, this.mask, this.isMasked, this.maskTokens)
        : returnValue
    },
    checkNumber (value) {
      if (!value) {
        return ''
      }

      let pattern

      if (this.currentTypeModificator === 'strict') {
        pattern = /[\d]/g
      } else {
        /**
         * Using <input type="text"> and regex, because inputs
         * with the type 'number' depend on browser language and
         * cause troubles with the different separators (comma, dot);
         *
         * Match [number][single dot (optional)][number]
         */
        pattern = /\d+\.?\d*/g
      }

      const parsed = value.toString().match(pattern)

      return parsed ? parsed.join('') : ''
    },
    checkMoney (value) {
      if (!value && value !== 0) return ''

      const MODIFICATOR = +this.currentTypeModificator
      let regExp
      let floatStr = ''

      if (MODIFICATOR === 0) {
        regExp = new RegExp(/-?[0-9]?/g)
      } else {
        regExp = new RegExp(/-?[0-9|,]?/g)
      }
      let allowedSymbols = value.toString().replace('.', ',').replace(/-{2,}/, '-').replace(/,{2,}/, ',').match(regExp)
      allowedSymbols = allowedSymbols ? allowedSymbols.join('') : ''

      let valueNumber = allowedSymbols ? allowedSymbols.replace(',', '.') : 0

      if (allowedSymbols === '-' || allowedSymbols === '-0' || allowedSymbols === '-0,' || /^-?\./.test(valueNumber) || isNaN(valueNumber)) {
        return allowedSymbols
      }
      if (this.minValue !== undefined && this.minValue !== null && valueNumber < this.minValue) {
        return ''
      }

      let splitted = allowedSymbols.split(',')
      if (splitted.length > 1 && MODIFICATOR > 0) {
        floatStr = ',' + splitted.slice(1).join('').substr(0, MODIFICATOR)
      }

      let prefix = valueNumber < 0 ? '-' : ''
      let decimal = Math.abs(+splitted[0])

      return allowedSymbols && allowedSymbols.length ? prefix + formatNumber(decimal) + floatStr : ''
    },
    checkWord (value) {
      if (!value) return ''

      let allowedSymbols = value.toString().match(/[а-яё|a-z|\s]/gi)
      return allowedSymbols ? allowedSymbols.join('').replace(/\s+/, ' ') : ''
    },
    convertValue (value, direction) {
      let result

      switch (this.currentType) {
        case 'money':
          result = this.convertMoney(value, direction)
          break
        case 'number':
          // 123.4 -> 123.4
          // 123.  -> 123
          result = String(value).replace(/(?!\d+)(\.?)$/, '')
          break
        default:
          result = value || value === 0 ? value.toString() : ''
      }

      return result
    },
    convertMoney (value, direction) {
      if (value === 0) return 0

      if (value) {
        value = value.toString().replace(/\s/g, '').replace(',', '.')
      }
      if (value === '-' && direction === 'from') {
        return value
      }
      if ((value === '-' || value === '-0' || value === '-0,' || isNaN(value) || /^-?\./.test(value)) && direction === 'to') {
        return null
      }

      if (!this.isConvertMoney) return value
      const MODIFICATOR = this.currentTypeModificator

      if (value) {
        value = direction === 'from' ? divide(bignumber(value), bignumber(100)) : multiply(bignumber(value), bignumber(100))
        let splitted = value.toString().split('.')
        if (MODIFICATOR > 0 && direction === 'from') {
          value = splitted.length > 1 ? splitted[0] + ',' + (splitted.slice(1).join('').substr(0, MODIFICATOR)) : splitted[0]
        }
        if (MODIFICATOR === '0') {
          value = splitted[0]
        }
      } else {
        value = ''
      }
      return value
    },
    onClickIcon (e) {
      if (this.$parent.$options.name !== 'ObDatepicker') {
        e.stopPropagation() // To allow to closing another one datepicker after new datepicker opening
      }

      if (this.allowAction) {
        this.$emit('on-click-icon')
      }
    },
    onClearHandler () {
      this.localValue = ''
      this.inputEmitter('')
      this.$emit('clear')
    }
  }
}
</script>
<style lang="scss" scoped>
@import "../../static/css/mixins";

$blue: $color-blue-navy-blue;
$focused: $blue-800;
$success: $Green-700;
$danger: $color-error;
$fontColor: $color-text-base;
$disFontColor: $color-grey-span;
$labelFontColor: $color-grey-label;
$disLabelFontColor: $color-grey-span;
$plhColor: $gray-window;
$borderColor: $color-grey-border;
$disabledBackColor: $color-white-lilac;
$iconSize: 24px;

.ob-input-wrap {
  display: flex;
  justify-content: space-between;
  padding: 7px 11px;
  height: 52px;
  background: $background-white;
  border: 1px solid $borderColor;
  border-radius: 3px;
  cursor: text;
  transition: border 0.1s linear;
  perspective: 1000px;
  &.--theme {
    &-blue {
      border: 1px solid $obInput-border_blue;
      &:hover {
        border: 1px solid $cornflower-craiola;
      }
      &.--focused {
        border-color: $focused;
      }
      &.--disabled, &.--readonly {
        border-color: $cornflower-craiola;
        background-color: $lavender;
      }
    }
    &-default {
      border: 1px solid $borderColor;
      &:hover {
        border: 1px solid $cornflower-craiola;
      }
      &.--focused {
        border-color: $focused;
      }
      &.--disabled, &.--readonly {
        border-color: $disabledBackColor;
        background-color: $disabledBackColor;
      }
      &.--disabled {
        color: $color-grey-span;
      }
    }
  }
  &.--warning {
    border-color: $obInput-border_color_warning !important;
  }
  &.--error {
    border-color: $danger !important;
  }
  &.--success {
    border-color: $success !important;
  }
  &.--disabled, &.--readonly {
    cursor: not-allowed;
  }
  &:hover {
    .clear-icon {
      opacity: 1;
    }
  }

  &.--large {
    height: 60px;
    padding-bottom: 9px;
    .col-left {
      justify-content: space-between;
    }
    .label {
      top: 1px;
    }
  }

  &.--mini {
    height: 30px;
    padding: 7px 12px 3px;
    .input {
      font-size: 14px;
      line-height: 20px;
      &::placeholder {
        font-size: 14px;
        line-height: 20px;
      }
    }
  }
}
.col-left {
  min-width: 0;
  display: flex;
  flex-direction: column;
  justify-content: center;
  flex-grow: 2;
}
.col-right {
  display: flex;
  justify-content: space-between;
  flex-shrink: 0;
}
.label {
  position: relative;
  top: -1px;
  display: flex;
  justify-content: flex-start;
  align-items: center;
  height: 16px;
  cursor: text;
  .--disabled &, .--readonly & {
    cursor: not-allowed;
  }
  &__text {
    @include font-mixin(12px, inherit, 400, $labelFontColor, 0.4px);

    .--theme-default.--disabled & {
      color: $color-grey-span;
    }
  }
  &__info {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 10px;
    margin-left: 3px;
    color: $color-blue-navy-blue;
    cursor: default;
  }
  &__required {
    padding-left: 4px;
  }
}
.ellipsis {
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}
.input-group {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  .pre-icon {
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    right: 1px;
    width: 14px;
    height: 14px;
    margin-right: 7px;
    flex-shrink: 0;
    cursor: default;
    i {
      display: flex;
      justify-content: center;
      align-items: center;
      text-align: center;
      font-size: 14px;
      line-height: 14px;
      color: $blue-700;

      .--disabled & {
        color: inherit;
      }

      &[class*='risk'] {
        width: 16px;
        height: 16px;
        font-size: 16px;
      }
    }
    &__html {
      display: flex;
    }
    &::v-deep {
      img {
        width: 14px;
        height: 14px;
      }
    }
  }
  .input {
    @include font-mixin(16px, 20px, 400, inherit, 0.2px);
    width: 100%;
    height: 20px;
    background-color: transparent;
    border: none !important;
    outline: none !important;
    text-align: left;
    .--disabled &, .--readonly & {
      cursor: not-allowed;
    }
    &::placeholder {
      @include font-mixin(16px, 20px, 400, $plhColor, 0.2px);
    }
  }
}
.input-block {
  flex-grow: 2;
  &__link {
    @include font-mixin(16px, 20px, 400, $color-blue-navy-blue);
    &:hover {
      opacity: 0.8;
    }
  }
}
.clear-icon {
  opacity: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: 10px;
  padding: 0 5px;
  color: $color-grey-span;
  font-size: 14px;
  transition: opacity .3s;
  &:hover {
    cursor: pointer;
    opacity: 0.8;
  }
}
.units {
  @include font-mixin(16px, 20px, 400, $plhColor, 0.2px);
  display: flex;
  align-items: flex-end;
  margin-left: 5px;

  &.active {
    color: $color-text-base;
  }
}
.main-icon {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-left: 12px;
  cursor: default;
  i {
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px;
    &.--clickable {
      cursor: pointer;
    }
  }

  .--disabled & {
    color: inherit !important;
  }
  &__link {
    text-decoration: none;
    color: $color-blue-navy-blue;
    &:hover {
      opacity: 0.8;
    }
    .icon {
      font-size: 22px;
    }
  }
}
.slideInDown-enter, .slideInDown-leave-to {
  opacity: 0;
  transform: translateY(-10px);
}
.slideInDown-enter-active, .slideInDown-leave-active {
  transition: all .2s ease-in-out
}
.-blue700 {
  color: $blue-700;
}
.-green700 {
  color: $Green-700;
}
.-red700 {
  color: $color-error;
}
</style>
<style lang="scss">
input:-webkit-autofill {
  -webkit-box-shadow: inset 0 0 0 50px white; /* цвет вашего фона */
  -webkit-text-fill-color: black; /* цвет текста */
}
input:-webkit-autofill:focus {
  -webkit-box-shadow: 0 0 5px 0 white, /* ваш box-shadow для :focus */
  inset 0 0 0 50px white; /* цвет вашего фона */
  -webkit-text-fill-color: black; /* цвет текста */
}
</style>
