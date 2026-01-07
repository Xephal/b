```
import { Controller } from '@hotwired/stimulus'
import * as echarts from 'echarts'

export default class extends Controller {

  static values = {
    data: Object
  }

  async connect() {
    await this.load()
  }

  async load(params = '') {
    const data = await this.getData(params)
    this.dataValue = data

    this.renderChart1()
    this.renderChart2()
  }

  /* -------------------- CHART 1 -------------------- */
  // Ticket-compliant:
  // - Bar: active users
  // - Line: messages
  // - Line: avg response time

  renderChart1() {
    const el = document.getElementById('chart')
    if (!el) {
      console.warn('[Chart1] DOM element #chart not found')
      return
    }

    // IMPORTANT: éviter "already initialized"
    if (echarts.getInstanceByDom(el)) {
      echarts.dispose(el)
    }

    const chart = echarts.init(el)
    window.addEventListener('resize', () => chart.resize())

    const d = this.dataValue

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: {
        type: 'scroll',
        bottom: 0
      },
      xAxis: {
        type: 'category',
        data: d.date
      },
      yAxis: [
        {
          type: 'value',
          name: 'Count'
        },
        {
          type: 'value',
          name: 'Avg response time',
          position: 'right'
        }
      ],
      series: [
        {
          name: 'Active users',
          type: 'bar',
          data: d.activeUsersPerDay
        },
        {
          name: 'Messages',
          type: 'line',
          data: d.messagesPerDay
        },
        {
          name: 'Avg response time',
          type: 'line',
          yAxisIndex: 1,
          data: d.avgResponseTimePerDay
        }
      ]
    })
  }

  /* -------------------- CHART 2 -------------------- */
  // Conservée telle quelle

  renderChart2() {
    const el = document.getElementById('chart2')
    if (!el) {
      console.warn('[Chart2] DOM element #chart2 not found')
      return
    }

    if (echarts.getInstanceByDom(el)) {
      echarts.dispose(el)
    }

    const chart = echarts.init(el)
    window.addEventListener('resize', () => chart.resize())

    const d = this.dataValue

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: {
        type: 'scroll',
        bottom: 0
      },
      xAxis: {
        type: 'category',
        data: d.date
      },
      yAxis: {
        type: 'value'
      },
      series: [
        {
          name: 'Connections per day',
          type: 'line',
          data: d.connectionPerDay
        },
        {
          name: 'Conversations per day',
          type: 'line',
          data: d.conversationPerDay
        }
      ]
    })
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
