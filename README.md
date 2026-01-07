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
  // Ticket:
  // - Active users per day (bar)
  // - Messages per day (line)
  // - Avg response time per day (line)

  renderChart1() {
    const el = document.getElementById('chart')
    if (!el) {
      console.warn('[Chart1] DOM element #chart not found')
      return
    }

    const existing = echarts.getInstanceByDom(el)
    if (existing) {
      existing.dispose()
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
        { type: 'value' },
        { type: 'value' }
      ],
      series: [
        {
          name: 'Active users per day',
          type: 'bar',
          data: d.activeUsersPerDay,
          color: '#2bf3b6'
        },
        {
          name: 'Messages per day',
          type: 'line',
          yAxisIndex: 0,
          data: d.messagesPerDay,
          color: '#005B50'
        },
        {
          name: 'Avg response time',
          type: 'line',
          yAxisIndex: 1,
          data: d.avgResponseTimePerDay,
          color: '#91a9dc'
        }
      ]
    })
  }

  /* -------------------- CHART 2 -------------------- */
  // Historique conservé:
  // - Connection per day
  // - Conversation per day

  renderChart2() {
    const el = document.getElementById('chart2')
    if (!el) {
      console.warn('[Chart2] DOM element #chart2 not found')
      return
    }

    const existing = echarts.getInstanceByDom(el)
    if (existing) {
      existing.dispose()
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
      yAxis: {},
      series: [
        {
          name: 'Connection per day',
          type: 'line',
          data: d.connectionPerDay,
          color: '#91a9dc'
        },
        {
          name: 'Conversation per day',
          type: 'line',
          data: d.conversationPerDay,
          color: '#005B50'
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
<div
  data-controller="chart"
  data-chart-target="container"
  class="justify-content-center container pb-2 mt-3"
>
  <div class="card rounded-3 pb-3 my-3 shadow">
    <div
      data-chart-target="chart1"
      style="height: 400px;"
    ></div>
  </div>

  <div class="card shadow rounded-3 pb-3">
    <div
      data-chart-target="chart2"
      style="height: 400px;"
    ></div>
  </div>
</div>
```
