<template>
  <single-pane class="regions__outer">
    <ob-func-panel
      :columns="[]"
      :show-sort="false"
      :show-group="false"
      :show-column-filter="false"
      :show-search="false"
      add-button-text="Добавить регион"
      :show-add-button="true"
      @add="openRegionModal"
    />
    <div
      v-if="regions.length > 0"
      class="regions"
    >
      <div class="regions__header">
        <div class="regions__header-title w700">
          Наименование региона
        </div>
        <div class="regions__header-title w170">
          Последние изменения
        </div>
        <div class="regions__header-title">
          Автор
        </div>
      </div>
      <div class="regions__content">
        <div
          v-for="region in regions"
          :key="region.uuid"
          class="regions__region"
        >
          <div
            class="regions__region-row"
            @click="openRegion(region.uuid)"
          >
            <div class="regions__region-chevron">
              <i
                :class="['icon', { 'chevron-down': !region.opened, 'chevron-up': region.opened, }]"
              />
            </div>
            <div class="regions__region-title w700">
              {{ region.title }}
            </div>
            <div class="regions__region-changed w170">
              {{ region.date }}
            </div>
            <div class="regions__region-author w220">
              {{ region.author.firstname }} {{ region.author.lastname }}
            </div>
            <div class="regions__region-count">
              Опорников: {{ region.countOfLocations }}
            </div>
            <div class="regions__controls">
              <i
                class="icon edit"
                title="Редактировать"
                @click.stop="openRegionModal({ uuid: region.uuid })"
              />
              <i
                class="icon close"
                title="Удалить"
                @click.stop="openConfirmRegionDeleteModal(region.uuid)"
              />
            </div>
          </div>
          <expand-transition
            scrollIntoView
            block="nearest"
          >
            <div
              v-if="region.opened"
              class="regions__list"
            >
              <div
                v-for="location in region.locations"
                :key="location.uuid"
                class="regions__list-item"
              >
                <i class="icon city-sm" />
                {{ location.title }}
                <i
                  class="icon close"
                  @click="openConfirmLocationDeleteModal(region.uuid, location.uuid, location.title)"
                />
              </div>
              <div
                class="regions__list-item regions__list-item--edit"
                @click="openRegionModal({ uuid: region.uuid })"
              >
                <i class="icon edit" />
                Редактировать список
              </div>
            </div>
          </expand-transition>
        </div>
      </div>
    </div>
    <div
      v-else
      class="regions__empty"
    >
      Здесь вы сможете настраивать регионы.<br />
      Для этого добавьте хотя бы один регион.
      <div
        class="regions__empty-add"
        @click="openRegionModal"
      >
        <i class="icon plus" />
        Добавить регион
      </div>
    </div>
    <div
      v-if="regions.length > 0"
      class="regions__count"
    >
      <div class="regions__count-value">
        Всего: {{ pagination.totalElements }}
      </div>
      <ob-pagination
        v-if="showPagination"
        is-wide
        class="regions__pagination"
        :page="pagination.page"
        :count="pagination.totalElements"
        :page-limit="pagination.pageLimit"
        :is-page-input="true"
        @onChangePage="onChangePage"
        @on-change="onChangePage"
      />
    </div>
    <ob-modal
      ref="regionModal"
      title="Регион"
      :title-second="currentTitle ? 'Редактирование' : 'Добавление'"
      size="medium"
      :key="`${currentUuid}_${id}`"
    >
      <region-modal
        :uuid="currentUuid"
        :title="currentTitle"
        :locations="currentLocations"
        @close="closeRegionModal"
      />
    </ob-modal>
    <ob-modal
      ref="confirmRegionDeleteModal"
      title="Удаление региона"
      ok-text="Удалить"
      cancel-text="Отмена"
      :is-error="true"
      :is-show-controls="true"
      @submit="deleteRegion(regionToDelete.uuid)"
    >
      <p>Наименование: {{ regionToDelete.title }}</p>
      <p>Локаций: {{ regionToDelete.count }}</p>
    </ob-modal>
    <ob-modal
      ref="confirmLocationDeleteModal"
      title="Удаление локации"
      ok-text="Удалить"
      cancel-text="Отмена"
      :is-error="true"
      :is-show-controls="true"
      @submit="deleteFromRegion(locationToDelete.regionUuid, locationToDelete.uuid)"
    >
      <p>Локация: <i class="icon city-sm confirm-icon" /> {{ locationToDelete.title }}</p>
    </ob-modal>
  </single-pane>
</template>

<script>
import cloneDeep from 'clone-deep'
import api from '@/api'
import { uuid } from 'vue-uuid'
import moment from 'moment'
import paging                from '@/mixins/dictionaries/paging';
import itemMixin             from '@/mixins/dictionaries/edit-item';
import ObModal               from 'common-components/src/obComponents/obModal';
import obFuncPanel from 'common-components/src/obComponents/obFuncPanel';
import SinglePane            from 'common-components/src/obComponents/obSinglePanels';
import RegionModal from './modal/region';
import expandTransition from '@/components/ExpandTransition';
import obPagination from 'common-components/src/obComponents/obPagination';

export default {
  components: {
    ObModal,
    SinglePane,
    obFuncPanel,
    RegionModal,
    expandTransition,
    obPagination,
  },
  mixins: [
    paging,
    itemMixin,
  ],
  data() {
    return {
      regions: [],
      currentUuid: null,
      currentTitle: null,
      currentLocations: null,
      regionToDelete: {
        uuid: null,
        title: null,
        count: null
      },
      locationToDelete: {
        uuid: null,
        title: null,
        regionUuid: null
      },
      id: 0,
      pagination: {
        page: 0,
        count: 0,
        pageLimit: 20,
        totalPages: 0,
        totalElements: 0
      },
    }
  },
  async mounted() {
    await this.getRegionsList()
  },
  computed: {
    showPagination() {
      return this.pagination.totalPages > 1;
    },
  },
  methods: {
    async openRegionModal({ uuid }) {
      this.id += 1
      await this.setCurrent(uuid)
      this.$refs.regionModal.open()
    },
    async closeRegionModal({ isEdit, uuid }) {
      this.$refs.regionModal.close()

      if (isEdit) {
        const region = this.regions.find(region => region.uuid === uuid)

        try {
          const data = await api.dictionaries.regions.getRegion(uuid)
          region.locations = data.locations
          region.countOfLocations = region.locations.length
        } catch (err) {
          console.error(err);
        }
        return
      }

      await this.getRegionsList()
    },
    async getRegionsList() {
      try {
        const data = await api.dictionaries.regions.getRegionsList({
          page: this.pagination.page,
          size: this.pagination.size
        })
        this.pagination.page = data.page;
        this.pagination.count = data.numberOfElements;
        this.pagination.totalElements = data.totalElements;
        this.pagination.totalPages = data.totalPages;
        this.regions = data.content.map(item => {
          return { ...item, date: moment(item.updatedAt).format('DD.MM.YYYY, HH:mm') ,opened: false, locations: [] }
        })
      } catch (err) {
        console.error(err);
      }
    },
    async openRegion(uuid, noNeedOpen) {
      const region = this.regions.find(region => region.uuid === uuid)

      if (region.opened) {
        region.opened = false
        return
      }
      if (region.locations.length) {
        region.opened = true
        return
      }

      try {
        const data = await api.dictionaries.regions.getRegion(uuid)
        region.locations = data.locations
        if (!noNeedOpen) {
          region.opened = true
        }
      } catch (err) {
        console.error(err);
      }

    },
    async deleteRegion(uuid) {
      try {
        await api.dictionaries.regions.deleteRegion(uuid)
        await this.getRegionsList()
        this.$refs.confirmRegionDeleteModal.close()
      } catch (err) {
        console.error(err);
      }
    },
    async deleteFromRegion(regionUuid, locationUuid) {
      const region = this.regions.find(region => region.uuid === regionUuid)
      if (region.locations.length === 1) {
        try {
          await api.dictionaries.regions.deleteRegion(regionUuid)
          this.$refs.confirmLocationDeleteModal.close()
          await this.getRegionsList()
          return
        } catch (err) {
          console.error(err);
        }
      }

      try {
        await api.dictionaries.regions.deleteFromRegion({
          regionUuid,
          locationUuid
        })

        try {
          const data = await api.dictionaries.regions.getRegion(regionUuid)
          region.locations = data.locations
          region.countOfLocations -= 1

          this.$refs.confirmLocationDeleteModal.close()
        } catch (err) {
          console.error(err);
        }
      } catch (err) {
        console.error(err);
      }
    },
    generateUuid () {
      return uuid.v4()
    },
    async setCurrent(uuid) {
      this.currentTitle = ''
      this.currentLocations = []
      this.currentUuid = uuid || this.generateUuid()
      const region = this.regions.find(region => region.uuid === uuid)
      if (region) {
        if (region.locations.length === 0) {
          await this.openRegion(uuid, true)
        }
        let currentLocations = cloneDeep(region.locations)
        this.currentLocations = currentLocations.map(location => {
          return { ...location, icon: 'city-sm' }
        })
        this.currentTitle = region.title
      }
    },
    openConfirmRegionDeleteModal(uuid) {
      const region = this.regions.find(region => region.uuid === uuid)
      this.regionToDelete = {
        title: region.title,
        count: region.countOfLocations,
        uuid: region.uuid
      }
      this.$refs.confirmRegionDeleteModal.open()
    },
    openConfirmLocationDeleteModal(regionUuid, locationUuid, title) {
      this.locationToDelete = {
        title,
        regionUuid,
        uuid: locationUuid
      }
      this.$refs.confirmLocationDeleteModal.open()
    },
    async onChangePage(page) {
      if (page !== this.pagination.page) {
        this.pagination.page = page
        try {
          await this.getRegionsList()
        } catch (err) {
          console.error(err);
        }
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.regions {
  &__outer {
    position: relative;
    padding-bottom: 34px;
  }

  &__header {
    display: flex;
    text-transform: uppercase;
    color: $shale-gray;
    font-size: 10px;
    padding: 15px 0 16px 66px;
  }

  &__region-row {
    display: flex;
    align-items: center;
    background-color: $blue-150;
    padding: 20px 0;
    font-size: 14px;
    cursor: pointer;

    .icon {
      font-size: 16px;
      color: $color-grey-span;

      &.edit {
        margin-right: 20px;
        opacity: 0;
      }
    }

    &:hover {
      .edit {
        opacity: 1;
      }
    }
  }

  &__controls {
    margin-left: auto;
    margin-right: 25px;
  }

  &__region-chevron {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 66px;

    .icon {
      margin: auto;
      font-size: 26px;
      color: $color-blue-navy-blue;
    }
  }

  &__region-title {
    color: $color-blue-navy-blue;
  }

  &__region-count {
    color: $gray-slate;
    font-size: 12px;
  }

  &__list {
    padding-left: 66px;
  }

  &__list-item {
    display: flex;
    align-items: center;
    font-size: 14px;
    line-height: 20px;
    padding: 11px 0;
    box-shadow: 0px -1px 0px 0px $blue-150 inset;

    &--edit {
      color: $color-blue-navy-blue;
      box-shadow: none;
      cursor: pointer;
    }

    .city-sm {
      margin-right: 8px;
      color: $color-blue-navy-blue;
    }

    .close {
      margin-left: auto;
      margin-right: 25px;
      font-size: 16px;
      color: $color-grey-span;
      cursor: pointer;
    }

    .edit {
      color: $color-grey-span;
      font-size: 16px;
      margin-right: 3px;
    }
  }

  &__empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 100%;
    color: $color-grey-span;
    font-size: 24px;
    line-height: 40px;
    margin-top: 70px;
  }

  &__empty-add {
    display: flex;
    align-items: center;
    color: $color-blue-navy-blue;
    font-size: 16px;
    margin-top: 20px;
    cursor: pointer;

    .icon {
      color: $color-grey-span;
      font-size: 24px;
      margin-right: 8px;
    }

    &:hover {
      opacity: .8;
    }
  }

  &__count {
    display: flex;
    align-items: center;
    position: absolute;
    width: 100%;
    bottom: 0;
    left: 0;
    text-transform: uppercase;
    color: $shale-gray;
    font-size: 12px;
    line-height: 30px;
    padding-left: 20px;
    padding-right: 20px;
    background-color: $background-color-light;

    &::after {
      content: "";
      position: absolute;
      left: 0;
      top: -4px;
      width: 100%;
      background: linear-gradient(180deg, rgba(196, 213, 237, 0.5) 0%, rgba(207, 222, 244, 0) 100%);
      opacity: 0.4;
      height: 4px;
    }
  }

  &__count-value {
    flex-grow: 1;
  }

  &__pagination {
    width: auto !important;
    padding: 5px 0;
  }

  .w170 {
    width: 170px;
  }

  .w220 {
    width: 220px;
  }

  .w700 {
    width: 700px;
  }

}
.confirm-icon {
  color: $color-blue-navy-blue;
  margin-left: 10px;
}
</style>