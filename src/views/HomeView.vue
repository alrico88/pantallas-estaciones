<template>
  <div class="min-h-screen bg-slate-900 p-6">
    <div class="max-w-7xl mx-auto">
      <header>
        <div
          class="flex flex-col md:flex-row items-center space-x-0 md:space-x-3 space-y-3 md:space-y-0 mb-7"
        >
          <div class="h-22 md:h-17 flex items-center justify-center flex-shrink-0 mr-5">
            <Logo class="logo w-full h-full" alt="Logo de Pantallas Estaciones" />
          </div>
          <div class="text-center md:text-left">
            <h1 class="text-2xl font-bold text-white leading-tight">Pantallas estaciones</h1>
            <p class="text-slate-400 text-md mt-1">Configurador pantallas estaciones ADIF</p>
          </div>
        </div>
      </header>

      <main>
        <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
          <section class="bg-slate-800 rounded-xl shadow-2xl p-4 lg:p-5 border border-slate-700">
            <StationFinder
              class="mb-4"
              v-model="formData.estacion"
              @station-selected="handleStationSelected"
              @station-cleared="handleStationCleared"
            />

            <StationInfo
              class="mb-4"
              :selected-station="selectedStation"
              :adif-data="adifData"
              :adif-status="adifStatus"
            />

            <StationForm
              :form-data="formData"
              :selected-station="selectedStation"
              :adif-data="adifData"
              :rows="boardRows"
              @form-change="handleFormChange"
            />
          </section>

          <aside class="space-y-6">
            <div class="bg-slate-800 rounded-xl shadow-2xl p-4 border border-slate-700">
              <div class="flex justify-center">
                <ResizableContainer>
                  <Gravita
                    class="gravita w-full h-full"
                    v-bind="gravitaProps"
                    :simulateClosedCheckIn="formData.simulateClosedCheckIn"
                    :simulateAlightingOnly="formData.simulateAlightingOnly"
                    :simulateAlertType="formData.simulateAlertType"
                    :simulatePlatformRouting="formData.simulatePlatformRouting"
                    :simulateAccessOpeningMargin="formData.simulateAccessOpeningMargin"
                    @data="handledata"
                    @status="handlestatus"
                    @rows="handleRows"
                  />
                  <template #controls>
                    <div
                      v-if="['arrivals', 'departures', 'alphabetical'].includes(formData.interfaz)"
                      class="flex items-center gap-2 text-xs text-slate-300"
                    >
                      <button
                        type="button"
                        aria-label="Página anterior"
                        :disabled="localPage <= 1"
                        @click="setLocalPage(localPage - 1)"
                        class="w-7 h-7 rounded-lg border border-slate-600 bg-slate-700 text-slate-300 hover:bg-slate-600 transition-colors cursor-pointer disabled:opacity-40 disabled:cursor-not-allowed"
                      >‹</button>
                      <input
                        :value="localPage"
                        min="1"
                        type="number"
                        inputmode="numeric"
                        aria-label="Página"
                        @change="updateLocalPage"
                        class="w-12 h-7 px-1 text-center bg-slate-700 border border-slate-600 rounded text-white focus:ring-1 focus:ring-dark-green focus:border-dark-green"
                      />
                      <button
                        type="button"
                        aria-label="Página siguiente"
                        @click="setLocalPage(localPage + 1)"
                        class="w-7 h-7 rounded-lg border border-slate-600 bg-slate-700 text-slate-300 hover:bg-slate-600 transition-colors cursor-pointer"
                      >›</button>
                    </div>
                  </template>
                </ResizableContainer>
              </div>

              <div class="mt-6 bg-light-green p-3 rounded-md border border-dark-green">
                <p class="text-xs text-dark-blue">
                  Proyecto no oficial ni afiliado con ADIF con propósito educacional. Esta web solo
                  permite configurar los parámetros de su sistema de información al viajero. La
                  marca y datos mostrados son propiedad de ADIF.
                </p>
              </div>
            </div>

            <div :class="{ 'opacity-50 pointer-events-none': !selectedStation }">
              <UrlSharing :url="generatedUrl" :disabled="!selectedStation" />
            </div>
          </aside>
        </div>
      </main>

      <footer class="mt-12 pb-7 text-center text-slate-500 text-sm">
        <div class="flex flex-col sm:flex-row items-center justify-center gap-1 sm:gap-2">
          <MonitorIcon />
          <div class="flex items-center gap-2">
            <span>por</span>
            <a
              href="https://x.com/mariomnts"
              target="_blank"
              class="hover:text-slate-400 transition-colors"
              rel="noopener noreferrer"
              >Mario Montes</a
            >
          </div>
          <span class="hidden sm:inline">•</span>
          <div>
            <a
              href="https://github.com/mariomnts/pantallas-estaciones"
              target="_blank"
              class="hover:text-slate-400 transition-colors"
              >Código en GitHub</a
            >
          </div>
        </div>
      </footer>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import Logo from '../components/icons/Logo.vue'
import MonitorIcon from '../components/icons/MonitorIcon.vue'
import ResizableContainer from '../components/ResizableContainer.vue'
import Gravita from '../components/Gravita.vue'
import StationFinder from '../components/StationFinder.vue'
import StationInfo from '../components/StationInfo.vue'
import StationForm from '../components/StationForm.vue'
import UrlSharing from '../components/UrlSharing.vue'
import { convertFormDataToGravitaProps, generateUrl } from '../utils/format'
import { Stations } from '../constants'
import { onMounted } from 'vue'

// Form data
const formData = ref({
  interfaz: 'departures',
  stationCode: '17000',
  traffic: ['cercanias', 'av', 'largaDistancia', 'regional'], // Default all except servicio interno
  languages: ['es', 'en'],
  showHeader: true,
  showAccess: true,
  showPlatform: true,
  showProduct: true,
  showNumber: true,
  countdown: true,
  showStops: true,
  showAllTrains: false,
  alphabeticalStations: [],
  alphabeticalStationNames: '',
  alphabeticalNetwork: '',
  alphabeticalStartStation: '',
  alphabeticalPage: 1,
  platformFilter: [], // Default none
  productFilter: [], // Changed from productFilter
  companyFilter: [], // Changed from companyFilter
  subtitle: 'station-name',
  subtitleParam: '',
  platformLocations: [],
  platformLocationRight: [],
  platformLocationLeft: [],
  platformLocationForwardLeft: [],
  platformLocationForwardRight: [],
  displayNumber: '',
  platformMode: 'platform',
  platformTrigger: 'next',
  showComposition: true,
  showObservation: true,
  platformArrangement: 'ascending',
  showPlatformSign: true,
  showPlatformPreview: true,
  showAlerts: true,
  showClosedCheckIn: true,
  showAlightingOnly: true,
  sectorizationMode: 'first_and_last',
  fontSize: 0,
  customFilter: [], // Línea de cercanías filter
  stopFilter: [], // Estaciones con parada filter
  // Simulation controls — not included in shareable URL, only used for preview
  simulateClosedCheckIn: false,
  simulateAlightingOnly: false,
  simulateAlertType: '',
  simulatePlatformRouting: false,
  simulateAccessOpeningMargin: false,
})

// Component state
const selectedStation = ref(Stations.find((s) => s.code === '17000') || null)
const adifData = ref(null)
const adifStatus = ref(null)
const boardRows = ref(null)
const localPage = ref(1)
const localPageActive = ref(false)

// Set default station on mount
onMounted(() => {
  const defaultStation = Stations.find((s) => s.code === '17000')
  if (defaultStation) {
    selectedStation.value = defaultStation
    formData.value.stationCode = defaultStation.code
    formData.value.estacion = defaultStation.name
  }
})

// Station handlers
const handleStationSelected = (station) => {
  selectedStation.value = station
  formData.value.stationCode = station.code
  boardRows.value = null
  localPage.value = 1
  localPageActive.value = false
}

const handleStationCleared = () => {
  selectedStation.value = null
  adifData.value = null
  formData.value.stationCode = ''
  boardRows.value = null
  localPage.value = 1
  localPageActive.value = false
}

const handleFormChange = (newFormData) => {
  const interfaceChanged = newFormData.interfaz !== formData.value.interfaz
  const alphabeticalStartStationChanged = newFormData.alphabeticalStartStation !== formData.value.alphabeticalStartStation
  if (interfaceChanged || alphabeticalStartStationChanged) {
    localPage.value = 1
    localPageActive.value = false
    newFormData.alphabeticalPage = 1
  }
  formData.value = newFormData
}

const handleRows = (rows) => {
  boardRows.value = rows
}

const setLocalPage = (page) => {
  localPage.value = Math.max(1, Number.parseInt(String(page), 10) || 1)
  if (formData.value.interfaz === 'alphabetical') {
    formData.value = { ...formData.value, alphabeticalPage: localPage.value }
    localPageActive.value = false
  } else {
    localPageActive.value = true
  }
}

const updateLocalPage = (event) => setLocalPage(event.target.value)

const handledata = (data) => {
  adifData.value = data
}

const handlestatus = (status) => {
  adifStatus.value = status
}

// Computed props for Gravita component
const gravitaProps = computed(() => {
  const props = convertFormDataToGravitaProps(formData.value)
  if (localPageActive.value && ['arrivals', 'departures'].includes(formData.value.interfaz) && boardRows.value) {
    props.startTrain = 1 + (localPage.value - 1) * boardRows.value
  }
  if (formData.value.interfaz === 'alphabetical' && boardRows.value) {
    props.alphabeticalPageRows = boardRows.value
  }
  return props
})

// URL generation
const generatedUrl = computed(() => {
  // Only generate URL if a station is selected
  if (!selectedStation.value) return 'Selecciona una estación...'
  return generateUrl(formData.value, selectedStation.value)
})

watch(
  formData,
  () => {
    window?.gtag?.('event', 'generation', {
      ...formData.value,
    })
  },
  { deep: true },
)
</script>

<style scoped>
.gravita {
  background: var(--color-blue);
}

.logo {
  stroke: var(--color-light-green);
  color: var(--color-light-green);
  fill: none;
}
</style>
