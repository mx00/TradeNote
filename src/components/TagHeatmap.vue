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

    
        <div class="dailyCard p-3 mt-4">
            <h4>Bubble Heatmap</h4>
            <small class="text-muted">X = Win Rate, Y = Avg P&L, Size = Trades, Color = Total P&L</small>

            <div style="position:relative;height:380px;border:1px solid #333;border-radius:8px;margin-top:16px;background:linear-gradient(180deg,rgba(0,255,120,.06),rgba(255,80,80,.06));">
                <div style="position:absolute;left:10px;top:10px;font-size:11px;color:#888;">Higher Avg P&L</div>
                <div style="position:absolute;right:10px;bottom:8px;font-size:11px;color:#888;">Higher Win Rate →</div>

                <div
                    v-for="item in tagHeatmapData"
                    :key="'bubble-' + item.tag"
                    :title="item.tag + ' | WR: ' + item.winRate + '% | Avg: ' + useThousandCurrencyFormat(item.avgPnl)"
                    :style="{
                        position: 'absolute',
                        left: Math.max(4, Math.min(94, Number(item.winRate))) + '%',
                        top: Math.max(6, Math.min(88, 88 - ((Number(item.avgPnl || 0) + maxAbsPnl) / (maxAbsPnl * 2)) * 80)) + '%',
                        width: Math.max(34, Math.min(90, 34 + item.trades * 14)) + 'px',
                        height: Math.max(34, Math.min(90, 34 + item.trades * 14)) + 'px',
                        transform: 'translate(-50%, -50%)',
                        borderRadius: '999px',
                        background: item.pnl >= 0 ? 'rgba(0,200,117,.7)' : 'rgba(255,77,79,.7)',
                        border: '1px solid rgba(255,255,255,.35)',
                        display: 'flex',
                        alignItems: 'center',
                        justifyContent: 'center',
                        textAlign: 'center',
                        fontSize: '10px',
                        padding: '5px',
                        overflow: 'hidden'
                    }">
                    {{ item.tag }}
                </div>
            </div>
        </div>

    </div>
</template>
