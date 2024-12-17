<template>
  <div class="ethiopian-calendar">
    <div class="calendar-header">
      <button @click="previousMonth">&lt;</button>
      <div class="month-year">
        <div @click="showMonthSelector = !showMonthSelector" class="month-selector">
          {{ currentMonthName }}
          <div v-if="showMonthSelector" class="selector-dropdown">
            <div v-for="(month, index) in monthNames" 
                 :key="index" 
                 @click="selectMonth(index + 1)"
                 :class="{ active: index + 1 === currentMonth }">
              {{ month }}
            </div>
          </div>
        </div>
        <div @click="showYearSelector = !showYearSelector" class="year-selector">
          {{ currentYear }}
          <div v-if="showYearSelector" class="selector-dropdown">
            <div v-for="year in yearRange" 
                 :key="year" 
                 @click="selectYear(year)"
                 :class="{ active: year === currentYear }">
              {{ year }}
            </div>
          </div>
        </div>
      </div>
      <button @click="nextMonth">&gt;</button>
    </div>
    <div class="weekdays">
      <div v-for="day in weekDays" :key="day" class="weekday">{{ day }}</div>
    </div>
    <div class="days">
      <div
        v-for="day in days"
        :key="day.date ? day.date.year + day.date.month + day.date.day : ''"
        class="day"
        :class="{
          'current': day.isCurrentDay,
          'selected': isSelected(day),
          'in-range': isInRange(day),
          'range-start': isRangeStart(day),
          'range-end': isRangeEnd(day),
          'disabled': !day.isCurrentMonth
        }"
        @click="selectDate(day)"
      >
        {{ day.dayOfMonth }}
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'

export default {
  name: 'EthiopianCalendar',
  props: {
    modelValue: {
      type: Object,
      default: null
    },
    startDate: {
      type: Object,
      default: null
    },
    endDate: {
      type: Object,
      default: null
    },
    isRange: {
      type: Boolean,
      default: false
    }
  },
  emits: ['update:modelValue', 'range-start', 'range-end'],
  setup(props, { emit }) {
    // Convert today's Gregorian date to Ethiopian date
    const getEthiopianToday = () => {
      const today = new Date()
      const gregYear = today.getFullYear()
      const gregMonth = today.getMonth() + 1
      const gregDay = today.getDate()

      // Ethiopian calendar is approximately 7-8 years behind Gregorian
      let ethYear = gregYear - 7
      let ethMonth
      let ethDay

      // Ethiopian new year starts on September 11/12
      if (gregMonth >= 9) {
        ethMonth = gregMonth - 8
        if (gregDay >= 11) {
          ethDay = gregDay - 10
        } else {
          ethMonth--
          ethDay = gregDay + 20
        }
      } else {
        ethMonth = gregMonth + 4
        if (gregDay >= 11) {
          ethDay = gregDay - 10
        } else {
          ethMonth--
          ethDay = gregDay + 20
        }
      }

      // Adjust for edge cases
      if (ethMonth <= 0) {
        ethMonth = 13
        ethYear--
      }
      if (ethMonth > 13) {
        ethMonth = 1
        ethYear++
      }

      return { year: ethYear, month: ethMonth, day: ethDay }
    }

    const today = getEthiopianToday()
    const currentDate = ref(today)
    const selectedDate = ref(props.modelValue || today)
    const showMonthSelector = ref(false)
    const showYearSelector = ref(false)

    const weekDays = ['እሁድ', 'ሰኞ', 'ማክሰኞ', 'ረቡዕ', 'ሐሙስ', 'አርብ', 'ቅዳሜ']
    const monthNames = [
      'መስከረም', 'ጥቅምት', 'ህዳር', 'ታህሳስ',
      'ጥር', 'የካቲት', 'መጋቢት', 'ሚያዚያ',
      'ግንቦት', 'ሰኔ', 'ሐምሌ', 'ነሐሴ', 'ጷጉሜን'
    ]

    const currentMonth = computed(() => currentDate.value.month)
    const currentMonthName = computed(() => monthNames[currentMonth.value - 1])
    const currentYear = computed(() => currentDate.value.year)

    const yearRange = computed(() => {
      const year = currentYear.value
      return Array.from({ length: 20 }, (_, i) => year - 10 + i)
    })

    const isLeapYear = (year) => {
      return (year % 4 === 0 && year % 100 !== 0) || (year % 400 === 0)
    }

    const days = computed(() => {
      const { year, month } = currentDate.value
      const daysInMonth = month === 13 ? (isLeapYear(year + 8) ? 6 : 5) : 30
      const firstDayOfMonth = getFirstDayOfMonth(year, month)
      
      const days = []
      
      // Fill with empty days for the first week
      for (let i = 0; i < firstDayOfMonth; i++) {
        days.push({
          date: null,
          dayOfMonth: '',
          isCurrentMonth: false,
          isCurrentDay: false
        })
      }
      
      // Current month days
      for (let i = 1; i <= daysInMonth; i++) {
        days.push({
          date: { year, month, day: i },
          dayOfMonth: i,
          isCurrentMonth: true,
          isCurrentDay: false
        })
      }
      
      // Fill remaining days of the last week
      const remainingDays = 7 - (days.length % 7)
      if (remainingDays < 7) {
        for (let i = 0; i < remainingDays; i++) {
          days.push({
            date: null,
            dayOfMonth: '',
            isCurrentMonth: false,
            isCurrentDay: false
          })
        }
      }
      
      return days
    })

    const getFirstDayOfMonth = (year, month) => {
      // Simplified calculation for Ethiopian calendar
      const baseDay = 2 // Ethiopian calendar typically starts on what would be Monday in Gregorian
      const totalMonths = (year * 13) + month
      return (baseDay + totalMonths) % 7
    }

    const isSameDate = (date1, date2) => {
      if (!date1 || !date2) return false
      return date1.year === date2.year && 
             date1.month === date2.month && 
             date1.day === date2.day
    }

    const isDateBefore = (date1, date2) => {
      if (!date1 || !date2) return false
      if (date1.year !== date2.year) return date1.year < date2.year
      if (date1.month !== date2.month) return date1.month < date2.month
      return date1.day < date2.day
    }

    const isDateAfter = (date1, date2) => {
      if (!date1 || !date2) return false
      if (date1.year !== date2.year) return date1.year > date2.year
      if (date1.month !== date2.month) return date1.month > date2.month
      return date1.day > date2.day
    }

    const isSelected = (day) => {
      return isSameDate(day.date, selectedDate.value)
    }

    const isRangeStart = (day) => {
      return isSameDate(day.date, props.startDate)
    }

    const isRangeEnd = (day) => {
      return isSameDate(day.date, props.endDate)
    }

    const isInRange = (day) => {
      if (!props.startDate || !props.endDate || !day.date) return false
      return isDateAfter(day.date, props.startDate) && 
             isDateBefore(day.date, props.endDate)
    }

    const selectDate = (day) => {
      if (!day.isCurrentMonth || !day.date) return
      
      if (props.isRange) {
        if (!props.startDate || (props.startDate && props.endDate)) {
          // Start new range
          emit('range-start', day.date)
          emit('range-end', null)
        } else {
          // Complete or update range
          if (isDateBefore(day.date, props.startDate)) {
            // If selected date is before start date, make it the new start
            emit('range-start', day.date)
          } else {
            emit('range-end', day.date)
          }
        }
      } else {
        selectedDate.value = day.date
        emit('update:modelValue', day.date)
      }
      
      showMonthSelector.value = false
      showYearSelector.value = false
    }

    const selectMonth = (month) => {
      currentDate.value = { ...currentDate.value, month }
      showMonthSelector.value = false
    }

    const selectYear = (year) => {
      currentDate.value = { ...currentDate.value, year }
      showYearSelector.value = false
    }

    const previousMonth = () => {
      const { year, month } = currentDate.value
      if (month === 1) {
        currentDate.value = { year: year - 1, month: 13, day: 1 }
      } else {
        currentDate.value = { year, month: month - 1, day: 1 }
      }
    }

    const nextMonth = () => {
      const { year, month } = currentDate.value
      if (month === 13) {
        currentDate.value = { year: year + 1, month: 1, day: 1 }
      } else {
        currentDate.value = { year, month: month + 1, day: 1 }
      }
    }

    watch(() => props.modelValue, (newValue) => {
      if (newValue) {
        selectedDate.value = newValue
        currentDate.value = { ...newValue }
      }
    })

    onMounted(() => {
      if (props.modelValue) {
        selectedDate.value = props.modelValue
        currentDate.value = { ...props.modelValue }
      }
    })

    return {
      currentDate,
      selectedDate,
      weekDays,
      monthNames,
      currentMonth,
      currentMonthName,
      currentYear,
      yearRange,
      days,
      showMonthSelector,
      showYearSelector,
      isSelected,
      isRangeStart,
      isRangeEnd,
      isInRange,
      selectDate,
      selectMonth,
      selectYear,
      previousMonth,
      nextMonth
    }
  }
}
</script>

<style scoped>
.ethiopian-calendar {
  width: 300px;
  padding: 1rem;
  font-family: Arial, sans-serif;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1)
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  color: #666;
}

.month-year {
  display: flex;
  gap: 0.5rem;
  align-items: center;
  position: relative;
}

.month-selector,
.year-selector {
  cursor: pointer;
  padding: 0.25rem 2rem;
  position: relative;
  color: #666;
  
}

.selector-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  background: white;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  z-index: 1000;
  max-height: 200px;
  overflow-y: auto;
}

.selector-dropdown div {
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.selector-dropdown div:hover {
  background: #f5f5f5;
}

.selector-dropdown div.active {
  background: #e0e0e0;
}

.weekdays {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 0.5rem;
  margin-bottom: 0.5rem;
  text-align: center;
}

.weekday {
  font-weight: bold;
  color: black;
}
.calendar-header button {
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1.2rem;
  color: #666;
}

.days {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 0.5rem;
  color: #666;

}

.day {
  aspect-ratio: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border-radius: 50%;
  transition: background-color 0.2s;
}

.day:hover:not(.disabled) {
  background-color: #f0f0f0;
}

.day.disabled {
  color: #ccc;
  cursor: default;
}

.day.current {
  color: #2196f3;
  font-weight: bold;
}

.day.selected {
  background-color: #2196f3;
  color: white;
}

.day.in-range {
  background-color: #e3f2fd;
  border-radius: 0;
}

.day.range-start {
  background-color: #2196f3;
  color: white;
  border-top-left-radius: 50%;
  border-bottom-left-radius: 50%;
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}

.day.range-end {
  background-color: #2196f3;
  color: white;
  border-top-right-radius: 50%;
  border-bottom-right-radius: 50%;
  border-top-left-radius: 0;
  border-bottom-left-radius: 0;
}

button {
  padding: 0.5rem;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1.2rem;
}

button:hover {
  background-color: #f0f0f0;
  border-radius: 50%;
}
</style>
