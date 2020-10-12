<template>
  <v-container>
    <v-row>
      <v-col>
        <v-card
          outlined
        >
          <v-card-text>
            <v-select
                :items="endpoints"
                label="Select API endpoint:"
                v-on:change="endpointChanged"
                v-model="selectedEndpoint"
                filled
              ></v-select>
            <small>
              Recommended patient IDs to try:
              <ul>
                <li v-for="p in recommendedPatients" :key="p">
                  <code>{{p}}</code> <a v-on:click="patientId = p">(choose)</a>
                </li>
              </ul>
            </small>
            <v-text-field
              v-model="patientId"
              class="align-baseline"
              label="Enter patient ID"
              outlined
            >
              <template v-slot:append-outer class="my-0">
                <v-btn
                  v-on:click="getVitals"
                  color="primary"
                  :disabled="!patientId"
                  large
                >
                  Load Vitals
                </v-btn>
              </template>
            </v-text-field>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
    <v-row class="text-center">
      <v-col cols="12">
        <v-card
          :loading="loading"
          outlined
        >
          <template slot="progress">
            <v-progress-linear
              color="deep-purple"
              height="10"
              indeterminate
            ></v-progress-linear>
          </template>
          <v-card-text>
            <template v-if="!loaded">
              <h3 class="ma-5">Vital signs overservation data not yet loaded.</h3>
            </template>
            <template v-else>
              <template v-if="errored">
                <h3 class="red--text">Data for patient "{{errorId}}" could not be loaded from "{{selectedEndpoint}}".</h3>
              </template>
              <template v-else>
                <v-select
                  :items="vitalCategories"
                  label="Select vitals to visualize"
                  v-model="selectedVitals"
                  multiple
                  chips
                ></v-select>
              </template>
            </template>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
    <v-row class="text-center">
      <v-col cols="6" v-for="vital in selectedVitals" :key="vital">
        <v-card
          outlined
        >
          <v-card-title>{{vital}}</v-card-title>
          <v-card-text>
            <div style="height: 400px">
              <line-chart
                v-if="loaded"
                :chartdata="chartSeries(vital)"
              />
            </div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
  import axios from 'axios';
  import LineChart from './Line.vue'
  import colors from 'vuetify/lib/util/colors'

  const endpoints = [
    'launch.smarthealthit.org/v/r4/fhir',
    'r4.smarthealthit.org'
  ]

  const testPatients = {
    'launch.smarthealthit.org/v/r4/fhir': [
      '46d3785e-865b-46a5-8ea9-86a72a183de3',
      '95c0165c-f900-4b50-90db-8176b4fd935d',
    ],
    'r4.smarthealthit.org': [
      '7c058fdc-75a9-4837-bedf-3bf1924311b7',
      '3ed2a74a-739a-4f67-8492-c7b94e61515f',
    ]
  }

  export default {
    name: 'Dashboard',
    components: { LineChart },
    data () {
      return {
        selectedEndpoint: endpoints[0],
        endpoints: endpoints,
        patientId: '',
        info: null,
        loaded: false,
        loading: false,
        errored: false,
        vitals: {},
        selectedVitals: [],
      }
    },
    computed: {
      vitalCategories: function () {
        return Object.keys(this.vitals)
      },
      recommendedPatients: function () {
        return testPatients[this.selectedEndpoint]
      }
    },
    filters: {
      currencydecimal (value) {
        return value.toFixed(2)
      },
    },
    methods: {
      chartSeries: function (metric) {
        const colorKeys = Object.keys(colors);
        const newColor = colorKeys[Math.floor(Math.random() * colorKeys.length)];
        const hex = colors[newColor].base;

        const r = parseInt(hex.slice(1, 3), 16),
              g = parseInt(hex.slice(3, 5), 16),
              b = parseInt(hex.slice(5, 7), 16)

        const backgroundColor = `rgba(${r}, ${g}, ${b}, 0.2)`

        return {
          datasets: [
            {
              label: metric,
              borderColor: hex,
              backgroundColor: backgroundColor,
              data: (this.vitals[metric] || []).map(e => {
                return {
                  x: new Date(e.date),
                  y: e.value
                }
              }).sort((a, b) => b.x - a.x)
            }
          ]
        }
      },
      endpointChanged() {
        this.vitals = {}
        this.selectedVitals = []
        this.loading = false
        this.loaded = false
        this.errored = false
        this.patientId = ''
      },
      getVitals() {
        this.selectedVitals = []
        this.loading = true
        this.errored = false
        axios
          .get(`https://${this.selectedEndpoint}/Observation?patient=${this.patientId}&category=vital-signs`)
          .then(response => {
            const vitalDictionary = response.data.entry.reduce((hash, e) => {
                e = e.resource;

                const observations = [];

                if ('valueQuantity' in e && 'issued' in e) {
                  observations.push({
                    name: e.code.text,
                    value: e.valueQuantity.value,
                    date: e.issued
                  })
                } else if ('component' in e) {
                  e.component.forEach(comp => {
                    observations.push({
                      name: comp.code.text,
                      value: comp.valueQuantity.value,
                      date: e.issued
                    })
                  })
                }

                observations.forEach(o => {
                  if (o.name in hash) {
                    hash[o.name].push(o);
                  } else {
                    hash[o.name] = [o]
                  }
                })

                return hash
            }, {});

            this.vitals = vitalDictionary
          })
          .catch(error => {
            console.log(error)
            this.errorId = this.patientId
            this.errored = true
          })
          .finally(() => {
            this.loading = false
            this.loaded = true
          })
      }
    }
  }
</script>
