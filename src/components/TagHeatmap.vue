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

    </div>
</template>
