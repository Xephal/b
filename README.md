```
import { Controller } from '@hotwired/stimulus'
import * as echarts from 'echarts'

export default class extends Controller {

  static values = {
    data: Object
  }

  connect() {
    this.load()
  }

  async load(params = '') {
    const data = await this.getData(params)
    this.dataValue = data

    this.renderChart1()
    this.renderChart2()
  }

  render() {
    const exportForm = document.getElementById('export-usage-data')
    const formData = new FormData(exportForm)
    const params = new URLSearchParams(formData).toString()

    this.load(params)

    this.getKpi(params).then((data) => {
      document.getElementById('totalUsers').innerHTML = data.totalUsers
      document.getElementById('totalConversations').innerHTML = data.totalConversations
      document.getElementById('averageMessagesPerConversation').innerHTML = data.averageMessagesPerConversation
      document.getElementById('averageTimeToAnswer').innerHTML = data.averageTimeToAnswer
      document.getElementById('percentageLikedMessages').innerHTML = data.percentageLikedMessages
    })
  }

  /* ==================== CHART 1 ==================== */
  /* Utilisateurs actifs / Messages / Temps réponse */

  renderChart1() {
    const d = this.dataValue
    const chart = echarts.init(document.getElementById('chart'))

    window.addEventListener('resize', () => chart.resize())

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: {
        top: '90%'
      },
      xAxis: {
        type: 'category',
        data: d.date
      },
      yAxis: [
        { type: 'value', name: 'Count' },
        { type: 'value', name: 'Seconds' }
      ],
      series: [
        {
          name: 'Active users',
          type: 'bar',
          data: d.activeUsersPerDay,
          color: '#2bf3b6'
        },
        {
          name: 'Messages',
          type: 'line',
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

  /* ==================== CHART 2 ==================== */
  /* Connexions / Conversations (legacy conservé) */

  renderChart2() {
    const d = this.dataValue
    const chart = echarts.init(document.getElementById('chart2'))

    window.addEventListener('resize', () => chart.resize())

    chart.setOption({
      tooltip: { trigger: 'axis' },
      legend: {
        top: '90%'
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
          name: 'Connections',
          type: 'line',
          data: d.connectionPerDay,
          color: '#e9ecef'
        },
        {
          name: 'Conversations',
          type: 'line',
          data: d.conversationPerDay,
          color: '#005B50'
        }
      ]
    })
  }

  /* ==================== DATA ==================== */

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
