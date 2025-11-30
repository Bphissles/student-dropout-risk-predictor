<script setup>
import { ref, reactive } from 'vue'
import PredictionForm from '~/components/PredictionForm.vue'
import PredictionResult from '~/components/PredictionResult.vue'
import BatchResults from '~/components/BatchResults.vue'

useHead({ title: 'Home | Student Dropout Risk Predictor' })

// --- State ---
const loading = ref(false)
const error = ref(null)
const success = ref(null)
const predictionResult = ref(null)
const batchResults = ref(null)

// Get runtime config
const config = useRuntimeConfig()
const API_URL = config.public.apiBase

// --- Categorical Options ---
const categoricalOptions = {
  course: {
    33: 'Biofuel Production Technologies', 171: 'Animation and Multimedia Design', 8014: 'Social Service (evening attendance)',
    9003: 'Agronomy', 9070: 'Communication Design', 9085: 'Veterinary Nursing', 9119: 'Informatics Engineering',
    9130: 'Equinculture', 9147: 'Management', 9238: 'Social Service', 9254: 'Tourism', 9500: 'Nursing',
    9556: 'Oral Hygiene', 9670: 'Advertising and Marketing Management', 9773: 'Journalism and Communication',
    9853: 'Basic Education', 9991: 'Management (evening attendance)'
  },
  yesNo: {
    1: 'Yes', 0: 'No'
  }
}

// --- Form Schema ---
// Reduced to Top 10 most predictive features
const formFields = [
  { name: 'Course', default: 171 },
  { name: 'Age at enrollment', default: 20 },
  { name: 'Tuition fees up to date', default: 1 },
  { name: 'Curricular units 1st sem (evaluations)', default: 0 },
  { name: 'Curricular units 1st sem (approved)', default: 0 },
  { name: 'Curricular units 1st sem (grade)', default: 0.0 },
  { name: 'Curricular units 2nd sem (enrolled)', default: 0 },
  { name: 'Curricular units 2nd sem (evaluations)', default: 0 },
  { name: 'Curricular units 2nd sem (approved)', default: 0 },
  { name: 'Curricular units 2nd sem (grade)', default: 0.0 }
]

// Initialize form data with defaults
const formData = reactive({})
formFields.forEach(field => {
  formData[field.name] = field.default
})

// --- Actions ---

const handleSinglePredict = async () => {
  loading.value = true
  error.value = null
  predictionResult.value = null
  
  try {
    const response = await fetch(`${API_URL}/predict`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(formData),
    })

    const data = await response.json()
    
    if (!response.ok) throw new Error(data.error || 'Prediction failed')
    
    predictionResult.value = data[0]
    success.value = 'Prediction successful!'
    
    // Scroll to result
    setTimeout(() => {
      document.getElementById('prediction-result')?.scrollIntoView({ behavior: 'smooth' })
    }, 100)
    
  } catch (e) {
    error.value = e.message
  } finally {
    loading.value = false
  }
}

const fileInput = ref(null)

const handleFileUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  loading.value = true
  error.value = null
  batchResults.value = null
  
  const formDataObj = new FormData()
  formDataObj.append('file', file)

  try {
    const response = await fetch(`${API_URL}/predict-file`, {
      method: 'POST',
      body: formDataObj,
    })

    const data = await response.json()
    
    if (!response.ok) throw new Error(data.error || 'Batch processing failed')
    
    batchResults.value = data
    success.value = `Processed ${data.length} records successfully!`
    
  } catch (e) {
    error.value = e.message
  } finally {
    loading.value = false
    // Reset file input
    if (fileInput.value) fileInput.value.value = ''
  }
}

const downloadCSV = () => {
  if (!batchResults.value || !batchResults.value.length) return
  
  const data = batchResults.value
  // Get headers
  const headers = Object.keys(data[0])
  const csvContent = [
    headers.join(','),
    ...data.map(row => headers.map(header => {
      let val = row[header]
      
      // Special formatting for probabilities column
      if (header === 'probabilities' && row.prediction && row.confidence) {
        return `${row.prediction}: ${(row.confidence * 100).toFixed(0)}%`
      }
      
      // Handle objects (like probabilities if prediction/confidence missing)
      if (typeof val === 'object' && val !== null) {
        val = JSON.stringify(val).replace(/"/g, "'") // Simplify for CSV
      }
      
      // Handle values that might contain commas
      return typeof val === 'string' && val.includes(',') ? `"${val}"` : val
    }).join(','))
  ].join('\n')
  
  const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.setAttribute('href', url)
  link.setAttribute('download', 'predictions.csv')
  link.style.visibility = 'hidden'
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}
</script>

<template>
  <div class="container py-5">
    <div class="row align-items-center mb-5">
      <div class="col-lg-8">
        <h1 class="display-5 fw-bold mb-3">Predict Student Dropout Risk</h1>
        <p class="lead mb-4">
          Utilize our Random Forest model to identify students at risk. 
          Choose between single-student prediction or batch processing via CSV.
        </p>
        <div class="d-flex gap-2 flex-wrap">
          <a href="#single-prediction" class="btn btn-primary">Start Prediction</a>
          <a href="/student_template.csv" download class="btn btn-outline-secondary">
            <i class="bi bi-download me-1"></i> Download Template CSV
          </a>
        </div>
      </div>
    </div>

    <!-- Alerts -->
    <div v-if="error" class="alert alert-danger alert-dismissible fade show" role="alert">
      {{ error }}
      <button type="button" class="btn-close" @click="error = null"></button>
    </div>
    <div v-if="success" class="alert alert-success alert-dismissible fade show" role="alert">
      {{ success }}
      <button type="button" class="btn-close" @click="success = null"></button>
    </div>

    <!-- Batch Upload Section -->
    <div class="card shadow-sm mb-5">
      <div class="card-header bg-light">
        <h2 class="h5 mb-0">Batch CSV Upload</h2>
      </div>
      <div class="card-body">
        <div class="row align-items-center">
          <div class="col-md-8">
            <p class="mb-0 text-muted">
              Upload a CSV file containing student data. The file should follow the template format.
              Results will be displayed below and can be exported.
            </p>
          </div>
          <div class="col-md-4 text-end">
             <label for="file-upload" class="visually-hidden">Upload CSV File</label>
             <input 
              id="file-upload"
              type="file" 
              ref="fileInput"
              class="form-control" 
              accept=".csv"
              @change="handleFileUpload"
              :disabled="loading"
            >
          </div>
        </div>

        <!-- Batch Results Component -->
        <BatchResults 
          v-if="batchResults" 
          :results="batchResults" 
          :categorical-options="categoricalOptions"
          @download="downloadCSV"
        />
      </div>
    </div>

    <hr class="my-5">

    <!-- Single Prediction Form -->
    <div id="single-prediction" class="card shadow-sm mb-4">
      <div class="card-header bg-primary text-white">
        <h2 class="h5 mb-0">Single Student Prediction</h2>
      </div>
      <div class="card-body">
        <PredictionForm 
          v-model="formData" 
          :loading="loading" 
          :categorical-options="categoricalOptions"
          @submit="handleSinglePredict"
        />

        <!-- Single Result Display -->
        <div v-if="predictionResult" id="prediction-result">
          <PredictionResult :result="predictionResult" />
        </div>

      </div>
    </div>
    
  </div>
</template>
