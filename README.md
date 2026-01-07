```
import { Controller } from '@hotwired/stimulus'
import * as echarts from 'echarts'

export default class extends Controller {

  static values = {
    chart: String,
    target: String,
    url: String,
    data: Object
  }

  connect() {
    this.getData().then((data) => {
      this.dataValue = data
      this.getChart()
    })
  }

  render() {
    const exportForm = document.getElementById('export-usage-data')
    const formData = new FormData(exportForm)
    const params = new URLSearchParams(formData).toString()

    this.getData(params).then((data) => {
      this.dataValue = data
      this.getChart()
    })

    this.getKpi(params).then((data) => {
      document.getElementById('totalUsers').innerHTML = data.totalUsers
      document.getElementById('totalConversations').innerHTML = data.totalConversations
      document.getElementById('averageMessagesPerConversation').innerHTML = data.averageMessagesPerConversation
      document.getElementById('averageTimeToAnswer').innerHTML = data.averageTimeToAnswer
      document.getElementById('percentageLikedMessages').innerHTML = data.percentageLikedMessages
    })
  }

  /* -------------------- GRAPH -------------------- */

  getChart() {
    const dataSet = this.dataValue
    const chart = echarts.init(document.getElementById('chart'))

    window.addEventListener('resize', () => chart.resize())

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: {
        type: 'scroll',
        orient: 'horizontal',
        top: '95%',
        bottom: 0
      },
      xAxis: {
        type: 'category',
        data: dataSet.date
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
          data: dataSet.activeUsersPerDay,
          color: '#2bf3b6'
        },
        {
          name: 'Messages per day',
          type: 'line',
          data: dataSet.messagesPerDay,
          color: '#005B50'
        },
        {
          name: 'Avg response time',
          type: 'line',
          yAxisIndex: 1,
          data: dataSet.avgResponseTimePerDay,
          color: '#91a9dc'
        }
      ]
    })
  }

  /* -------------------- DATA FETCH -------------------- */

  async getData(params = '') {
    const response = await fetch('/chart_data?' + params)
    return response.json()
  }

  async getKpi(params = '') {
    const response = await fetch('/kpi_data?' + params)
    return response.json()
  }
}
```
