```
import { Controller } from '@hotwired/stimulus'
import * as echarts from 'echarts'

export default class extends Controller {
  static targets = ['chart', 'chart2']
  static values = {
    data: Object
  }

  chartInstance = null
  chart2Instance = null

  async connect() {
    await this.load()
  }

  async render() {
    await this.load()
  }

  async load(params = '') {
    const data = await this.getData(params)
    this.dataValue = data

    this.renderChart1()
    this.renderChart2()
  }

  /* -------------------- CHART 1 -------------------- */

  renderChart1() {
    if (!this.hasChartTarget) {
      console.warn('[Chart1] target not found')
      return
    }

    if (this.chartInstance) {
      this.chartInstance.dispose()
    }

    this.chartInstance = echarts.init(this.chartTarget)

    this.chartInstance.setOption({
      tooltip: { trigger: 'axis' },
      legend: { bottom: 0 },
      xAxis: {
        type: 'category',
        data: this.dataValue.date
      },
      yAxis: { type: 'value' },
      series: [
        {
          name: 'Liked responses',
          type: 'bar',
          stack: 'responses',
          data: this.dataValue.likedResponses,
          itemStyle: { color: '#2bf3b6' }
        },
        {
          name: 'Disliked responses',
          type: 'bar',
          stack: 'responses',
          data: this.dataValue.dislikedResponses,
          itemStyle: { color: '#e9ecef' }
        },
        {
          name: 'Conversation per day',
          type: 'line',
          data: this.dataValue.conversationPerDay,
          itemStyle: { color: '#005B50' }
        }
      ]
    })

    window.addEventListener('resize', () => this.chartInstance.resize())
  }

  /* -------------------- CHART 2 -------------------- */

  renderChart2() {
    if (!this.hasChart2Target) return

    if (this.chart2Instance) {
      this.chart2Instance.dispose()
    }

    this.chart2Instance = echarts.init(this.chart2Target)

    this.chart2Instance.setOption({
      tooltip: { trigger: 'axis' },
      legend: { bottom: 0 },
      xAxis: {
        type: 'category',
        data: this.dataValue.date
      },
      yAxis: { type: 'value' },
      series: [
        {
          name: 'Connections per day',
          type: 'line',
          data: this.dataValue.connectionPerDay,
          itemStyle: { color: '#91a9dc' }
        },
        {
          name: 'Conversations per day',
          type: 'line',
          data: this.dataValue.conversationPerDay,
          itemStyle: { color: '#005B50' }
        }
      ]
    })

    window.addEventListener('resize', () => this.chart2Instance.resize())
  }

  /* -------------------- DATA -------------------- */

  async getData(params = '') {
    const response = await fetch('/chart_data?' + params)
    return response.json()
  }
}

```

```
<div {{ stimulus_controller('chart') }}>

  <div class="card shadow rounded-3 p-3 mb-3">
    <div data-chart-target="chart" style="height:400px"></div>
  </div>

  <div class="card shadow rounded-3 p-3">
    <div data-chart-target="chart2" style="height:400px"></div>
  </div>

</div>
```
