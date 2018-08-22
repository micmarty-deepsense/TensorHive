<template>
  <div>
    <button v-on:click="addWatch">Add watch</button>
    <div class="watch_table" >
      <WatchBox
        class="watch_box"
        v-for="watch in watches"
        :key="watch.index"
        :resources-indexes="resourcesIndexes"
        :chart-datasets="chartDatasets"
        :update-chart="updateChart"
        :time="time"
      />
    </div>
  </div>
</template>

<script>
import WatchBox from './WatchBox.vue'
import api from '../../../api'
import _ from 'lodash'
export default {
  components: {
    WatchBox
  },

  data () {
    return {
      watches: [],
      chartDatasets: {},
      time: 5000,
      chartLength: 25,
      space: 2,
      interval: null,
      errors: [],
      updateChart: false,
      uniqueMetrics: {},
      resourcesIndexes: {}
    }
  },

  created () {
    this.loadData()
    let self = this
    this.interval = setInterval(function () {
      self.changeData()
    }, this.time)
  },

  methods: {
    setColor: function (node) {
      var color = '#123456'
      var step = node * 123456
      var colorToInt = parseInt(color.substr(1), 16)
      var nstep = parseInt(step)
      if (!isNaN(colorToInt) && !isNaN(nstep)) {
        colorToInt += nstep
        var ncolor = colorToInt.toString(16)
        ncolor = '#' + (new Array(7 - ncolor.length).join(0)) + ncolor
        if (/^#[0-9a-f]{6}$/i.test(ncolor)) {
          return ncolor
        }
      }
      return color
    },

    loadData: function () {
      api
        .request('get', '/nodes/metrics')
        .then(response => {
          this.watches = [
            {
              index: 0
            }
          ]
          this.parseData(response.data)
        })
        .catch(e => {
          this.errors.push(e)
        })
    },

    isVisible: function (metric, metricName) {
      if (metric.value === null) {
        return false
      } else {
        if (metricName === 'mem_total') return false
        return true
      }
    },

    parseData: function (apiResponse) {
      var node, resourceType, metrics, resourceTypes
      for (var nodeName in apiResponse) {
        resourceTypes = {}
        node = apiResponse[nodeName]
        if (node !== null) {
          for (var resourceTypeName in node) {
            resourceType = node[resourceTypeName]
            if (resourceType !== null) {
              metrics = this.findMetrics(resourceType)
              resourceTypes[resourceTypeName] = {
                metrics: metrics,
                uniqueMetricNames: this.uniqueMetrics
              }
            }
          }
        }
        this.chartDatasets[nodeName] = resourceTypes
      }
    },

    findMetrics: function (resourceType) {
      var resource, metric, tempMetrics
      this.uniqueMetrics = {}
      tempMetrics = {}
      for (var resourceUUID in resourceType) {
        this.resourcesIndexes[resourceUUID] = resourceType[resourceUUID].index
        resource = resourceType[resourceUUID]
        for (var metricName in resource.metrics) {
          if (isNaN(resource.metrics[metricName])) {
            metric = resource.metrics[metricName]
            metric['visible'] = this.isVisible(resource.metrics[metricName], metricName)
          } else {
            metric = {
              value: resource.metrics[metricName],
              unit: '',
              visible: this.isVisible(resource.metrics[metricName], metricName)
            }
          }
          if (this.uniqueMetrics.hasOwnProperty(metricName)) {
            if (this.uniqueMetrics[metricName].visible === false) {
              this.uniqueMetrics[metricName] = metric
            }
          } else {
            this.uniqueMetrics[metricName] = metric
          }
        }
      }
      for (var uniqueMetricName in this.uniqueMetrics) {
        if (this.uniqueMetrics[uniqueMetricName].visible === true) {
          tempMetrics[uniqueMetricName] = this.createMetric(resourceType, uniqueMetricName)
        }
      }
      return tempMetrics
    },

    createMetric: function (resourceType, metricName) {
      var labels, totalMemory, value, datasets, orderedDatasets
      labels = []
      for (var i = (this.chartLength - 1) * this.time / 1000; i >= 0; i -= this.time / 1000) {
        if (i % ((this.space + 1) * this.time / 1000) === 0) {
          labels.push(i)
        } else {
          labels.push('')
        }
      }
      datasets = []
      for (var resourceUUID in resourceType) {
        if (resourceType[resourceUUID].metrics[metricName] !== null) {
          value = isNaN(resourceType[resourceUUID].metrics[metricName]) ? resourceType[resourceUUID].metrics[metricName].value : resourceType[resourceUUID].metrics[metricName]
          totalMemory = resourceType[resourceUUID].metrics['mem_total'].value
          datasets.push(
            this.createDataset(
              resourceUUID,
              'GPU' + resourceType[resourceUUID].index,
              this.setColor(resourceType[resourceUUID].index + 1),
              value
            )
          )
        }
      }
      orderedDatasets = _.orderBy(datasets, 'label')
      var obj = {
        metricName: metricName,
        data: {
          labels: labels,
          datasets: orderedDatasets
        },
        options: this.createOptions(totalMemory, metricName)
      }
      return obj
    },

    createDataset: function (uuid, label, color, data) {
      var defaultData = []
      for (var i = 0; i < this.chartLength - 1; i++) {
        defaultData.push(0)
      }
      if (data !== null) {
        defaultData.push(data)
      } else {
        defaultData.push(-1)
      }
      var obj = {
        uuid: uuid,
        label: label,
        fill: true,
        borderColor: color,
        pointBackgroundColor: color,
        backgroundColor: 'rgba(0, 0, 0, 0)',
        data: defaultData
      }
      return obj
    },

    createOptions: function (totalMemory, metricName) {
      var obj = {
        responsive: true,
        maintainAspectRatio: false,
        legend: {
          position: 'bottom',
          display: true
        },
        tooltips: {
          mode: 'label',
          xPadding: 10,
          yPadding: 10,
          bodySpacing: 10
        },
        scales: {
          xAxes: [{
            scaleLabel: {
              display: true,
              labelString: 'seconds ago'
            }
          }],
          yAxes: [{
            id: 'y-axis-1',
            type: 'linear',
            position: 'left',
            scaleLabel: {
              display: true,
              labelString: ''
            }
          }]
        }
      }
      obj['scales']['yAxes'][0]['scaleLabel']['labelString'] = this.uniqueMetrics[metricName].unit
      if (metricName === 'mem_util' || metricName === 'gpu_util') {
        obj['scales']['yAxes'][0]['ticks'] = {
          suggestedMin: 0,
          max: 100
        }
      }
      if (metricName === 'mem_used' || metricName === 'mem_free') {
        obj['scales']['yAxes'][0]['ticks'] = {
          suggestedMin: 0,
          suggestedMax: totalMemory
        }
      }
      return obj
    },

    changeData: function () {
      var node, metric, resourceType, value
      var data = []
      for (var nodeName in this.chartDatasets) {
        node = this.chartDatasets[nodeName]
        api
          .request('get', '/nodes/' + nodeName + '/gpu/metrics')
          .then(response => {
            data = response.data
            for (var resourceTypeName in node) {
              resourceType = node[resourceTypeName]
              for (var metricName in resourceType.metrics) {
                metric = resourceType.metrics[metricName]
                for (var i = 0; i < metric.data.datasets.length; i++) {
                  value = isNaN(data[metric.data.datasets[i].uuid][metric.metricName])
                    ? data[metric.data.datasets[i].uuid][metric.metricName].value
                    : data[metric.data.datasets[i].uuid][metric.metricName]
                  metric.data.datasets[i].data.shift()
                  metric.data.datasets[i].data.push(value)
                }
              }
            }
            this.updateChart = !(this.updateChart)
          })
          .catch(e => {
            this.errors.push(e)
          })
      }
    },

    addWatch: function () {
      this.watches.push({ index: this.watches.length })
    }
  }
}
</script>

<style>
.watch_table{
  display: flex;
  flex-wrap: wrap;
}
.watch_box{
  height: 40vh;
  width: 25vw;
  margin-left: 3vh;
}
</style>