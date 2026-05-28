<script setup>
import { computed } from 'vue'
import { filteredTradesTrades, amountCase } from '../stores/globals'
import { useThousandCurrencyFormat } from '../utils/utils'

const tagHeatmapData = computed(() => {
    const stats = {}

    filteredTradesTrades.forEach(trade => {
        if (!Array.isArray(trade.tags)) return

        const pnl = Number(
            trade[amountCase.value + 'Proceeds']
            || trade.grossProceeds
            || trade.netProceeds
            || 0
        )

        trade.tags.forEach(tagObj => {
            const tagName = typeof tagObj === 'string'
                ? tagObj
                : tagObj?.name

            if (!tagName) return

            if (!stats[tagName]) {
                stats[tagName] = {
                    tag: tagName,
                    trades: 0,
                    wins: 0,
                    pnl: 0
                }
            }

            stats[tagName].trades += 1
            stats[tagName].pnl += pnl

            if (pnl > 0) {
                stats[tagName].wins += 1
            }
        })
    })

    return Object.values(stats)
        .map(item => ({
            ...item,
            winRate: item.trades
                ? ((item.wins / item.trades) * 100).toFixed(1)
                : '0.0',
            avgPnl: item.trades
                ? item.pnl / item.trades
                : 0
        }))
        .sort((a, b) => b.pnl - a.pnl)
})

const tagSymbolMatrix = computed(() => {
    const matrix = {}
    const symbols = new Set()

    filteredTradesTrades.forEach(trade => {
        if (!Array.isArray(trade.tags)) return

        const symbol = trade.symbol || trade.ticker || trade.asset || 'Unknown'
        symbols.add(symbol)

        const pnl = Number(
            trade[amountCase.value + 'Proceeds']
            || trade.grossProceeds
            || trade.netProceeds
            || 0
        )

        trade.tags.forEach(tagObj => {
            const tagName = typeof tagObj === 'string' ? tagObj : tagObj?.name
            if (!tagName) return

            if (!matrix[tagName]) matrix[tagName] = {}
            if (!matrix[tagName][symbol]) {
                matrix[tagName][symbol] = {
                    tag: tagName,
                    symbol,
                    trades: 0,
                    pnl: 0,
                    wins: 0
                }
            }

            matrix[tagName][symbol].trades += 1
            matrix[tagName][symbol].pnl += pnl
            if (pnl > 0) matrix[tagName][symbol].wins += 1
        })
    })

    return {
        symbols: Array.from(symbols).sort(),
        rows: Object.keys(matrix)
            .sort()
            .map(tag => ({
                tag,
                cells: matrix[tag]
            }))
    }
})

const maxMatrixAbsPnl = computed(() => {
    let max = 1
    tagSymbolMatrix.value.rows.forEach(row => {
        Object.values(row.cells).forEach(cell => {
            max = Math.max(max, Math.abs(cell.pnl))
        })
    })
    return max
})

const matrixCellBackground = cell => {
    if (!cell) return 'transparent'

    const intensity = Math.max(0.12, Math.min(0.85, Math.abs(cell.pnl) / maxMatrixAbsPnl.value))

    if (cell.pnl >= 0) {
        return `rgba(0, 200, 117, ${intensity})`
    }

    return `rgba(255, 77, 79, ${intensity})`
}

const matrixCellSize = cell => {
    if (!cell) return 0
    return Math.max(28, Math.min(76, 28 + cell.trades * 12))
}

const matrixWinRate = cell => {
    if (!cell || !cell.trades) return '0.0'
    return ((cell.wins / cell.trades) * 100).toFixed(1)
}


const bar3dHeight = cell => {
    if (!cell) return 10
    return Math.max(16, Math.min(58, 16 + (Math.abs(cell.pnl) / maxMatrixAbsPnl.value) * 42))
}

const bar3dColor = cell => {
    if (!cell) return 'rgba(120,120,120,.15)'
    const opacity = Math.max(0.28, Math.min(0.95, Math.abs(cell.pnl) / maxMatrixAbsPnl.value))
    return cell.pnl >= 0
        ? `rgba(0,200,117,${opacity})`
        : `rgba(255,77,79,${opacity})`
}

</script>

<template>
    <div class="dailyCard p-3">

        <h4 class="mb-4">Tag Heatmap</h4>

        <div
            v-if="tagHeatmapData.length === 0"
            class="text-muted">
            No tagged trades found.
        </div>

        <div
            v-for="item in tagHeatmapData"
            :key="item.tag"
            class="mb-3">

            <div class="d-flex justify-content-between mb-1">

                <div>
                    <strong>{{ item.tag }}</strong>
                    <small class="ms-2 text-muted">
                        {{ item.trades }} trades
                    </small>
                </div>

                <div
                    :class="item.pnl >= 0
                        ? 'text-success'
                        : 'text-danger'">

                    {{ useThousandCurrencyFormat(item.pnl) }}

                </div>
            </div>

            <div
                style="
                    height: 14px;
                    background: #222;
                    border-radius: 999px;
                    overflow: hidden;
                ">

                <div
                    :style="{
                        height: '14px',
                        width: Math.min(Math.abs(item.pnl) / 50, 100) + '%',
                        background:
                            item.pnl >= 0
                                ? '#00c875'
                                : '#ff4d4f'
                    }">
                </div>

            </div>

            <small class="text-muted">
                Win Rate: {{ item.winRate }}%
                · Avg:
                {{ useThousandCurrencyFormat(item.avgPnl) }}
            </small>

        </div>


        <div class="dailyCard p-3 mt-4">
            <h4>Tag × Symbol 3D Matrix Heatmap</h4>
            <small class="text-muted">
                Rows = tags, columns = symbols, color = total P&L, bubble size = number of trades.
            </small>

            <div
                v-if="tagSymbolMatrix.rows.length === 0"
                class="text-muted mt-3">
                No tag/symbol combinations found.
            </div>

            <div
                v-else
                class="table-responsive mt-3">

                <table class="table table-dark align-middle text-center">
                    <thead>
                        <tr>
                            <th class="text-start">Tag / Symbol</th>
                            <th
                                v-for="symbol in tagSymbolMatrix.symbols"
                                :key="'sym-head-' + symbol">
                                {{ symbol }}
                            </th>
                        </tr>
                    </thead>

                    <tbody>
                        <tr
                            v-for="row in tagSymbolMatrix.rows"
                            :key="'matrix-row-' + row.tag">

                            <td class="text-start">
                                <strong>{{ row.tag }}</strong>
                            </td>

                            <td
                                v-for="symbol in tagSymbolMatrix.symbols"
                                :key="row.tag + '-' + symbol">

                                <div
                                    v-if="row.cells[symbol]"
                                    class="mx-auto d-flex flex-column align-items-center justify-content-center"
                                    :title="row.tag + ' / ' + symbol + ' | P&L: ' + useThousandCurrencyFormat(row.cells[symbol].pnl) + ' | Trades: ' + row.cells[symbol].trades + ' | WR: ' + matrixWinRate(row.cells[symbol]) + '%'"
                                    :style="{
                                        width: matrixCellSize(row.cells[symbol]) + 'px',
                                        height: matrixCellSize(row.cells[symbol]) + 'px',
                                        borderRadius: '14px',
                                        background: matrixCellBackground(row.cells[symbol]),
                                        border: '1px solid rgba(255,255,255,.25)',
                                        boxShadow: row.cells[symbol].pnl >= 0
                                            ? '0 10px 22px rgba(0,200,117,.18)'
                                            : '0 10px 22px rgba(255,77,79,.18)',
                                        transform: 'perspective(500px) rotateX(10deg)',
                                        fontSize: '11px',
                                        lineHeight: '1.2'
                                    }">

                                    <strong>
                                        {{ useThousandCurrencyFormat(row.cells[symbol].pnl) }}
                                    </strong>

                                    <small>
                                        {{ row.cells[symbol].trades }} trades
                                    </small>

                                    <small>
                                        {{ matrixWinRate(row.cells[symbol]) }}%
                                    </small>

                                </div>

                                <span
                                    v-else
                                    class="text-muted">
                                    —
                                </span>

                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>


        <div class="dailyCard p-3 mt-4">
            <h4>Tag × Symbol 3D Bar Heatmap</h4>
            <small class="text-muted">
                Rows = tags, columns = symbols, height/color = P&L strength.
            </small>

            <div class="table-responsive mt-3">
                <table class="table table-dark align-middle text-center">
                    <thead>
                        <tr>
                            <th class="text-start">Tag</th>
                            <th v-for="symbol in tagSymbolMatrix.symbols" :key="'bar-head-' + symbol">
                                {{ symbol }}
                            </th>
                        </tr>
                    </thead>

                    <tbody>
                        <tr v-for="row in tagSymbolMatrix.rows" :key="'bar-row-' + row.tag">
                            <td class="text-start">
                                <strong>{{ row.tag }}</strong>
                            </td>

                            <td v-for="symbol in tagSymbolMatrix.symbols" :key="'bar-cell-' + row.tag + '-' + symbol">
                                <div
                                    v-if="row.cells[symbol]"
                                    class="mx-auto"
                                    :title="row.tag + ' / ' + symbol + ' | P&L: ' + useThousandCurrencyFormat(row.cells[symbol].pnl)"
                                    style="width:80px;height:88px;position:relative;perspective:500px;">

                                    <div
                                        :style="{
                                            width: '46px',
                                            height: bar3dHeight(row.cells[symbol]) + 'px',
                                            position: 'absolute',
                                            left: '14px',
                                            bottom: '22px',
                                            background: bar3dColor(row.cells[symbol]),
                                            border: '1px solid rgba(255,255,255,.45)',
                                            transform: 'skewY(-12deg)',
                                            boxShadow: row.cells[symbol].pnl >= 0
                                                ? '9px -9px 0 rgba(0,200,117,.22)'
                                                : '9px -9px 0 rgba(255,77,79,.22)',
                                            borderRadius: '2px'
                                        }">
                                    </div>

                                    <small
                                        :class="row.cells[symbol].pnl >= 0 ? 'text-success' : 'text-danger'"
                                        style="position:absolute;left:0;right:0;bottom:0;font-size:10px;">
                                        {{ useThousandCurrencyFormat(row.cells[symbol].pnl) }}
                                    </small>
                                </div>

                                <span v-else class="text-muted">—</span>
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>

    </div>
</template>
