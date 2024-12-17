<template>
  <div class="app">
    <h1>Date Picker Demo</h1>
    
    <div class="picker-type-controls">
      <label>
        <input type="radio" v-model="datePickerType" value="gregorian"> Gregorian
      </label>
      <label>
        <input type="radio" v-model="datePickerType" value="ethiopian"> Ethiopian
      </label>
      <label>
        <input type="radio" v-model="datePickerType" value="both"> Both
      </label>
    </div>

    <div class="picker-container">
      <DualDatePicker
        v-model="selectedDate"
        :date-picker-type="datePickerType"
      />
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import DualDatePicker from './components/DualDatePicker.vue'

export default {
  name: 'App',
  components: {
    DualDatePicker
  },
  setup() {
    const datePickerType = ref('both')
    const selectedDate = ref(new Date())

    return {
      datePickerType,
      selectedDate
    }
  }
}
</script>

<style>
.app {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

h1 {
  color: #333;
  margin-bottom: 2rem;
}

.picker-type-controls {
  margin-bottom: 2rem;
  display: flex;
  gap: 1rem;
}

.picker-container {
  width: 100%;
}
</style>