<script>
  import { Line } from 'vue-chartjs'

  export default {
    extends: Line,
    props: {
      chartdata: {
        type: Object,
        default: null
      },
      options: {
        type: Object,
        default: function () {
          return {
            responsive: true,
            maintainAspectRatio: false,
            legend: {
              display: false,
            },
            title: {
              display: false,
            },
            scales: {
              xAxes: [{
                type: "time",
                time: {
                  format: 'DD/MM/YYYY',
                  tooltipFormat: 'll'
                },
                scaleLabel: {
                  display: true,
                  labelString: 'Date'
                }
              }],
              yAxes: [{
                scaleLabel: {
                  display: true,
                  labelString: 'value'
                }
              }]
            }
          }
        }
      }
    },
    mounted () {
      this.renderChart(this.chartdata, this.options)
    }
  }
</script>
