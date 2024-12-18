<template>
  <div class="dual-date-picker">
    <!-- Input Field -->
    <input
      type="text"
      :value="displayValue"
      @click="toggleDatePicker"
      readonly
      placeholder="Select Date"
      class="date-input"
    />

    <!-- Date Picker Modal -->
    <div v-if="isVisible" class="date-picker-modal">
      <div class="date-picker-container">
        <!-- Gregorian Date Picker -->
        <div v-if="showGregorian" class="picker-section">
          <h3>Gregorian Calendar</h3>
          <VueDatePicker
            v-model="gregorianDate"
            range
            inline
            :enable-time-picker="false"
            auto-apply
            @update:model-value="onGregorianDateChange"
          />
        </div>

        <!-- Ethiopian Date Picker -->
        <div v-if="showEthiopian" class="picker-section">
          <h3>Ethiopian Calendar</h3>
          <EthiopianCalendar
            v-model="ethiopianDate"
            :start-date="ethiopianStartDate"
            :end-date="ethiopianEndDate"
            :is-range="true"
            @range-start="onEthiopianRangeStart"
            @range-end="onEthiopianRangeEnd"
          />
        </div>
      </div>

      <!-- Control Buttons -->
      <div class="picker-controls">
        <button @click="confirmSelection" class="confirm-btn">OK</button>
        <button @click="cancelSelection" class="cancel-btn">Cancel</button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue'
import VueDatePicker from '@vuepic/vue-datepicker'
import EthiopianCalendar from './EthiopianCalendar.vue'

export default {
  name: 'DualDatePicker',
  components: {
    VueDatePicker,
    EthiopianCalendar
  },
  props: {
    datePickerType: {
      type: String,
      default: 'both',
      validator: (value) => ['gregorian', 'ethiopian', 'both'].includes(value)
    },
    modelValue: {
      type: Array,
      default: () => []
    }
  },
  emits: ['update:modelValue'],
  setup(props, { emit }) {
    const isVisible = ref(false)
    const gregorianDate = ref([])
    const ethiopianDate = ref(null)
    const ethiopianStartDate = ref(null)
    const ethiopianEndDate = ref(null)

    // Custom date conversion functions
    const toEthiopianDate = (gregorianDate) => {
      if (!gregorianDate) return null
      
      const gregorianYear = gregorianDate.getFullYear()
      const gregorianMonth = gregorianDate.getMonth() + 1
      const gregorianDay = gregorianDate.getDate()
      
      let ethiopianYear, ethiopianMonth, ethiopianDay
      
      // Ethiopian new year is on September 11/12
      const isLeapYear = (year) => (year % 4 === 0 && year % 100 !== 0) || year % 400 === 0
      const newYearDay = isLeapYear(gregorianYear) ? 12 : 11
      
      if (gregorianMonth > 9 || (gregorianMonth === 9 && gregorianDay >= newYearDay)) {
        ethiopianYear = gregorianYear - 7
      } else {
        ethiopianYear = gregorianYear - 8
      }
      
      // Calculate Ethiopian month and day
      let totalDays = 0
      const gregorianMonthDays = [31, isLeapYear(gregorianYear) ? 29 : 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]
      
      // Days from beginning of year to current date
      for (let i = 0; i < gregorianMonth - 1; i++) {
        totalDays += gregorianMonthDays[i]
      }
      totalDays += gregorianDay
      
      // Adjust for Ethiopian new year
      if (gregorianMonth > 9 || (gregorianMonth === 9 && gregorianDay >= newYearDay)) {
        totalDays -= isLeapYear(gregorianYear) ? 255 : 254
      } else {
        totalDays += 110 // Days from Ethiopian new year to end of Gregorian year
      }
      
      ethiopianMonth = Math.ceil(totalDays / 30)
      ethiopianDay = totalDays % 30 || 30
      
      if (ethiopianMonth > 13) {
        ethiopianMonth = 13
        ethiopianDay = totalDays - 360
      }
      
      return { year: ethiopianYear, month: ethiopianMonth, day: ethiopianDay }
    }
    
    const toGregorianDate = (ethDate) => {
      if (!ethDate) return null;
      
      const { year, month, day } = ethDate;
      let gregYear = year + 7;
      let gregMonth, gregDay;

      // Ethiopian new year starts on September 11/12
      if (month <= 4) { // Months after Ethiopian new year
        gregMonth = month + 8;
        gregDay = day + 10;
        if (gregDay > 30) {
          gregMonth++;
          gregDay -= 30;
        }
      } else if (month <= 12) { // Regular months
        gregMonth = month - 4;
        gregDay = day + 10;
        if (gregDay > 30) {
          gregMonth++;
          gregDay -= 30;
        }
        if (gregMonth <= 0) {
          gregMonth += 12;
          gregYear--;
        }
      } else { // Pagume
        gregMonth = 9;
        gregDay = day + 5;
        if (gregDay > 30) {
          gregMonth = 10;
          gregDay -= 30;
        }
      }

      return new Date(gregYear, gregMonth - 1, gregDay);
    }

    // Computed properties for visibility
    const showGregorian = computed(() => 
      props.datePickerType === 'gregorian' || props.datePickerType === 'both'
    )
    const showEthiopian = computed(() => 
      props.datePickerType === 'ethiopian' || props.datePickerType === 'both'
    )

    // Display value based on active picker type
    const displayValue = computed(() => {
      if (!props.modelValue || !Array.isArray(props.modelValue) || props.modelValue.length !== 2) {
        return ''
      }

      const [start, end] = props.modelValue
      if (!start || !end) {
        return ''
      }

      const formatGregorianDate = (date) => {
        return date instanceof Date ? date.toLocaleDateString('en-US', {
          year: 'numeric',
          month: '2-digit',
          day: '2-digit'
        }) : ''
      }

      const formatEthiopianDate = (date) => {
        const ethDate = toEthiopianDate(date)
        return `${ethDate.day.toString().padStart(2, '0')}/${ethDate.month.toString().padStart(2, '0')}/${ethDate.year}`
      }

      if (props.datePickerType === 'ethiopian') {
        const ethStart = toEthiopianDate(start)
        const ethEnd = toEthiopianDate(end)
        return `${ethStart.day.toString().padStart(2, '0')}/${ethStart.month.toString().padStart(2, '0')}/${ethStart.year} - ${ethEnd.day.toString().padStart(2, '0')}/${ethEnd.month.toString().padStart(2, '0')}/${ethEnd.year}`
      }
      
      if (props.datePickerType === 'gregorian') {
        return `${formatGregorianDate(start)} - ${formatGregorianDate(end)}`
      }
      
      // For 'both', show both formats
      return `${formatGregorianDate(start)} - ${formatGregorianDate(end)} / ${formatEthiopianDate(start)} - ${formatEthiopianDate(end)}`
    })

    // Event Handlers
    const toggleDatePicker = () => {
      isVisible.value = !isVisible.value
    }

    const onGregorianDateChange = (dates) => {
      if (!Array.isArray(dates) || dates.length !== 2) return
      
      gregorianDate.value = dates
      const [start, end] = dates
      
      ethiopianStartDate.value = toEthiopianDate(start)
      ethiopianEndDate.value = toEthiopianDate(end)
    }

    const onEthiopianRangeStart = (date) => {
      ethiopianStartDate.value = date
      const gregDate = toGregorianDate(date)
      if (gregDate && (!gregorianDate.value || !Array.isArray(gregorianDate.value))) {
        gregorianDate.value = [gregDate, null]
      } else if (gregDate && Array.isArray(gregorianDate.value)) {
        gregorianDate.value = [gregDate, gregorianDate.value[1]]
      }
    }

    const onEthiopianRangeEnd = (date) => {
      ethiopianEndDate.value = date
      const gregDate = toGregorianDate(date)
      if (gregDate && Array.isArray(gregorianDate.value)) {
        gregorianDate.value = [gregorianDate.value[0], gregDate]
      }
    }

    const confirmSelection = () => {
      emit('update:modelValue', gregorianDate.value)
      isVisible.value = false
    }

    const cancelSelection = () => {
      gregorianDate.value = props.modelValue || []
      isVisible.value = false
    }

    return {
      isVisible,
      gregorianDate,
      ethiopianDate,
      ethiopianStartDate,
      ethiopianEndDate,
      showGregorian,
      showEthiopian,
      displayValue,
      toggleDatePicker,
      onGregorianDateChange,
      onEthiopianRangeStart,
      onEthiopianRangeEnd,
      confirmSelection,
      cancelSelection
    }
  }
}
</script>

<style scoped>
.dual-date-picker {
  position: relative;
  width: 100%;
  max-width: 400px;
}

.date-input {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  cursor: pointer;
  background-color: white;
  color: #333;
}

.date-picker-modal {
  position: absolute;
  top: 100%;
  left: 0;
  z-index: 1000;
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  padding: 1rem;
  margin-top: 4px;
}

.date-picker-container {
  display: flex;
  gap: 1rem;
}

.picker-section {
  flex: 1;
}

.picker-section h3 {
  margin-bottom: 1rem;
  color: #333;
}

.picker-controls {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #eee;
}

button {
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
  border: none;
}

.confirm-btn {
  background-color: #2427eb;
  color: white;
}

.cancel-btn {
  background-color: #f1efef;
  color: #333;
}
</style>
