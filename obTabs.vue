<template>
  <div class="ob-tabs-container">
    <div
      ref="tab-panel"
      class="ob-tabs__panel"
      :class="tabPanelClasses"
    >
      <div
        v-for="(item, index) in tabsView"
        ref="tabs-button"
        :key="`tab-${item.index}`"
        :data-index="item.index"
        class="ob-tabs__panel-item"
        :class="{
          '--active': selected && getItemByIndex(item.index)[valueExp] === selected[valueExp],
          '--hidden': visibleIndex != null && item.index > visibleIndex,
          '--hidden': getItemByIndex(item.index).isHidden,
          '--disabled': getItemByIndex(item.index).isDisabled,
          '--secondary': getItemByIndex(item.index).isSecondary,
          '--right-alignment': getItemByIndex(item.index).isAlignRight,
        }"
      >
        <div
          v-if="item.isDivider"
          v-show="Boolean(tabsView[index + 1])"
          class="ob-tabs__panel-item-divider"
        />
        <template v-else>
          <!-- tab w/ tooltip -->
          <template v-if="getItemByIndex(item.index).tooltip">
            <ObTooltip
              :content="getItemByIndex(item.index).tooltip.content"
              :position="getItemByIndex(item.index).tooltip.position || 'top'"
              :close-after="getItemByIndex(item.index).tooltip.closeAfter || 3000"
              :open-delay="getItemByIndex(item.index).tooltip.openDelay || 200"
            >
              <button
                :style="getButtonStyles(item, item.index)"
                :class="{
                  '--with-notification': getItemByIndex(item.index).notification,
                  '--error': getItemByIndex(item.index).notification === 'error'
                }"
                class="ob-tabs__panel-button"
                :data-qa="item.qa || dataQa"
                @click="onClickTab(item.index)"
              >
                <template v-if="$scopedSlots.button">
                  <slot
                    name="button"
                    :data="getItemByIndex(item.index)"
                  />
                </template>
                <template v-else-if="getItemByIndex(item.index).tabButtonHtml">
                  <div v-html="getItemByIndex(item.index).tabButtonHtml" />
                </template>
                <template v-else>
                  <i
                    v-if="getItemByIndex(item.index).icon"
                    class="ob-tabs__panel-button-icon"
                    :class="'icon ' + (getItemByIndex(item.index).icon)"
                  />
                  <span
                    v-if="getItemByIndex(item.index).title"
                  >
                    {{ getItemByIndex(item.index).title }}
                  </span>
                </template>
              </button>
            </ObTooltip>
          </template>
          <!-- tab without tooltip -->
          <template v-else>
            <button
              :style="getButtonStyles(item, item.index)"
              :class="{
                '--with-notification': getItemByIndex(item.index).notification,
                '--error': getItemByIndex(item.index).notification === 'error'
              }"
              :disabled="getItemByIndex(item.index).isDisabled"
              class="ob-tabs__panel-button"
              :data-qa="item.qa || dataQa"
              @click="onClickTab(item.index)"
            >
              <template v-if="$scopedSlots.button">
                <slot
                  name="button"
                  :data="getItemByIndex(item.index)"
                />
              </template>
              <template v-else-if="getItemByIndex(item.index).tabButtonHtml">
                <div v-html="getItemByIndex(item.index).tabButtonHtml" />
              </template>
              <template v-else>
                <i
                  v-if="getItemByIndex(item.index).icon"
                  class="ob-tabs__panel-button-icon"
                  :class="'icon ' + getItemByIndex(item.index).icon"
                />
                <span
                  v-if="getItemByIndex(item.index).title"
                >
                  {{ getItemByIndex(item.index).title }}
                </span>
              </template>
            </button>
          </template>
        </template>
      </div>
      <template v-if="dropdownTabs">
        <ObMenuDropdown
          :items="dropdownTabs || []"
          :is-arrow="true"
          :data-qa="dataQa"
          label="Еще"
        />
      </template>
      <!-- Added v-if to show sliding line without initial sliding -->
      <div
        v-if="transform"
        class="ob-tabs__sliding-line"
        :style="{
          background: gradient,
          transform: transform
        }"
      />
    </div>
    <div class="ob-tabs__content">
      <slot />
    </div>
  </div>
</template>
<script>
import ObMenuDropdown from '../../obComponents/obMenuDropdown'
import ObTooltip from '../../obComponents/obTooltip'

export default {
  name: 'ObTabs',

  components: {
    ObMenuDropdown,
    ObTooltip
  },

  props: {
    items: {
      type: Array,
      default: null
    },
    buttonMargin: {
      type: Number,
      default: 0
    },
    valueExp: {
      type: String,
      default: 'title'
    },
    value: {
      type: [Object, String]
    },
    noBorder: {
      type: Boolean,
      default: false
    },
    justifyToCenter: {
      type: Boolean,
      default: false
    },
    dataQa: {
      type: String
    },
    itemsWithDivider: {
      type: Array,
      default: () => []
    },
    beforeTabChange: {
      type: Function,
      default: undefined
    }
  },

  data () {
    return {
      prevItems: this.items,
      selected: {},
      visibleIndex: null,
      gradient: null,
      transform: null
    }
  },

  computed: {
    tabs () {
      const result = this.items || this.$slots.default
        .filter(item => item.componentOptions && this.isTabPane(item.componentOptions.tag))
        .map(item => {
          return item.componentOptions ? item.componentOptions.propsData : item
        })

      return result.map((item, index) => {
        return { ...item, index }
      })
    },
    tabsView () {
      let result = []

      this.tabs.forEach(item => {
        const index = item.index
        const data = this.getItemByIndex(index)

        result.push(item)

        if (this.itemsWithDivider.some(item => [data[this.valueExp], index, data.uuid].includes(item))) {
          result.push({
            isDivider: true
          })
        }
      })

      return result
    },
    tabsSlots () {
      if (!this.$slots.default) return

      return this.$slots.default.filter(item => item.componentOptions && this.isTabPane(item.componentOptions.tag))
    },
    tabsButtons () {
      return this.$refs['tabs-button']
    },
    dropdownTabs () {
      const { visibleIndex, tabs } = this
      if (
        visibleIndex === tabs.length ||
        visibleIndex === null ||
        !visibleIndex
      ) return null

      return tabs
        .map((item, index) => {
          return {
            ...item,
            isSelected: this.selected[this.valueExp] === item[this.valueExp],
            method: () => this.onClickTab(index)
          }
        })
        .slice(visibleIndex + 1)
    },
    tabPanelClasses () {
      return {
        '--with-border': !this.noBorder,
        '--to-center': this.justifyToCenter
      }
    }
  },
  watch: {
    value (val) {
      if (val[this.valueExp] !== this.selected[this.valueExp]) this.selected = val || {}
    }
  },
  mounted () {
    this.$nextTick(() => {
      this.setInitSelected()
    })
    window.addEventListener('resize', this.calcWidth)
  },
  updated () {
    this.calcWidth()
    const {
      gradient,
      transform
    } = this.generateGradient()
    this.gradient = gradient
    this.transform = transform
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.calcWidth)
  },
  methods: {
    setInitSelected () {
      let selected
      if (!this.tabs.length) return null

      if (!this.value) {
        selected = this.tabs.find(item => item.isSelected)
      } else {
        const itemByValue = this.tabs.find(item => item[this.valueExp] === this.value[this.valueExp])
        if (itemByValue) {
          itemByValue.isSelected = true
        }
        selected = this.value
      }

      this.selected = selected || this.tabs[0]
      if (this.selected && (!this.value || this.value[this.valueExp] !== this.selected[this.valueExp])) this.$emit('on-change-tab', this.selected)
    },
    async onClickTab (index) {
      if (typeof (this.beforeTabChange) === 'function' && !await this.beforeTabChange(this.selected, this.getItemByIndex(index))) {
        return false
      }
      this.selected = this.getItemByIndex(index)
      this.$emit('on-change-tab', this.selected)
      this.$emit('input', this.selected)
    },
    calcWidth () {
      let result
      let width = +this.$refs['tab-panel'].offsetWidth
      const tabsWidth = this.tabsButtons.map(item => item.getBoundingClientRect().width)
      const tabsWidthSum = tabsWidth.reduce((a, b) => a + b)
      if (tabsWidthSum > width) {
        width = width - 40

        this.tabs.forEach((item, index) => {
          const elWidth = this.tabsButtons[index].getBoundingClientRect().width
          width = width - +elWidth
          if (width > 0) result = index
        })
      }
      this.visibleIndex = result
    },
    getButtonStyles (item, index) {
      let marginLeft = ''
      if (index) marginLeft = this.buttonMargin + 'px'

      return {
        marginLeft
      }
    },
    getItemByIndex (index) {
      if (index === undefined) return {}

      const { items, tabsSlots } = this
      if (items && items.length) return items[index]

      return (tabsSlots && tabsSlots[index] && tabsSlots[index].componentInstance) || {}
    },
    isTabPane: tagName => ['ObTabsPane', 'ob-tabs-pane'].includes(tagName),
    generateGradient () {
      if (!this.tabsButtons || !this.tabsButtons.length) return
      const btn = this.tabsButtons.find((btn, idx) => {
        const index = btn.dataset.index
        return this.getItemByIndex(index)[this.valueExp] === this.selected[this.valueExp]
      })
      const currentColor = btn && this.getItemByIndex(btn.dataset.index) && this.getItemByIndex(btn.dataset.index).isSecondary
        ? '--tabs-secondary-color'
        : '--tabs-color'
      if (!btn) return { background: 'none', transform: 'none' }

      const startPos = 0
      const endPos = startPos + btn.getBoundingClientRect().width
      const color = `rgba(var(${currentColor}), 0)`
      const colorActive = `rgba(var(${currentColor}), 1)`

      const gradient = `linear-gradient(
        90deg,
        ${color} 0%,
        ${color} ${startPos}px,
        ${colorActive} ${startPos}px,
        ${colorActive} ${endPos}px,
        ${color} ${endPos}px,
        ${color} 100%)`
      const transform = `translateX(${btn.offsetLeft}px)`

      return { gradient, transform }
    }
  }
}
</script>
<style lang="scss">
@import "../../static/css/mixins";
@import "../../static/css/variables";
.ob-tabs {
  $tab-height: 40px;
  $tab-margin: 10px;
  $main-color: $color-blue-navy-blue;
  $secondary-color: $green-800;

  &-container {
    --tabs-color: #{red($main-color) + ', ' + green($main-color) + ', ' + blue($main-color)};
    --tabs-secondary-color: #{red($secondary-color) + ', ' + green($secondary-color) + ', ' + blue($secondary-color)};

    width: 100%;
  }
  &__panel {
    position: relative;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    &-item + &-item {
      margin-left: $tab-margin;
    }
    &-item {
      position: relative;
      display: flex;
      align-items: center;
      color: $main-color;
      transition: color .3s ease-in-out;
      &:hover {
        opacity: 0.8;
      }
      &.--active {
        color: RGB(var(--tabs-color));
        &:hover {
          opacity: 1;
          button {
            cursor: default;
          }
        }
      }
      &.--secondary {
        color: RGB(var(--tabs-secondary-color));
        &.--active {
          color: RGB(var(--tabs-secondary-color));
        }
      }
      &.--disabled {
        color: $gray-window;
        &:hover {
          opacity: 1;
        }
      }
      &.--right-alignment {
        margin-left: auto;
      }
      &.--right-alignment ~ &.--right-alignment {
        margin-left: $tab-margin;
      }
    }
    &-item-divider {
      width: 1px;
      height: 20px;
      margin-left: $tab-margin;
      margin-right: $tab-margin;
      background-color: $color-grey-border;
    }
    &-button {
      display: flex;
      align-items: center;
      justify-content: center;
      height: $tab-height;
      padding: 0 10px;
      white-space: nowrap;
      border: none;
      text-align: center;
      outline: none;
      cursor: pointer;
      background: none;
      @include font-mixin(16px, 20px, 400, inherit, 0.4px);
      &:disabled {
        cursor: not-allowed;
      }
      &-icon {
        display: block;
        font-size: 20px;
        &.icon {
          &::before {
            color: inherit;
          }
        }
        & + span {
          padding-left: 8px;
        }
      }
    }
    .ob-menu-dropdown-container {
      position: absolute;
      right: 0;
      top: 0;
      bottom: 0;
      margin: auto;
      .menu-button {
        height: $tab-height;
        width: 100%;
        &-text {
          margin: 0;
        }
      }
    }
  }
  &__sliding-line {
    position: absolute;
    width: 100%;
    left: 0;
    right: 0;
    bottom: -1px;
    height: 2px;
    transition: all .24s ease-in-out;
    transform: translateX(0);
    background: none;
  }
}
.--with-border {
  border-bottom: 1px solid $color-grey-border;
}
.--to-center {
  justify-content: center;
}
.--hidden {
  display: none;
}

.--with-notification {
  &::after {
    content: "";
    width: 6px;
    height: 6px;
    position: relative;
    border-radius: 50%;
    top: -6px;
    left: 4px;
    background-color: $color-blue-navy-blue;
  }

  &.--error {
    &::after {
      background-color: $color-error;
    }
  }
}
</style>
