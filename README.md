```
import { Controller } from '@hotwired/stimulus'
import * as echarts from 'echarts'

export default class extends Controller {
  static targets = ['chart1', 'chart2']

  connect() {
    this.load()
  }

  async load() {
    const data = await this.getData()
    this.dataValue = data

    this.renderChart1()
    this.renderChart2()
  }

  /* ---------- CHART 1 ---------- */

  renderChart1() {
    if (!this.hasChart1Target) {
      console.warn('[Chart1] target not found')
      return
    }

    const el = this.chart1Target
    const existing = echarts.getInstanceByDom(el)
    if (existing) existing.dispose()

    const chart = echarts.init(el)

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: { top: '95%' },
      xAxis: { type: 'category', data: this.dataValue.date },
      yAxis: [
        { type: 'value', name: 'Utilisateurs / Messages' },
        { type: 'value', name: 'Temps de réponse (s)' }
      ],
      series: [
        {
          name: 'Utilisateurs actifs',
          type: 'bar',
          data: this.dataValue.userCount
        },
        {
          name: 'Messages',
          type: 'line',
          data: this.dataValue.messagesPerDay,
          yAxisIndex: 0,
          smooth: true
        },
        {
          name: 'Temps de réponse moyen',
          type: 'line',
          data: this.dataValue.avgResponseTimePerDay,
          yAxisIndex: 1,
          smooth: true
        }
      ]
    })
  }

  /* ---------- CHART 2 ---------- */

  renderChart2() {
    if (!this.hasChart2Target) {
      console.warn('[Chart2] target not found')
      return
    }

    const el = this.chart2Target
    const existing = echarts.getInstanceByDom(el)
    if (existing) existing.dispose()

    const chart = echarts.init(el)

    chart.setOption({
      tooltip: { trigger: 'axis' },
      xAxis: { type: 'category', data: this.dataValue.date },
      yAxis: { type: 'value' },
      series: [
        {
          name: 'Connexions',
          type: 'line',
          data: this.dataValue.connectionPerDay
        }
      ]
    })
  }

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
