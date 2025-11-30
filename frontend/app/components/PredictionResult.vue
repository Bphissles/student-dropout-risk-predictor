<script setup>
defineProps({
  result: {
    type: Object,
    required: true
  }
})

const getRiskBadgeClass = (prediction) => {
  if (prediction === 'Dropout') return 'bg-danger'
  if (prediction === 'Enrolled') return 'bg-warning text-dark'
  return 'bg-success'
}
</script>

<template>
  <div class="mt-5 p-4 bg-light border rounded text-center animate__animated animate__fadeIn">
    <h3 class="mb-3">Prediction Result</h3>
    <div class="display-1 mb-3">
      <span class="badge rounded-pill" :class="getRiskBadgeClass(result.prediction)">
        {{ result.prediction }}
      </span>
    </div>
    <p class="lead">
      Confidence: <strong>{{ (result.confidence * 100).toFixed(1) }}%</strong>
    </p>
    
    <div class="row justify-content-center mt-4">
      <div class="col-md-6">
        <h5 class="h4">Probability Distribution</h5>
        <div v-for="(prob, label) in result.probabilities" :key="label" class="mb-2">
          <div class="d-flex justify-content-between small mb-1">
            <span>{{ label }}</span>
            <span>{{ (prob * 100).toFixed(1) }}%</span>
          </div>
          <div class="progress" style="height: 10px;">
            <div 
              class="progress-bar" 
              :class="getRiskBadgeClass(label)" 
              role="progressbar" 
              :style="{ width: prob * 100 + '%' }"
            ></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
