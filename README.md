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

  renderChart1() {
    const el = document.getElementById('chart')
    if (!el) {
      console.warn('[Chart1] DOM element #chart not found')
      return
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
          name: 'Messages per day',
          type: 'bar',
          data: d.messagesPerDay
        },
        {
          name: 'Active users per day',
          type: 'bar',
          data: d.activeUsersPerDay
        },
        {
          name: 'Avg response time',
          type: 'line',
          data: d.avgResponseTimePerDay
        }
      ]
    })
  }

  /* -------------------- CHART 2 -------------------- */

  renderChart2() {
    const el = document.getElementById('chart2')
    if (!el) {
      console.warn('[Chart2] DOM element #chart2 not found')
      return
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
