<template>
  <div class="widget_container fr-grid-row" :class="(loading)?'loading':''" :data-display="display" :id="widgetId">
    <div class="fr-warning" v-if="geoFallback">
      <div class="scheme-border">
        <span class="fr-fi-information-fill fr-px-1w fr-py-3v" aria-hidden="true"></span>
      </div>
      <p class="fr-text--sm fr-mb-0 fr-p-3v">{{ geoFallbackMsg }}</p>
    </div>
    <LeftCol :props="leftColProps"></LeftCol>
    <div class="r_col fr-col-12 fr-col-lg-9">
      <div class="chart ml-lg">
        <div class="multiline_tooltip">
          <div class="tooltip_header"></div>
          <div class="tooltip_body">
            <div class="tooltip_value">
              <span class="tooltip_dot"></span>
            </div>
          </div>
        </div>
        <canvas :id="chartId"></canvas>
        <div v-for="index in nbIndicateurs" :key="index" class="flex fr-mt-3v fr-mb-1v" :style="style">
          <span class="legende_dot" v-bind:style="{'background-color':colors[index-1]}"></span>
          <p class="fr-text--sm fr-text--bold fr-ml-1v fr-mb-0">
            {{capitalize(names[index - 1])}}
          </p>
        </div>
        <div v-for="index in periodsProps.date.length" :key="index" class="flex fr-mt-3v fr-mb-1v" :style="style">
          <span class="legende_dash_line1" v-bind:style="{'background-color': periodsProps.color}"></span>
          <span class="legende_dash_line2" v-bind:style="{'background-color': periodsProps.color}"></span>
          <p class="fr-text--sm fr-text--bold fr-ml-1v fr-mb-0">{{capitalize(periodsProps.name[index - 1])}}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import store from '@/store'
import Chart from 'chart.js'
import chroma from 'chroma-js'
import LeftCol from '@/components/LeftCol'
import { mixin } from '@/utils.js'

export default {
  name: 'PieChart',
  components: {
    LeftCol
  },
  mixins: [mixin],
  data () {
    return {
      listIndicateurs: [],
      nbIndicateurs: undefined,
      colors: [],
      colors_gradient: [],
      indicateur_data: [],
      labels: [],
      dataset: [],
      widgetId: '',
      chartId: '',
      display: '',
      periodsProps: {
        date: [],
        name: [],
        color: '#161616'
      },
      leftColProps: {
        localisation: '',
        currentValues: [],
        currentDate: '',
        names: [],
        evolcodes: [],
        evolvalues: [],
        isMap: false,
        date: '',
        colors_legend: [],
        legendDisplay: ['', '', ''],
        units: []
      },
      units: [],
      names: [],
      chart: undefined,
      loading: true,
      legendLeftMargin: 0,
      geoFallback: false,
      geoFallbackMsg: ''
    }
  },
  props: {
    indicateurs: String,
    periods: {
      type: String,
      default: null
    }
  },
  computed: {
    selectedGeoLevel () {
      return store.state.user.selectedGeoLevel
    },
    selectedGeoCode () {
      return store.state.user.selectedGeoCode
    },
    selectedGeoLabel () {
      return store.state.user.selectedGeoLabel
    },
    style () {
      return 'margin-left: ' + this.legendLeftMargin + 'px'
    }
  },
  methods: {
    async getData () {
      this.listIndicateurs = this.indicateurs.split(' ')
      this.nbIndicateurs = this.listIndicateurs.length
      const promises = []
      this.indicateur_data.length = this.nbIndicateurs
      promises.length = 0
      for (let i = 0; i < this.nbIndicateurs; i++) {
        const promise = store.dispatch('getData', this.listIndicateurs[i]).then(data => {
          this.indicateur_data[i] = data
        })
        promises.push(promise)
      }
      Promise.all(promises).then((values) => {
        this.loading = false
        this.createChart()
      })
    },

    updateData () {
      const self = this
      if (this.periods !== null) {
        this.periodsProps.date.length = 0
        this.periodsProps.name.length = 0
        const periodsList = this.periods.split(' ')
        periodsList.forEach(function (p) {
          self.periodsProps.date.push(store.state.periods[p].date)
          self.periodsProps.name.push(store.state.periods[p].nom)
        })
      }

      const listColors = ['#000091', '#007c3a', '#A558A0'].concat(chroma.brewer.Set2.reverse())
      this.colors = listColors.slice(0, this.nbIndicateurs)
      this.colors_gradient.length = 0
      this.colors.forEach(function (col) {
        self.colors_gradient.push(chroma(col).alpha(0.3).hex())
      })

      this.leftColProps.localisation = this.selectedGeoLabel

      let geolevel = this.selectedGeoLevel
      let geocode = this.selectedGeoCode

      let geoObject
      const listGeoObject = []

      this.leftColProps.names.length = 0
      this.units.length = 0
      this.names.length = 0
      this.leftColProps.currentValues.length = 0
      this.leftColProps.evolcodes.length = 0
      this.leftColProps.evolvalues.length = 0
      this.leftColProps.units.length = 0
      this.labels.length = 0
      this.dataset.length = 0

      this.geoFallback = false
      this.geoFallbackMsg = ''

      let isRegionOk = true
      let isDepOk = true
      if (geolevel === 'departements') {
        for (let i = 0; i < this.nbIndicateurs; i++) {
          geoObject = this.indicateur_data[i][geolevel].find(obj => {
            return obj.code_level === geocode
          })
          isDepOk = (isDepOk) && (typeof geoObject !== 'undefined')
        }
        if (!isDepOk) {
          const depObj = store.state.dep.find(obj => {
            return obj.value === geocode
          })
          this.leftColProps.localisation = depObj.region
          geolevel = 'regions'
          geocode = depObj.region_value
          this.geoFallback = true
          this.geoFallbackMsg = 'Affichage des résultats au niveau régional, faute de données au niveau départemental'
        }
      }

      if (geolevel === 'regions') {
        for (let i = 0; i < this.nbIndicateurs; i++) {
          geoObject = this.indicateur_data[i][geolevel].find(obj => {
            return obj.code_level === geocode
          })
          isRegionOk = (isRegionOk) && (typeof geoObject !== 'undefined')
        }
        if (!isRegionOk) {
          this.leftColProps.localisation = 'France entière'
          geolevel = 'France'
          geocode = '01'
          this.geoFallback = true
          this.geoFallbackMsg = 'L\'affichage des données est uniquement disponible au niveau national'
        }
      }

      const data = []
      data.length = 0

      for (let i = 0; i < this.nbIndicateurs; i++) {
        data[i] = []
        if (geolevel === 'France') {
          geoObject = this.indicateur_data[i].france[0]
        } else {
          geoObject = this.indicateur_data[i][geolevel].find(obj => {
            return obj.code_level === geocode
          })
        }

        this.leftColProps.names.push(this.indicateur_data[i].nom)
        this.units.push(this.indicateur_data[i].unite)
        this.leftColProps.units.push(this.indicateur_data[i].unite_short)
        this.names.push(this.indicateur_data[i].nom)
        this.leftColProps.currentValues.push(geoObject.last_value)
        this.leftColProps.evolcodes.push(geoObject.evol_color)
        this.leftColProps.evolvalues.push(geoObject.evol_percentage)

        listGeoObject.push(geoObject)
      }
      this.leftColProps.date = this.convertDateToHuman(listGeoObject[0].last_date)
      this.leftColProps.currentDate = this.convertDateToHuman(listGeoObject[0].last_date)
      this.leftColProps.colors_legend = listColors.slice(0, this.nbIndicateurs)

      // let date = []
      // geoObject.values.forEach(function (d) {
      //   date.push(d.date)
      // })
      const date = geoObject.values[geoObject.values.length - 1].date

      listGeoObject[0].values.forEach(function (d) {
        if (date === d.date) {
          self.labels.push(self.convertDateToHuman(d.date))
          data[0] = d.value

          for (let i = 1; i < self.nbIndicateurs; i++) {
            const correspondingValue = listGeoObject[i].values.find(obj => {
              return obj.date === d.date
            })

            data[i] = correspondingValue.value
          }
        }
      })

      const tmp = {
        data: data,
        borderColor: self.colors,
        backgroundColor: self.colors
      }
      this.dataset.push(tmp)
    },

    updateChart () {
      this.updateData()
      this.chart.update()
    },

    createChart () {
      const self = this

      this.updateData()

      const ctx = document.getElementById(self.chartId).getContext('2d')

      this.chart = new Chart(ctx, {
        type: 'doughnut',
        data: {
          labels: self.labels,
          datasets: self.dataset
        },
        options: {
          animation: {
            easing: 'easeInOutBack',
            duration: 0
          },
          legend: {
            display: false
          },
          tooltips: {
            displayColors: false,
            backgroundColor: '#6b6b6b',
            enabled: false,
            callbacks: {
              label: function (tooltipItems) {
                return self.dataset[0].data[tooltipItems.index] + ' ' + self.units[tooltipItems.index]
              },
              title: function (tooltipItems) {
                return self.labels[0]
              },
              labelTextColor: function (tooltipItems) {
                return self.colors[tooltipItems.index]
              }
            },
            custom: function (context) {
              // Tooltip Element
              const tooltipEl = self.$el.querySelector('.multiline_tooltip')

              // Hide if no tooltip
              const tooltipModel = context
              if (tooltipModel.opacity === 0) {
                tooltipEl.style.opacity = 0
                return
              }

              // Set caret Position
              tooltipEl.classList.remove('above', 'below', 'no-transform')
              if (tooltipModel.yAlign) {
                tooltipEl.classList.add(tooltipModel.yAlign)
              } else {
                tooltipEl.classList.add('no-transform')
              }

              function getBody (bodyItem) {
                return bodyItem.lines
              }
              // Set Text
              if (tooltipModel.body) {
                const titleLines = tooltipModel.title || []
                const bodyLines = tooltipModel.body.map(getBody)

                const divDate = self.$el.querySelector('.tooltip_header')
                divDate.innerHTML = titleLines

                const color = tooltipModel.labelTextColors[0]
                const divValue = self.$el.querySelector('.tooltip_value')

                const nodeName = self.$el.querySelector('.tooltip_dot').attributes[0].nodeName
                divValue.innerHTML = '<span ' + nodeName + '= "" class="tooltip_dot" style = "background-color:' + color + '"></span>' + ' ' + bodyLines[0]
              }

              const {
                offsetLeft: positionX,
                offsetTop: positionY
              } = self.chart.canvas

              const canvasWidth = Number(self.chart.canvas.style.width.replace(/\D/g, ''))
              const canvasHeight = Number(self.chart.canvas.style.height.replace(/\D/g, ''))
              tooltipEl.style.position = 'absolute'
              tooltipEl.style.padding = tooltipModel.padding + 'px ' + tooltipModel.padding + 'px'
              tooltipEl.style.pointerEvents = 'none'
              let tooltipX = positionX + tooltipModel.caretX + 10
              let tooltipY = positionY + tooltipModel.caretY - 18
              if (tooltipX + tooltipEl.clientWidth + self.legendLeftMargin > positionX + canvasWidth) { // tooltip disappears at the right of the canvas
                tooltipX = positionX + tooltipModel.caretX - tooltipEl.clientWidth - 10
              }
              if (tooltipY + tooltipEl.clientHeight > positionY + 0.9 * canvasHeight) { // tooltip disappears at the bottom of the canvas
                tooltipY = positionY + tooltipModel.caretY - tooltipEl.clientHeight + 18
              }
              if (tooltipX < positionX) {
                tooltipX = positionX + tooltipModel.caretX - tooltipEl.clientWidth / 2
                tooltipY = positionY + tooltipModel.caretY - tooltipEl.clientHeight - 18
              }
              tooltipEl.style.left = tooltipX + 'px'
              tooltipEl.style.top = tooltipY + 'px'
              tooltipEl.style.opacity = 1
            }
          }
        }
      })
    }
  },

  watch: {
    selectedGeoCode: function () {
      this.updateChart()
    },
    selectedGeoLevel: function () {
      this.updateChart()
    }
  },

  created () {
    this.chartId = 'myChart' + Math.floor(Math.random() * (1000))
    this.widgetId = 'widget' + Math.floor(Math.random() * (1000))
    this.getData()
  },

  mounted () {
    document.getElementById(this.widgetId).offsetWidth > 486 ? this.display = 'big' : this.display = 'small'
  }

}
</script>

<style scoped lang="scss">

  .widget_container{
    .fr-warning {
      display: flex;
      min-width: 100%;
      margin: 0 0 0.75rem;
      background-color: var(--w);
      width: 100%;
      .scheme-border {
        min-width: 2.5rem;
        background-color: #0768d5;
        display: flex;
        justify-content: center;
      }
      span {
        display: block;
        color: var(--w);
      }
      p {
        border: solid 1px #0768d5;
        width: 100%;
      }
    }
    .ml-lg {
      margin-left:0;
    }
    @media (min-width: 62em) {
      .ml-lg {
        margin-left:3rem;
      }
    }
    @media (max-width: 62em) {
      .chart .flex {
        margin-left:0!important
      }
    }
    .r_col {
      align-self:center;
      .flex{
        display: flex;
        .legende_dot{
          min-width: 1rem;
          width: 1rem;
          height: 1rem;
          border-radius: 50%;
          display: inline-block;
          margin-top: 0.25rem;
        }
        .legende_dash_line1{
          min-width: 0.4rem;
          width: 0.4rem;
          height: 0.2rem;
          border-radius: 0%;
          display: inline-block;
          margin-top: 0.6rem;
        }
        .legende_dash_line2{
          min-width: 0.4rem;
          width: 0.4rem;
          height: 0.2rem;
          border-radius: 0%;
          display: inline-block;
          margin-top: 0.6rem;
          margin-left: 0.2rem;
        }
      }
    }

    .chart canvas {
      max-width:100%;
    }
    .multiline_tooltip{
      width: 11.25rem;
      height: auto;
      background-color: white;
      position: fixed;
      z-index: 999;
      border-radius: 4px;
      box-shadow: 0 8px 16px 0 rgba(22, 22, 22, 0.12), 0 8px 16px -16px rgba(22, 22, 22, 0.32);
      text-align: left;
      pointer-events: none;
      font-size: 0.75rem;
      .tooltip_header{
        width: 100%;
        height: 30px;
        background-color: #f6f6f6;
        color:#6b6b6b;
        padding-left: 0.75rem;
        padding-bottom: 0.25rem;
        padding-top:0.25rem;
      }
      .tooltip_body{
        padding-left: 0.75rem;
        padding-right: 0.75rem;
        padding-top:0.25rem;
        .tooltip_dot{
          min-width: 0.7rem;
          width: 0.7rem;
          height: 0.7rem;
          border-radius: 50%;
          background-color: #000091;
          display: inline-block;
          margin-top: 0.25rem;
        }
        .tooltip_place{
          color:#242424;
        }
        .tooltip_value{
          color:#242424;
          font-weight: bold;
        }
      }
    }

  }

</style>
