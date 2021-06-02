
<script>
  import { invert } from 'lodash'
  import moment from 'moment'
  import { mapActions, mapGetters, mapMutations, mapState } from 'vuex'

  import {
    BaseDialog,
    BaseForm,
  } from '@/components/lazy'
  import FormFieldsGenerator from '@/components/lazy'// '@/components/forms/form-fields-generator/FormFieldsGenerator'

  import {
    BLOOD_READING_TIMES,
    BLOOD_READING_TYPES,
    DATE_FORMATS,
    GLUCOSE_UNITS,
    HBA1C_UNITS,
  } from '@/components/lazy'// '@/constants'

  import i18n from '@/components/lazy'// '@/mixins/i18n'
  import requireImage from '@/components/lazy'// '@/mixins/requireImage'
  import testid from '@/components/lazy'// '@/mixins/testid/testid'

  import eventBus from '@/components/lazy'// '@/tools/event-bus'

  const baseTestid = 'dialog-glucose-tracking-log'

  export default /*#__PURE__*/defineComponent({
    name: 'GlucoseTracking', // vue component name
    components: {
      BaseDialog,
      BaseForm,
      FormFieldsGenerator,
    },

    mixins: [
      i18n,
      testid(baseTestid),
      requireImage,
    ],

    props: {
      date: {
        type: [String, Object],
        required: false,
        default: null,
      },
      id: {
        type: [String, Number],
        required: false,
        default: null,
      },
      readingValue: {
        type: [String, Number],
        required: false,
        default: null,
      },
      time: {
        type: [String, Number],
        required: false,
        default: null,
      },
      type: {
        type: [String, Number],
        required: false,
        default: null,
      },
    },

    created() {
      if (this.isEdit) {
        this.fillForm()
      } else {
        this.form.date = moment.utc().utcOffset(this.getTimezone).format(DATE_FORMATS.dateShort)
        this.form.hour = moment.utc().utcOffset(this.getTimezone).format(DATE_FORMATS.time)
      }
    },

    watch: {
      /* eslint-disable-next-line object-shorthand */
      'form.type'() {
        if (this.isEdit) return

        this.form.value = null
      },
    },

    data() {
      return {
        form: {
          type: BLOOD_READING_TYPES.glucose,
          date: '',
          hour: '',
          time: Object.keys(BLOOD_READING_TIMES)[0],
          value: null,
        },
      }
    },

    methods: {
      ...mapActions('dialog', [
        'closeDialog',
      ]),
      ...mapActions('glucoseTracking', [
        'saveReading',
        'updateReading',
      ]),
      ...mapMutations('snackbars', [
        'addSnackbar',
      ]),
      onSubmit(isValid) {
        if (!isValid) return

        const reading = {
          isGlucose: this.isGlucose,
          timestamp: this.timestamp,
          type: this.form.type,
          value: this.form.value,
          ...(this.form.time ? { time: BLOOD_READING_TIMES[this.form.time] } : null),
        }

        if (this.isEdit) {
          this.updateReading({
            reading,
            id: this.id,
          }).finally(this.finalize)
        } else {
          this.saveReading(reading)
            .finally(this.finalize)
        }
      },
      fillForm() {
        this.form = {
          date: moment
            .parseZone(this.date)
            .format(DATE_FORMATS.dateShort),
          hour: moment
            .parseZone(this.date)
            .format(DATE_FORMATS.time),
          time: invert(BLOOD_READING_TIMES)[this.time],
          type: this.type,
          value: this.readingValue,
        }
      },
      finalize() {
        this.addSnackbar({ message: this.$t('Saved') })
        eventBus.$emit('glucoseTracking/refresh')
        this.closeDialog()
      },
    },

    computed: {
      ...mapGetters('loading', [
        'getLoadingStatesForActions',
      ]),
      ...mapState('glucoseTracking', [
        'settings',
      ]),
      ...mapGetters('user', [
        'getTimezone',
      ]),
      isSaving() {
        return this.getLoadingStatesForActions([
          'glucoseTracking/logReading',
          'glucoseTracking/updateReading',
        ])
      },
      isEdit() {
        return !!this.id
      },
      formFields() {
        return [
          ...(this.isEdit ? [null] : this.typeFields),
          this.dateField,
          ...(this.isGlucose ? this.glucoseFields : [null]),
          this.valueField,
        ].filter(item => item !== null)
      },
      valueField() {
        return {
          type: 'InputGroup',
          name: 'value',
          class: 'log-glucose-dialog__form-field',
          props: {
            afterText: this.unit,
            hasHiddenAsterisk: true,
            label: this.i18n('value'),
            step: this.unitStep,
            type: 'number',
            validation: {
              required: true,
              min_value: this.unitMinValue,
              max_value: this.unitMaxValue,
              decimal: [this.unitDecimalPlaces, '.'],
            },
          },
        }
      },
      dateField() {
        /* eslint-disable camelcase */
        return {
          type: 'CalendarPickerGroup',
          class: 'log-glucose-dialog__form-field',
          name: 'date',
          props: {
            format: DATE_FORMATS.dateShort,
            formatted: DATE_FORMATS.date,
            hasHiddenAsterisk: true,
            label: 'Select the date of your reading',
            maxDate: moment
              .utc()
              .utcOffset(this.getTimezone)
              .format('YYYY-MM-DD 23:59'),
            minDate: moment
              .utc(this.settings?.created_at, 'YYYY-MM-DD HH:mm:ss')
              .utcOffset(this.getTimezone)
              .format('YYYY-MM-DD 00:00'),
            testid: this.dataTestid,
            outputFormat: DATE_FORMATS.dateShort,
            validation: { required: true },
          },
        }
        /* eslint-enable camelcase */
      },
      unit() {
        /* eslint-disable camelcase */
        return this.isGlucose
          ? GLUCOSE_UNITS[this.settings?.glucose_units]
          : HBA1C_UNITS[this.settings?.hba1c_units]
        /* eslint-enable camelcase */
      },
      isGlucose() {
        return this.form.type === BLOOD_READING_TYPES.glucose
      },
      isHba1c() {
        return this.form.type === BLOOD_READING_TYPES.hba1c
      },
      isMgDl() {
        return this.unit === GLUCOSE_UNITS[1]
      },
      isMmolL() {
        return this.unit === GLUCOSE_UNITS[2]
      },
      isPercentage() {
        return this.unit === HBA1C_UNITS[1]
      },
      isMmolMol() {
        return this.unit === HBA1C_UNITS[2]
      },
      unitDecimalPlaces() {
        return this.isMgDl || this.isMmolMol ? 0 : 1
      },
      unitMinValue() {
        return this.isPercentage ? 2.2 : 0
      },
      unitMaxValue() {
        return this.isGlucose
          ? this.isMgDl
            ? 10000
            : 556
          : this.isMmolMol
            ? 10000
            : 918
      },
      unitStep() {
        return this.unitDecimalPlaces === 1 ? 0.1 : 1
      },
      timestamp() {
        const date = moment
          .utc(this.form.date, DATE_FORMATS.dateShort)
          .utcOffset(this.getTimezone, true)
        const hour = moment
          .utc(this.form.hour, 'HH:mm')
          .utcOffset(this.getTimezone, true)

        if (this.isGlucose) {
          date.set({
            hour: hour.hour(),
            minute: hour.minutes(),
          })
        }

        return date
      },
      submitText() {
        return this.isEdit ? 'Edit reading' : 'Log reading'
      },
      headerText() {
        return this.isEdit
          ? this.isGlucose
            ? 'Edit blood glucose'
            : 'Edit HbA1c'
          : 'Log blood glucose'
      },
      hourPickerPosition() {
        return this.isMinMd ? null : 'top'
      },
      typeFields() {
        return [{
          type: 'RadioMultiGroup',
          name: 'type',
          class: 'log-glucose-dialog__radio',
          props: {
            legend: this.i18n('reading-type'),
            labelKey: 'label',
            options: [
              { value: BLOOD_READING_TYPES.glucose, label: this.$t('Blood glucose') },
              { value: BLOOD_READING_TYPES.hba1c, label: this.$t('HbA1c') },
            ],
            testid: baseTestid,
            validation: { required: true },
            valueKey: 'value',
          },
        }]
      },
      glucoseFields() {
        return [{
          type: 'HourPickerGroup',
          name: 'hour',
          class: 'log-glucose-dialog__form-field',
          props: {
            label: this.i18n('time-label'),
            format: 'HH:mm',
            testid: baseTestid,
            validation: { required: true },
          },
        }, {
          type: 'SelectGroup',
          name: 'time',
          class: 'log-glucose-dialog__form-field',
          props: {
            hasHiddenAsterisk: true,
            label: this.i18n('time-of-day-label'),
            options: Object.keys(BLOOD_READING_TIMES),
            testid: baseTestid,
            validation: { required: true },
          },
        }]
      },
    },

    slug: 'views.plugins.glucose-tracking.glucose-tracking-log',
  });
</script>


<template>
  <base-dialog
    is-overflow-visible
    class="glucose-tracking"
    data-testid="dialog-glucose-tracking-log"
    :headerLevel="1"
    :image="requireImage('modular-tile/top-tracking-log.jpg')"
    v-bind="{
      headerText,
    }"
  >
    <div
      v-if="formFields"
      class="glucose-tracking__content"
      data-testid="dialog-glucose-tracking-log-content"
    >
      <base-form
        :data-testid="dataTestid"
        @submit="onSubmit"
        v-bind="{ isSaving, submitText }"
      >
        <form-fields-generator
          :dataModel="form"
          v-bind="{ formFields }"
        />
      </base-form>
    </div>
  </base-dialog>
</template>

<style scoped>
.glucose-tracking.dialog {
	 max-width: 37rem;
	 max-height: none;
}
 .glucose-tracking .dialog__container--is-visible {
	 margin: -0.5rem 0 1rem 0;
}
 .glucose-tracking__content {
	 width: 100%;
}
 .glucose-tracking__radio {
	 margin: 0 0 -1rem;
}
 .glucose-tracking__form-field {
	 margin: 1.8rem 0 0;
}
 .glucose-tracking.modular-tile__main {
	 padding: 0 3rem 3rem;
}
 .glucose-tracking .form__end-row {
	 margin: 3rem 0 0;
}
 .glucose-tracking .base-select {
	 padding: 0;
}
 .glucose-tracking .input-group:first-child {
	 margin: 0;
}
 @media all and (min-width: 416px) and (max-width: 1024px) {
	 .glucose-tracking .hour-picker .date-time-picker .datetimepicker {
		 top: 0 !important;
		 bottom: 100% !important;
	}
}
 @media all and (min-width: 416px) and (max-width: 1024px) {
	 .glucose-tracking .hour-picker .date-time-picker .datetimepicker .datepicker {
		 top: unset !important;
		 bottom: 100% !important;
	}
}
 
</style>
