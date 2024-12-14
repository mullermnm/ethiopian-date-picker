<template>
  <div class="ethiopian-calendar">
    <div class="calendar-header">
      <button @click="previousMonth">&lt;&lt;</button>
      <span>{{ currentMonthName }} {{ currentYear }}</span>
      <button @click="nextMonth">&gt;&gt;</button>
    </div>
    
    <div class="weekdays">
      <div v-for="day in weekDays" :key="day" class="weekday">{{ day }}</div>
    </div>
    
    <div class="days">
      <div
        v-for="day in days"
        :key="day.date"
        class="day"
        :class="{
          'current': day.isCurrentDay,
          'selected': isSelected(day),
          'range': isInRange(day),
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
import { ref, computed, onMounted, watch } from 'vue'
import { EthDateTime } from 'ethiopian-calendar-date-converter'

export default {
  name: 'EthiopianCalendar',
  props: {
    modelValue: {
      type: [Date, Array],
      default: null
    },
    range: {
      type: Boolean,
      default: false
    }
  },
  emits: ['update:modelValue', 'range-start', 'range-end'],
  setup(props, { emit }) {
    const currentDate = ref(new Date())
    const selectedDates = ref([])

    // Watch for changes in modelValue prop
    watch(() => props.modelValue, (newValue) => {
      if (Array.isArray(newValue)) {
        selectedDates.value = newValue.filter(date => date !== null).map(date => new Date(date))
      } else if (newValue) {
        selectedDates.value = [new Date(newValue)]
      } else {
        selectedDates.value = []
      }
    }, { immediate: true, deep: true })
    
    const weekDays = ['እሁድ', 'ሰኞ', 'ማክሰኞ', 'ረቡዕ', 'ሐሙስ', 'አርብ', 'ቅዳሜ']
    const monthNames = [
      'መስከረም', 'ጥቅምት', 'ህዳር', 'ታህሳስ',
      'ጥር', 'የካቲት', 'መጋቢት', 'ሚያዚያ',
      'ግንቦት', 'ሰኔ', 'ሐምሌ', 'ነሐሴ', 'ጷጉሜን'
    ]

    const currentMonthName = computed(() => {
      const ethDate = EthDateTime.fromEuropeanDate(currentDate.value)
      return monthNames[ethDate.month - 1]
    })

    const currentYear = computed(() => {
      const ethDate = EthDateTime.fromEuropeanDate(currentDate.value)
      return ethDate.year
    })

    const days = computed(() => {
      const ethDate = EthDateTime.fromEuropeanDate(currentDate.value)
      const daysInMonth = ethDate.month === 13 ? (isLeapYear(ethDate.year) ? 6 : 5) : 30
      const firstDayOfMonth = new EthDateTime(ethDate.year, ethDate.month, 1).toEuropeanDate()
      const firstDayWeekday = firstDayOfMonth.getDay()
      
      const days = []
      
      // Previous month days
      const prevMonthDays = firstDayWeekday
      for (let i = prevMonthDays - 1; i >= 0; i--) {
        const date = new Date(firstDayOfMonth)
        date.setDate(firstDayOfMonth.getDate() - i)
        days.push({
          date,
          dayOfMonth: EthDateTime.fromEuropeanDate(date).date,
          isCurrentMonth: false,
          isCurrentDay: false
        })
      }
      
      // Current month days
      for (let i = 1; i <= daysInMonth; i++) {
        const date = new EthDateTime(ethDate.year, ethDate.month, i).toEuropeanDate()
        days.push({
          date,
          dayOfMonth: i,
          isCurrentMonth: true,
          isCurrentDay: i === ethDate.date
        })
      }
      
      // Next month days
      const remainingDays = 42 - days.length
      for (let i = 1; i <= remainingDays; i++) {
        const date = new Date(firstDayOfMonth)
        date.setDate(firstDayOfMonth.getDate() + daysInMonth + i - 1)
        days.push({
          date,
          dayOfMonth: EthDateTime.fromEuropeanDate(date).date,
          isCurrentMonth: false,
          isCurrentDay: false
        })
      }
      
      return days
    })

    const isLeapYear = (year) => {
      return (year % 4 === 0 && year % 100 !== 0) || (year % 400 === 0)
    }

    const isSelected = (day) => {
      return selectedDates.value.some(date => 
        date && day.date && date.getTime() === day.date.getTime()
      )
    }

    const isInRange = (day) => {
      if (!props.range || selectedDates.value.length !== 2) return false
      const [start, end] = selectedDates.value
      const dayTime = day.date.getTime()
      return start && end && dayTime >= start.getTime() && dayTime <= end.getTime()
    }

    const isRangeStart = (day) => {
      if (!props.range || !selectedDates.value.length) return false
      const start = selectedDates.value[0]
      return start && day.date && day.date.getTime() === start.getTime()
    }

    const isRangeEnd = (day) => {
      if (!props.range || selectedDates.value.length !== 2) return false
      const end = selectedDates.value[1]
      return end && day.date && day.date.getTime() === end.getTime()
    }

    const selectDate = (day) => {
      if (!props.range) {
        selectedDates.value = [day.date]
        emit('update:modelValue', day.date)
      } else {
        if (selectedDates.value.length === 0 || selectedDates.value.length === 2) {
          // Start new range
          selectedDates.value = [day.date]
          emit('range-start', day.date)
          emit('update:modelValue', [day.date])
        } else {
          // Complete range
          const startDate = selectedDates.value[0]
          if (day.date < startDate) {
            // If end date is before start date, swap them
            selectedDates.value = [day.date, startDate]
            emit('range-start', day.date)
            emit('range-end', startDate)
            emit('update:modelValue', [day.date, startDate])
          } else {
            selectedDates.value = [startDate, day.date]
            emit('range-end', day.date)
            emit('update:modelValue', [startDate, day.date])
          }
        }
      }
    }

    const previousMonth = () => {
      const date = new Date(currentDate.value)
      date.setMonth(date.getMonth() - 1)
      currentDate.value = date
    }

    const nextMonth = () => {
      const date = new Date(currentDate.value)
      date.setMonth(date.getMonth() + 1)
      currentDate.value = date
    }

    onMounted(() => {
      if (props.modelValue) {
        if (Array.isArray(props.modelValue)) {
          selectedDates.value = props.modelValue
            .filter(date => date !== null)
            .map(date => new Date(date))
        } else {
          selectedDates.value = [new Date(props.modelValue)]
        }
      }
    })

    return {
      weekDays,
      currentMonthName,
      currentYear,
      days,
      isSelected,
      isInRange,
      isRangeStart,
      isRangeEnd,
      selectDate,
      previousMonth,
      nextMonth
    }
  }
}
</script>

<style scoped>
.ethiopian-calendar {
  width: 300px;
  border: 1px solid #ddd;
  padding: 1rem;
  background-color: #fff;
  color: black;
  font-family: system-ui, -apple-system, sans-serif;
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.calendar-header button {
  background: none;
  color: black;
  border: none;
  cursor: pointer;
  padding: 0.5rem;
}

.weekdays {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  text-align: center;
  margin-bottom: 0.5rem;
  color: black;
}

.weekday {
  padding: 0.5rem;
  font-weight: bold;
}

.days {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 1px;
}

.day {
  padding: 0.5rem;
  text-align: center;
  cursor: pointer;
  border-radius: 4px;
}

.day:hover {
  background-color: #f0f0f0;
}

.day.current {
  background-color: #e6f7ff;
}

.day.selected {
  background-color: #2427eb;
  color: white;
}

.day.range {
  background-color: #f1efef;
}

.day.range-start {
  background-color: #2427eb;
  color: white;
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}

.day.range-end {
  background-color: #2427eb;
  color: white;
  border-top-left-radius: 0;
  border-bottom-left-radius: 0;
}

.day.disabled {
  color: #ccc;
  cursor: not-allowed;
}
</style>
