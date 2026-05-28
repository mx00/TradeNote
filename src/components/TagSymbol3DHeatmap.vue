<script setup>
import { computed, onMounted, onBeforeUnmount, watch, nextTick } from 'vue'
import * as echarts from 'echarts'
import 'echarts-gl'
import { filteredTradesTrades, amountCase } from '../stores/globals'
import { useThousandCurrencyFormat } from '../utils/utils'

let chart = null

const chartId = 'tagSymbol3DHeatmapChart'

const matrixData = computed(() => {
    const symbolsSet = new Set()
    const tagsSet = new Set()
    const map = {}

    filteredTradesTrades.forEach(trade => {
        if (!Array.isArray(trade.tags)) return

        const symbol = trade.symbol || 'Unknown'
        symbolsSet.add(symbol)

        const pnl = Number(
            trade[amountCase.value + 'Proceeds']
            || trade.grossProceeds
            || trade.netProceeds
            || 0
        )

        trade.tags.forEach(tagObj => {
            const tag = typeof tagObj === 'string' ? tagObj : tagObj?.name
            if (!tag) return

            tagsSet.add(tag)

            const key = `${tag}|||${symbol}`

            if (!map[key]) {
                map[key] = {
                    tag,
                    symbol,
                    pnl: 0,
                    trades: 0,
                    wins: 0
                }
            }

            map[key].pnl += pnl
            map[key].trades += 1
            if (pnl > 0) map[key].wins += 1
        })
    })

    const symbols = Array.from(symbolsSet).sort()
    const tags = Array.from(tagsSet).sort()

    const data = []

    tags.forEach((tag, y) => {
        symbols.forEach((symbol, x) => {
            const cell = map[`${tag}|||${symbol}`]

            if (cell) {
                data.push([
                    x,
                    y,
                    Math.abs(cell.pnl),
                    cell.pnl,
                    cell.trades,
                    cell.wins,
                    tag,
                    symbol
                ])
            } else {
                data.push([x, y, 0, 0, 0, 0, tag, symbol])
            }
        })
    })

    return {
        symbols,
        tags,
        data
    }
})

const renderChart = async () => {
    await nextTick()

    const el = document.getElementById(chartId)
    if (!el) return

    if (!chart) {
        chart = echarts.init(el)
    }

    const payload = matrixData.value

    chart.resize()

    chart.setOption({
        tooltip: {
            formatter: params => {
                const v = params.value
                const pnl = v[3]
                const trades = v[4]
                const wins = v[5]
                const tag = v[6]
                const symbol = v[7]
                const winRate = trades ? ((wins / trades) * 100).toFixed(1) : '0.0'

                return `
                    <strong>${tag}</strong><br/>
                    Symbol: ${symbol}<br/>
                    P&L: ${useThousandCurrencyFormat(pnl)}<br/>
                    Trades: ${trades}<br/>
                    Win Rate: ${winRate}%
                `
            }
        },
        visualMap: {
            min: -5000,
            max: 5000,
            dimension: 3,
            calculable: true,
            inRange: {
                color: ['#ff4d4f', '#2b2b2b', '#00c875']
            },
            textStyle: {
                color: '#aaa'
            }
        },
        xAxis3D: {
            type: 'category',
            data: payload.symbols,
            axisLabel: {
                color: '#aaa'
            }
        },
        yAxis3D: {
            type: 'category',
            data: payload.tags,
            axisLabel: {
                color: '#aaa'
            }
        },
        zAxis3D: {
            type: 'value',
            name: 'P&L Strength',
            axisLabel: {
                color: '#aaa'
            }
        },
        grid3D: {
            boxWidth: 140,
            boxDepth: 80,
            viewControl: {
                projection: 'perspective',
                autoRotate: false,
                distance: 190,
                alpha: 35,
                beta: 35
            },
            light: {
                main: {
                    intensity: 1.2,
                    shadow: true
                },
                ambient: {
                    intensity: 0.35
                }
            }
        },
        series: [{
            type: 'bar3D',
            data: payload.data.map(item => ({
                value: item,
                itemStyle: {
                    opacity: item[4] > 0 ? 0.95 : 0.12
                }
            })),
            shading: 'lambert',
            bevelSize: 0.3,
            bevelSmoothness: 2,
            minHeight: 2,
            label: {
                show: false
            }
        }]
    })
}

onMounted(async () => {
    await renderChart()

    setTimeout(() => {
        if (chart) chart.resize()
    }, 300)

    setTimeout(() => {
        if (chart) chart.resize()
    }, 1000)

    window.addEventListener('resize', resizeChart)
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', resizeChart)
    if (chart) {
        chart.dispose()
        chart = null
    }
})

const resizeChart = () => {
    if (chart) chart.resize()
}

watch(matrixData, () => {
    renderChart()
}, { deep: true })
</script>

<template>
    <div class="dailyCard p-3 mt-4">
        <h4>Interactive Tag × Symbol 3D Heatmap</h4>
        <small class="text-muted">
            X = symbol, Y = tag, Z = P&L strength, color = profitability.
        </small>

        <div
            :id="chartId"
            style="height: 520px; width: 100%; margin-top: 16px;">
        </div>
    </div>
</template>
