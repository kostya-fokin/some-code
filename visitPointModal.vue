<template>
  <div class="form__container">
    <ob-form
      class="form"
      data-qa="form"
      :form-data="formData"
      :is-loading="isLoading"
      :is-show-controls="false"
    >
      <div
        v-if="!hasContractors"
        class="form__add-contractor"
        data-qa="addContractor"
        @click="openContractorModal()"
      >
        + Добавить {{ CONSIGNOR ? 'грузоотправителя' : 'грузополучателя' }}
      </div>

      <template
        v-if="hasContractors"
      >
        <ob-select
          ref="contractorSelect"
          class="mb10"
          :value="formData.consigneeOrConsignor.uuid"
          :label="CONSIGNOR ? 'Грузоотправитель' : 'Грузополучатель'"
          :items="form.contractor.items"
          :placeholder="form.contractor.placeholder"
          :itemBottom="{
            icon: 'plus',
            title: `Добавить ${CONSIGNOR ? 'грузоотправителя' : 'грузополучателя'}`,
          }"
          required
          data-qa="contractorSelect"
          @selected="handleContractorChange"
          @bottom="openContractorModal()"
        >
          <template v-slot="{ items }">
            <div
              v-if="items"
              class="form__list"
            >
              <div
                v-for="(item, id) in items"
                :key="item.uuid + id"
                class="form__list-item"
                :data-qa="'contractorSelect_' + id"
                @click="handleContractorChange({ uuid: item.uuid, title: item.title })"
              >
                <div class="form__list-item-left">
                  <div class="form__list-item-title">
                    {{ item.title }}
                  </div>
                  <div class="form__list-item-inn">
                    {{ item.inn }}
                  </div>
                </div>
                <i
                  class="icon edit"
                  @click.stop="openContractorModal(item.uuid, true)"
                />
              </div>
            </div>
          </template>
        </ob-select>
        <div
          v-if="formData.consigneeOrConsignor.uuid"
          class="form__contractor-info"
        >
          <div
            v-if="currentInn"
            class="form__contractor-inn"
          >
            ИНН
            <span>{{ currentInn }}</span>
          </div>
          <div
            v-else
            :class="{ 'form__contractor-inn-error': CONSIGNOR }"
          >
            ИНН не указан
          </div>
          <div
            class="form__contractor-edit"
            data-qa="editContractor"
            @click="openContractorModal(formData.consigneeOrConsignor.uuid)"
          >
            <i class="icon edit" />
            Редактировать
          </div>
        </div>
      </template>

      <div class="form__title">Слот</div>
      <div
        class="form__row form__row--half"
        :class="{ 'form__row--last': !(!isContractorSelected && isReferenceLocation) }"
      >
        <div>
          <ob-datepicker
            v-model="formData.timeSlotStartAt"
            label="Начало слота"
            data-qa="timeSlotStartAt"
            :sinceTo="disabledStartDates"
            :blockTimeDateBefore="new Date(pvpDate)"
            :isValidDate="!this.errors.TIMESLOTSTARTAT && !this.errors.TIMESLOT"
            isValidateImmediatly
            @input="validate"
            is-time
            required
          />
          <span class="error" v-if="this.errors.TIMESLOTSTARTAT">{{ this.errors.TIMESLOTSTARTAT }}</span>
        </div>
        <div>
          <ob-datepicker
            v-model="formData.timeSlotEndAt"
            label="Окончание слота"
            data-qa="timeSlotEndAt"
            :sinceTo="disabledEndDates"
            :blockTimeDateBefore="new Date(pvpDate)"
            :isValidDate="!this.errors.TIMESLOTENDAT && !this.errors.TIMESLOT"
            isValidateImmediatly
            @input="validate"
            is-time
            required
          />
          <span class="error" v-if="this.errors.TIMESLOTENDAT">{{ this.errors.TIMESLOTENDAT }}</span>
        </div>
        <span class="error" v-if="this.errors.TIMESLOT">{{ this.errors.TIMESLOT }}</span>
      </div>
      <div
        v-if="!isContractorSelected && isReferenceLocation"
        class="form__row form__row--last form__row form__row--wrap"
      >
        <ob-address-input
          title="Фактическое место прибытия"
          placeholder="Укажите адрес"
          required
          disabled
          :address="addressLabel"
          :mapAddress="mapAddress"
          :showMap="false"
          customIcon="map"
          customAction
          prefixIcon="pin"
          :locationsTypes="locationsTypes"
          :isAddressOnly="true"
          mapId="add-modal-map"
          display-exp="russian_address"
        />
        <span class="error" v-if="this.errors.POINTADDRESS">{{ this.errors.POINTADDRESS }}</span>
      </div>

      <template v-if="isContractorSelected">
        <div class="form__title">Параметры погрузки</div>
        <div class="form__row form__row--wrap">
          <ob-address-input
            v-if="infographs.length === 0 || showPointAddressInput"
            title="Фактическое место прибытия"
            placeholder="Укажите адрес"
            required
            :address="formData.pointAddress"
            :mapAddress="mapAddress"
            :showMap="false"
            customIcon="map"
            customAction
            prefixIcon="pin"
            :disabled="!isReferenceLocation"
            :locationsTypes="locationsTypes"
            :isAddressOnly="true"
            :isValid="!this.errors.POINTADDRESS"
            mapId="add-modal-map"
            display-exp="russian_address"
            data-qa="pointAddress"
            @input="setPointAddress"
            @on-change-address="setPointAddress"
            @customAction="onOpenMapModal(mapAddress)"
          />
          <!-- <span class="error" v-if="this.errors.POINTADDRESS">{{ this.errors.POINTADDRESS }}</span> -->
          <!-- :disabled="isAddressInputDisabled" -->
          <!-- :popoverMessage="this.form.address.errorTooltip" -->
          <!-- :isValid="form.address.valid" -->
          <div
            v-if="errors.POINTADDRESS"
            class="form__alert"
          >
            <i class="icon alert" />
            Необходимо указать точный фактический адрес
          </div>
          <ob-select
            v-if="infographs.length > 0 && !showPointAddressInput"
            ref="pointAddressSelect"
            :value="formData.pointAddress"
            display-exp="pointAddress"
            value-exp="pointAddress"
            label="Фактическое место прибытия"
            :items="infographs"
            data-qa="pointAddressSelect"
            required
            :itemBottom="{
              icon: 'plus',
              title: `Добавить новый адрес`,
            }"
            @bottom="handleNewPointAddress"
            @selected="handlePointAddressChange"
          />
          <!-- <span class="error" v-if="this.errors.POINTADDRESS">{{ this.errors.POINTADDRESS }}</span> -->

        </div>
        <div class="form__row">
          <ob-input
            v-if="!tsdAddressesList || showTsdAddressInput"
            v-model="formData.tsdAddress"
            label="Адрес для печатной формы заявки"
            placeholder="Укажите адрес"
            data-qa="tsdAddress"
            required
            @blur="validate"
          />
          <ob-select
            v-else
            ref="tsdAddressSelect"
            :value="formData.tsdAddress"
            label="Адрес для печатной формы заявки"
            :items="tsdAddressesList"
            :placeholder="form.contractor.placeholder"
            display-exp="tsdAddress"
            value-exp="tsdAddress"
            data-qa="tsdAddressSelect"
            required
            :itemBottom="{
              icon: 'plus',
              title: `Добавить новый адрес`,
            }"
            @selected="handleTsdAddressChange"
            @bottom="handleNewTsdAddress"
          />
        </div>
        <div class="form__row form__row--half form__row--last">
          <ob-input
            v-model="formData.pointCode"
            label="Код объекта"
            placeholder="Укажите код"
            data-qa="pointCode"
            @input="validate"
          />
          <ob-input
            v-model="formData.phone"
            label="Телефон для связи"
            mask="+7 (999) 999-9999"
            data-qa="phone"
            @input="validate"
          />
        </div>

        <div class="form__title">Доверенность</div>
        <div class="form__row">
          <ob-input
            v-model="formData.attorneyLetter.fio"
            label="ФИО лица, на которого выдана доверенность"
            placeholder="Укажите ФИО"
            data-qa="fio"
            @input="validate"
          />
        </div>
        <div class="form__row">
          <ob-input
            v-model="formData.attorneyLetter.number"
            label="Номер доверенности лица, выдающего груз"
            placeholder="Укажите код"
            data-qa="number"
            @input="validate"
          />
        </div>
        <div class="form__row form__row--half form__row--last">
          <ob-datepicker
            v-model="formData.attorneyLetter.date"
            label="Дата доверенности"
            placeholder="Укажите дату"
            data-qa="date"
            @input="validate"
          />
          <ob-input
            v-model="formData.attorneyLetter.inn"
            label="ИНН выдавшего доверенность"
            placeholder="Укажите код"
            mask="999999999999"
            data-qa="inn"
            @input="validate"
          />
        </div>
      </template>

      <div class="form__title">Заметки</div>
      <div class="form__row mb40">
        <ob-textarea
          v-model="formData.comments"
          label="Заметки для водителя"
          placeholder="Введите комментарий"
          data-qa="comments"
          @input="validate"
        />
      </div>
      <div class="form-footer">
        <div class="ob-form-controls-container">
          <div class="form-controls__left">
            <ob-button
              label="Сохранить"
              :disabled="disabledSubmit"
              data-qa="submit"
              @click="handleSubmit"
            >
              <div class="button-loader" v-if="isSaving">
                <svg version="1.1" id="L9" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" viewBox="0 0 100 100" enable-background="new 0 0 0 0" xml:space="preserve">
                  <path fill="currentColor" d="M73,50c0-12.7-10.3-23-23-23S27,37.3,27,50 M30.9,50c0-10.5,8.5-19.1,19.1-19.1S69.1,39.5,69.1,50">
                    <animateTransform
                      attributeName="transform"
                      attributeType="XML"
                      type="rotate"
                      dur="1s"
                      from="0 50 50"
                      to="360 50 50"
                      repeatCount="indefinite"
                    />
                  </path>
                </svg>
              </div>
            </ob-button>
            <ob-button
              type="flat"
              label="Отмена"
              :disabled="isSaving"
              data-qa="cancel"
              @click="handleCancel"
            />
          </div>
        </div>
      </div>
    </ob-form>
    <ob-modal
      ref="addContractor"
      :title="addContractorTitle"
      size="small"
      data-qa="addContractorModal"
    >
      <add-contractor-modal
        :contractor-uuid="contractorUuid"
        :consignor="CONSIGNOR"
        :data="contractorData"
        :key="contractorData.uuid"
        @close="updateContractors"
      />
    </ob-modal>
    <address-map-modal
      ref="mapModal"
      :data-source="{ address: {} }"
      :value="this.activeAddress"
      :locationsTypes="locationsTypes"
      isVisitPoint
      :isRadius="false"
      mapId="add-modal-map"
      dataQa="addContractorAddress"
      @input="setAddressFromModal"
    />
  </div>
</template>

<script>
import moment from "moment";
import { uuid } from 'vue-uuid';
import api from '@/api';
import ObAddressInput from 'common-components/src/obComponents/obAddressInput';
import ObDatepicker from 'common-components/src/obComponents/obDatePickerConstructor';
import ObForm from 'common-components/src/obComponents/obForm';
import ObInput from 'common-components/src/obComponents/obInput';
import ObModal from 'common-components/src/obComponents/obModal';
import ObSelect from 'common-components/src/obComponents/obSelect';
import ObTextarea from 'common-components/src/obComponents/obTextarea';
import ObButton from "common-components/src/obComponents/obButton";
import AddressMapModal from "@/pages/logchainPage/components/logchainMapModal";
import addContractorModal from './addContractorModal';
import cloneDeep from 'clone-deep'

const t = {
  consignor: 'CONSIGNOR',
  consignee: 'CONSIGNEE',
};

export default {
  components: {
    ObAddressInput,
    ObDatepicker,
    ObForm,
    ObInput,
    ObModal,
    ObSelect,
    ObTextarea,
    addContractorModal,
    AddressMapModal,
    ObButton,
  },
  props: {
    contractorUuid: {
      type: String,
      required: true,
    },
    visitPointUuid: {
      type: String,
      required: true,
    },
    locationsTypes: {
      type: Array,
      required: true,
    },
    address: {
      type: Object,
      required: true,
    },
    orderUuid: {
      type: String
    },
    pvpDate: {
      type: String,
      required: true
    },
    brif: {
      type: Object,
      required: true,
    },
    baseLocations: {
      type: Array
    }
  },
  data() {
    return {
      form: {
        contractor: {
          items: [],
          label: `${this.CONSIGNOR ? 'грузоотправителя' : 'грузополучателя'}`,
          placeholder: 'Выберите из списка'
        },
      },
      formData: {
        orderUuid: this.$route.params.id,
        consigneeOrConsignor: {
          uuid: '',
          title: '',
          contractorType: ''
        },
        clientUuid: null,
        pointAddress: '',
        tsdAddress: '',
        pointCode: '',
        pointCoordinate: {
           lat: null,
           lon: null
        },
        timeSlotStartAt: '',
        timeSlotEndAt: '',
        comments: '',
        phone: '',
        attorneyLetter: {
          number: '',
          date: '',
          inn: '',
          fio: ''
        },
        infographUuid: uuid.v4(),
        type: 'ADDRESS',
        locationUuid: '',
      },
      visitPoint: null,
      activeAddress: {},
      isLoading: false,
      contractorData: {},
      errors: {
        CARGOCONTRACTORTITLE: null,
        CARGOCONTRACTORUUID: null,
        POINTADDRESS: null,
        TIMESLOTENDAT: null,
        TIMESLOTSTARTAT: null,
        TIMESLOT: null,
      },
      infographs: [],
      showTsdAddressInput: true,
      showPointAddressInput: true,
      needAddress: false,
      isSaving: false,
    }
  },
  computed: {
    hasContractors() {
      return this.form.contractor.items.length > 0
    },
    contractorType() {
      if (!this.visitPoint) return;

      return this.visitPoint.consigneeOrConsignor.contractorType;
    },
    CONSIGNOR() {
      return this.contractorType === t.consignor;
    },
    CONSIGNEE() {
      if (!this.visitPoint) return;

      return this.contractorType === t.consignee;
    },
    currentInn() {
      const contractorUuid = this.formData.consigneeOrConsignor.uuid
      if (!contractorUuid) return

      const contractor = this.form.contractor.items.find(
        ({ uuid }) => uuid === contractorUuid
      );

      return contractor?.inn || null;
    },
     mapAddress() {
      return {
        coordinates: {
          lat:
            (this.formData.pointCoordinate &&
              this.formData.pointCoordinate.lat) ||
            "",
          lon:
            (this.formData.pointCoordinate &&
              this.formData.pointCoordinate.lon) ||
            "",
        },
        label: this.formData.pointAddress || this.address.russianName,
      };
    },
    isContractorSelected() {
      return !!this.formData.consigneeOrConsignor.uuid
    },
    isReferenceLocation() {
      if (!this.visitPoint) return

      return this.visitPoint.type === 'REFERENCE_POSITION'
    },
    firstPointDt() {
      let dt = null;
      if (this.brif && this.brif.visitPoints && this.brif.visitPoints.find) {
        const point = this.brif.visitPoints.find(point => point.type === 'FROM');
        if (point && point.timeSlot) {
          dt = point.timeSlot.date;
        }
      }
      return dt;
    },
    disabledStartDates() {
      return [
        {
          from: new Date(1976, 0, 0),
          to: new Date(this.firstPointDt).setHours(0, 0, 0),
        },
      ];
    },
    disabledEndDates() {
      return [
        {
          from: new Date(1976, 0, 0),
          to: new Date(this.firstPointDt).setHours(0, 0, 0),
        },
      ];
    },
    tsdAddressesList() {
      if (this.infographs.length === 0 || this.showPointAddressInput) return

      return this.infographs.find(infograph => {
        return infograph.pointAddress === this.formData.pointAddress
      })?.tsdAddresses
    },
    addContractorTitle() {
      if (this.contractorData.uuid) {
        return this.CONSIGNOR ? 'Грузоотправитель' : 'Грузополучатель'
      }

      return this.CONSIGNOR ? 'Новый грузоотправитель' : 'Новый грузополучатель'
    },
    disabledSubmit() {
      const filledForm = !!(
        this.formData.consigneeOrConsignor.uuid &&
        this.formData.timeSlotStartAt &&
        this.formData.timeSlotEndAt &&
        this.formData.pointAddress?.length &&
        this.formData.tsdAddress?.length
      )
      const isInnValid = this.CONSIGNOR ? !!this.currentInn : true

      return !filledForm || this.errors.length > 0 || this.isSaving || !isInnValid;
    },
    addressLabel() {
      return this.baseLocations.find(location => location.uuid === this.formData.locationUuid)?.russianName
    }
  },
  async mounted() {
    this.isLoading = true
    // await this.processVisitPoint({ uuid: this.visitPointUuid })
    this.needAddress = this.address.icon !== "pin"
    this.isLoading = false
  },
  watch: {
    'formData.tsdAddress': {
      async handler() {
        await this.searchInfographs()
      }
    }
  },
  methods: {
    async processVisitPoint({ uuid }) {
      try {
        const { data } = await api.orders.getVisitPoint2({ uuid });
        this.visitPoint = data;
        this.setFormData(this.visitPoint)
      } catch (e) {
        console.error(e);
        this.$notification({
          type: 'system_notification_error',
          message: 'Ошибка чтения данных',
        });
        this.$emit('close');
      }

      try {
        await this.generateContractorsList({
          type: this.CONSIGNOR ? 'CONSIGNOR' : 'CONSIGNEE',
          uuid: this.contractorUuid,
          coordinates: this.visitPoint.pointCoordinate || this.coordinates,
        });
      } catch (e) {
        this.$notification({
          type: 'system_notification_error',
          message: 'Ошибка при генерации списка исполнителей',
        });
        console.error(e);
      }

    },
    async generateContractorsList({ type, uuid, coordinates }) {
      this.form.contractor.items = [];
      let list = [];
      const payloadForDistance = { uuid, coordinates, type };
      const payloadForType = { ownerUuid: uuid, type };

      try {
        try {
          const {
            data: { list: excludedList },
          } = await api.contractors.getContractorsByDistance(
            payloadForDistance
          );
          const formatedExcludedList = excludedList.map((item) => item.uuid);
          payloadForType.list = formatedExcludedList;

          list = list.concat(
            excludedList.map((item) => ({ ...item, location: 'near' }))
          );

        } catch (error) {
          console.error('fetch_contractors error', error);
          this.$notification({
            type: 'system_notification_error',
            message: 'Ошибка загрузки данных',
          });
        }

        const {
          data: { list: othersList },
        } = await api.contractors.getContractorsByType(payloadForType);
        list = list.concat(
          othersList.map((item) => ({ ...item, location: 'other' }))
        );
      } catch (error) {
        console.error('fetch_list_exclude error', error);
      }

      this.form.contractor.items = this.form.contractor.items.concat(list);
    },
    async handleContractorChange({ uuid: contractorUuid, title }) {
      this.formData.consigneeOrConsignor = {
        uuid: contractorUuid,
        title,
        contractorType: this.CONSIGNOR ? 'CONSIGNOR' : 'CONSIGNEE'
      }
      this.$refs.contractorSelect.close();
      await this.getVisitPointAddresses(true)
      if (this.infographs.length === 0) {
        this.formData.infographUuid = uuid.v4();
      }
      await this.searchInfographs()
      this.validate()
    },
    async getVisitPointAddresses(isForced) {
      const payload = {
        uuid: this.visitPointUuid,
        contractorUuid: this.formData.consigneeOrConsignor.uuid,
        contractorType: this.CONSIGNOR ? 'CONSIGNOR' : 'CONSIGNEE',
        clientUuid: this.contractorUuid,
        pointAddress: this.formData.pointAddress,
        isReferenceLocation: this.isReferenceLocation,
        pointCoordinate: this.formData.pointCoordinate
      }
      try {
        const { data } = await api.orders.getVisitPointAddresses(payload);
        this.infographs = data || []
        if (this.infographs.length > 0) {
          this.showPointAddressInput = false
          this.showTsdAddressInput = false
          if (isForced) {
            this.formData.pointAddress = this.infographs[0].pointAddress
            this.formData.tsdAddress = this.infographs[0].tsdAddresses[0].tsdAddress
            this.formData.infographUuid = this.infographs[0].tsdAddresses[0].infographUuid
          }
        } else {
          if (this.visitPoint.type !== 'REFERENCE_POSITION' && !this.formData.tsdAddress) {
            this.formData.tsdAddress = this.formData.pointAddress
          }
        }
      }
      catch (e) {
        console.error('getVisitPointAddresses error', e);
      }
    },
    async init() {
      this.isLoading = true;
      await this.processVisitPoint({ uuid: this.visitPointUuid });
      if (this.needAddress && this.isReferenceLocation) {
        this.formData.pointAddress = this.formData.pointAddress || this.activeAddress.label
      }
      if (this.formData.consigneeOrConsignor.uuid) {
        await this.getVisitPointAddresses()
      }
      this.isLoading = false;
      this.validate();
    },
    async setPointAddress($event) {
      this.needAddress = false;
      const pointAddress = $event;
      this.formData.pointAddress = pointAddress.label;
      if (!this.formData.tsdAddress) {
        this.formData.tsdAddress = pointAddress.label;
      }
      this.formData.pointCoordinate = pointAddress.coordinates;
      this.formData.locationUuid = pointAddress.locationUuid;
      await this.searchInfographs()
      this.validate();
    },
    setAddressFromModal(e) {
      const addr = {
        coordinates: {
          lat: e.payload.coordinates.lat,
          lon: e.payload.coordinates.long,
        },
        isValid: true,
        icon: e.payload.icon,
        label: e.payload.label,
        locationUuid: e.payload.uuid || "",
      };
      this.setPointAddress(addr);
    },
    onOpenMapModal(point) {
      this.activeAddress = point;
      this.$refs.mapModal.open();
    },
    async handleSubmit() {
      this.isSaving = true;
      try {
        await api.orders.saveVisitPoint(this.visitPointUuid, this.formData)
        this.$emit('close')
        this.$emit('visit-point-updated');
      } catch (e) {
        console.error('submit error', e);
      }
      this.isSaving = false;
    },
    async searchInfographs() {
      const payload = {
        consignor_or_consignee_uuid: this.formData.consigneeOrConsignor.uuid,
        owner_uuid: this.contractorUuid,
        point_address: this.formData.pointAddress,
        tsd_address: this.formData.tsdAddress,
        consignee_or_consignor: this.CONSIGNOR ? 'CONSIGNOR' : 'CONSIGNEE',
      }
      try {
        this.clearInfographData()
        const { data } = await api.contractors.searchInfographs(payload)
        const infograph = data[0]
        if (infograph?.uuid) {
          this.formData.infographUuid = infograph.uuid
          this.formData.attorneyLetter = infograph.attorneyLetter;
          this.formData.phone = infograph.phone;
          this.formData.pointCoordinate = infograph.pointCoordinate;
          this.formData.pointCode = infograph.code;
          this.formData.tsdAddress = infograph.tsdAddress;
          this.formData.pointAddress = infograph.pointAddress;
          this.formData.consigneeOrConsignor.uuid = infograph.consigneeOrConsignorUuid;
        }
      }
      catch (e) {
        console.error('searchInfographs error', e);
      }
    },
    openContractorModal(uuid, closeSelect) {
      if (closeSelect) this.$refs.contractorSelect.close();
      if (uuid) this.contractorData = this.findContractor(uuid)
      this.$refs.addContractor.open()
    },
    findContractor(uuid) {
      return this.form.contractor.items.find(item => item.uuid === uuid )
    },
    async updateContractors($event) {
      const { uuid, title } = $event
      this.$refs.addContractor.close()
      try {
        await this.generateContractorsList({
          type: this.CONSIGNOR ? 'CONSIGNOR' : 'CONSIGNEE',
          uuid: this.contractorUuid,
          coordinates: this.visitPoint.pointCoordinate || this.coordinates,
        });
        this.handleContractorChange({ uuid, title })
      }
      catch (e) {
        console.error('fetch_contractors error', e);
        this.$notification({
          type: 'system_notification_error',
          message: 'Ошибка загрузки данных',
        });
      }
    },
    setFormData(visitPoint = {}) {
      this.formData.consigneeOrConsignor = visitPoint.consigneeOrConsignor;
      this.formData.attorneyLetter = visitPoint.attorneyLetter;
      this.formData.infographUuid = visitPoint.infographUuid || uuid.v4();
      this.formData.locationUuid = visitPoint.locationUuid;
      this.formData.phone = visitPoint.phone;
      this.formData.comments = visitPoint.comments;
      this.formData.timeSlotStartAt = moment(visitPoint.timeSlotStartAt);
      this.formData.timeSlotEndAt = moment(visitPoint.timeSlotEndAt);
      this.formData.pointCoordinate = visitPoint.pointCoordinate;
      this.formData.pointCode = visitPoint.pointCode;
      this.formData.tsdAddress = visitPoint.tsdAddress;
      this.formData.pointAddress = visitPoint.pointAddress || this.activeAddress.label || this.addressLabel;
      this.formData.clientUuid = this.contractorUuid
      this.formData.orderUuid = this.$route.params.id
    },
    handleCancel() {
      this.$emit('close')
    },
    async validate() {
      this.clearValidation()
      const payload = cloneDeep(this.formData)
      try {
        const res = await api.orders.validatePoints(this.visitPointUuid, payload)
        for (let error of res) {
          this.errors[error.response.code] = error.response.description
        }
      } catch (e) {
        console.error(e);
      }
    },
    clearValidation() {
      for (let key in this.errors) {
        this.errors[key] = null
      }
    },
    async handlePointAddressChange({ pointAddress, tsdAddresses }) {
      this.formData.pointAddress = pointAddress
      this.formData.tsdAddress = tsdAddresses[0].tsdAddress
      this.formData.infographUuid = tsdAddresses[0].infographUuid
      await this.searchInfographs()
    },
    async handleTsdAddressChange({ infographUuid, tsdAddress }) {
      this.formData.infographUuid = infographUuid
      this.formData.tsdAddress = tsdAddress
      await this.searchInfographs()
    },
    handleNewPointAddress() {
      this.formData.pointAddress = ''
      this.formData.tsdAddress = ''
      this.showPointAddressInput = true
      this.showTsdAddressInput = true
      this.formData.infographUuid = uuid.v4()
    },
    handleNewTsdAddress() {
      this.formData.tsdAddress = ''
      this.showTsdAddressInput = true
      this.formData.infographUuid = uuid.v4()
    },
    clearInfographData() {
      this.formData.pointCode = '';
      this.formData.phone = '';
      this.formData.attorneyLetter = {
        number: '',
        date: '',
        inn: '',
        fio: ''
      }
      this.formData.infographUuid = uuid.v4()
    }
  },
}
</script>

<style lang="scss" scoped>
@import "~common-components/src/static/css/mixins";

.form {
  &__title {
    @include font-mixin(16px, 20px, 700, $color-text-base);
    margin-bottom: 15px;
  }

  &__contractor-info {
    display: flex;
    justify-content: space-between;
    margin-bottom: 30px;
    @include font-mixin(14px, 18px, 400, $gray-slate);
  }

  &__contractor-inn {

    span {
      color: $color-text-base;
    }
  }

  &__contractor-inn-error {
    color: #E62D3E;
  }

  &__contractor-edit {
    display: flex;
    align-items: center;
    cursor: pointer;

    .icon {
      font-size: 15px;
      margin-right: 2px;
    }
  }

  &__row {
    margin-bottom: 20px;
    display: flex;

    & > * {
      width: 100%;
    }

    &--half {
      justify-content: space-between;
      flex-wrap: wrap;

      & > * {
        width: calc(50% - 5px)
      }
    }

    &--last {
      margin-bottom: 30px;
    }

    &--wrap {
      flex-wrap: wrap;
    }

  }

  &__add-contractor {
    @include font-mixin(16px, 20px, 400, #0070A6);
    display: flex;
    border: 1px dashed #80B7D2;
    border-radius: 3px;
    height: 157px;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    margin-bottom: 30px;
  }

  &__alert {
    @include font-mixin(14px, 20px, 400, #191E32);
    display: flex;
    align-items: center;
    padding: 10px;
    width: 100%;
    background-color: rgba(#FFB137, .2);
    margin-top: 10px;

    i {
      color: #FFB137;
      font-size: 16px;
      margin-right: 12px;
    }
  }

  &__list-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    padding: 5px 20px;
    cursor: pointer;

    .icon {
      font-size: 16px;
      color: #A3A5AD;
    }

    &:hover {
      background-color: #EAF5FA;
    }
  }

  &__list-item-title {
    @include font-mixin(16px, 19px, 400, #191E32);
    margin-bottom: 2px;
  }

  &__list-item-inn {
    @include font-mixin(12px, 14px, 400, #737682);
  }
}

.mb10 {
  margin-bottom: 10px;
}

.mb40 {
  margin-bottom: 40px;
}

.error {
  display: block;
  width: 100%;
  margin-top: 5px;
  font-size: 12px;
  color: $color-error;
}

.button-loader svg {
  width: 40px;
  height: 40px;
}
</style>