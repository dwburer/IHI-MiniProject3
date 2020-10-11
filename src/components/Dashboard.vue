<template>
  <v-container>
    <v-row>
      <v-col>
        <v-card
          outlined
        >
          <v-card-text>
            <v-text-field
              v-model="patientId"
              class="align-baseline"
              label="Patient ID"
              outlined
              hide-details="true"
            >
              <template v-slot:append-outer class="my-0">
                <v-btn
                  v-on:click="getVitals"
                  color="primary"
                  dark
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
              <v-select
                :items="vitalCategories"
                label="Select vitals to visualize"
                v-model="selectedVitals"
                multiple
                chips
              ></v-select>
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

  export default {
    name: 'Dashboard',
    components: { LineChart },
    data () {
      return {
        patientId: '46d3785e-865b-46a5-8ea9-86a72a183de3',
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
      getVitals() {
        this.loading = true
        axios
          .get(`https://launch.smarthealthit.org/v/r4/fhir/Observation?patient=${this.patientId}&category=vital-signs`)
          .then(response => {
            const vitalDictionary = response.data.entry.reduce((hash, e) => {
                e = e.resource;

                const observations = [];

                console.log(e)

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

            this.loaded = true
            this.vitals = vitalDictionary
          })
          .catch(error => {
            console.log(error)
            this.errored = true
          })
          .finally(() => this.loading = false)
      }
    },
    mounted () {
      axios
        .get('https://api.coindesk.com/v1/bpi/currentprice.json')
        .then(response => {
          this.info = response.data.bpi
        })
        .catch(error => {
          console.log(error)
          this.errored = true
        })
        .finally(() => this.loading = false)
    }
  }
</script>
