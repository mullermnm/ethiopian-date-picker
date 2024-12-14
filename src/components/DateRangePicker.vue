<template>
  <div class="flex gap-4 " style="display: flex; gap: 10%; ">
    <!-- Gregorian Calendar -->
    <div class="w-1/2">
      <h3>Gregorian Calendar</h3>
      <VueDatePicker
        v-model="gregorianDate"
        range
        inline
        :enable-time-picker="false" 
        auto-apply
        @range-start="onGregorianStartDateChange"
        @range-end="onGregorianEndDateChange"
      />
    </div>

    <!-- Ethiopian Calendar -->
    <div class="w-1/2">
      <h3>Ethiopian Calendar</h3>
      <EthiopianCalendar
        v-model="ethiopianDate"
        range
        @range-start="onEthiopianStartDateChange"
        @range-end="onEthiopianEndDateChange"
      />
    </div>

    <!-- Selected Range Display -->
    <div class="selected-range" v-if="gregorianStartDate && gregorianEndDate">
      <h4>Selected Range:</h4>
      <p>Gregorian: {{ gregorianStartDate }} - {{ gregorianEndDate }}</p>
      <p>Ethiopian: {{ ethiopianStartDate }} - {{ ethiopianEndDate }}</p>
    </div>
  </div>
</template>

<script>
import { ref} from 'vue'
import { EthDateTime } from 'ethiopian-calendar-date-converter'
import VueDatePicker from '@vuepic/vue-datepicker'
import EthiopianCalendar from './EthiopianCalendar.vue'
import '@vuepic/vue-datepicker/dist/main.css'

export default {
  name: 'DateRangePicker',
  components: { 
    VueDatePicker,
    EthiopianCalendar
  },
  setup(props, { emit }) {
    const gregorianStartDate = ref()
    const gregorianEndDate = ref()
    const ethiopianStartDate = ref()
    const ethiopianEndDate = ref()
    const gregorianDate = ref()
    const ethiopianDate = ref([])

    // Convert Gregorian to Ethiopian date
    const toEthiopianDate = (gregorianDate) => {
      if (!gregorianDate) return null
      const date = new Date(gregorianDate)
      return EthDateTime.fromEuropeanDate(date)
    }

    const onGregorianStartDateChange = (date) => {
      gregorianStartDate.value = date
      // Update Ethiopian calendar with the start date only
      ethiopianDate.value = [date]
    }

    const onGregorianEndDateChange = (date) => {
      gregorianEndDate.value = date
      // Update Ethiopian calendar with both start and end dates to trigger range selection
      ethiopianDate.value = [gregorianStartDate.value, date]
    }

    const onEthiopianStartDateChange = (date) => {
      gregorianStartDate.value = date
      gregorianDate.value = [date]
    }

    const onEthiopianEndDateChange = (date) => {
      gregorianEndDate.value = date
      gregorianDate.value = [gregorianStartDate.value, date]
    }

    return {
      gregorianStartDate,
      ethiopianDate,
      gregorianEndDate,
      ethiopianEndDate,
      ethiopianStartDate,
      gregorianDate,
      onGregorianStartDateChange,
      onGregorianEndDateChange,
      onEthiopianStartDateChange,
      onEthiopianEndDateChange
    }
  }
}
</script>
