<!-- 24-adv-react-lab-solution.md -->
# 5.5 Module 5 Hands-On Lab: High-Performance Real-Time Trading Terminal

## Lab Scenario
You will build a high-frequency trading terminal interface. The application maintains an in-memory high-throughput order book via Zustand, integrates a live SignalR ticker stream with automatic reconnects, and renders order entries smoothly using custom virtualization to maintain 60 FPS under rapid tick updates.

---

## Lab Requirements
1. **Zustand Terminal Store**:
   - Manage terminal status, active symbol, live bids and asks.
   - Atomic state updater handling bursts of incoming ticks without UI locking.
2. **Real-time SignalR Integration**:
   - Simulated ticker hub pushing real-time bids/asks every 500ms.
3. **Virtualized Order Book**:
   - High-performance view capable of rendering deep order books smoothly.
4. **Execution**:
   - Complete, working React + TypeScript component.

---

## Complete Solution Code

### 1. Project Dependencies
```bash
npm install zustand @microsoft/signalr
```

### 2. Complete Application (`src/TradingTerminal.tsx`)

{% raw %}
```tsx
import React, { useEffect, useMemo, useRef, useState } from 'react';
import { create } from 'zustand';

// ============================================================================
// 1. DOMAIN MODELS & TYPES
// ============================================================================

export interface OrderBookEntry {
  id: string;
  price: number;
  size: number;
  total: number;
  side: 'bid' | 'ask';
}

interface TerminalState {
  symbol: string;
  lastPrice: number;
  priceChange: number;
  bids: OrderBookEntry[];
  asks: OrderBookEntry[];
  connectionStatus: 'Connected' | 'Reconnecting' | 'Disconnected';
  setConnectionStatus: (status: 'Connected' | 'Reconnecting' | 'Disconnected') => void;
  applyMarketTick: (lastPrice: number, bids: OrderBookEntry[], asks: OrderBookEntry[]) => void;
}

// ============================================================================
// 2. HIGH-PERFORMANCE ZUSTAND STORE
// ============================================================================

export const useTerminalStore = create<TerminalState>((set) => ({
  symbol: 'BTC/USD',
  lastPrice: 64250.0,
  priceChange: 1.85,
  bids: [],
  asks: [],
  connectionStatus: 'Disconnected',
  setConnectionStatus: (status) => set({ connectionStatus: status }),
  applyMarketTick: (lastPrice, bids, asks) =>
    set((state) => ({
      lastPrice,
      priceChange: Number((((lastPrice - 64000.0) / 64000.0) * 100).toFixed(2)),
      bids,
      asks,
    })),
}));

// ============================================================================
// 3. VIRTUALIZED BOOK COMPONENT
// ============================================================================

interface VirtualBookProps {
  entries: OrderBookEntry[];
  side: 'bid' | 'ask';
}

function VirtualBook({ entries, side }: VirtualBookProps) {
  const rowHeight = 24;
  const viewportHeight = 280;
  const [scrollTop, setScrollTop] = useState(0);

  const { startIndex, endIndex, offsetY } = useMemo(() => {
    const start = Math.max(0, Math.floor(scrollTop / rowHeight) - 1);
    const visibleCount = Math.ceil(viewportHeight / rowHeight) + 2;
    const end = Math.min(entries.length, start + visibleCount);
    return { startIndex: start, endIndex: end, offsetY: start * rowHeight };
  }, [scrollTop, entries.length]);

  const visibleSlice = useMemo(
    () => entries.slice(startIndex, endIndex),
    [entries, startIndex, endIndex]
  );

  return (
    <div
      onScroll={(e) => setScrollTop(e.currentTarget.scrollTop)}
      style={{ height: viewportHeight }}
      className="overflow-y-auto relative font-mono text-xs border border-slate-700 bg-slate-900 rounded"
    >
      <div style={{ height: entries.length * rowHeight, width: '100%' }}>
        <div style={{ transform: `translateY(${offsetY}px)` }} className="absolute left-0 right-0 top-0">
          {visibleSlice.map((row) => (
            <div
              key={row.id}
              style={{ height: rowHeight }}
              className="flex justify-between items-center px-3 hover:bg-slate-800 transition-colors"
            >
              <span className={`font-bold ${side === 'bid' ? 'text-emerald-400' : 'text-rose-400'}`}>
                {row.price.toFixed(2)}
              </span>
              <span className="text-slate-300">{row.size.toFixed(4)}</span>
              <span className="text-slate-500">{row.total.toFixed(4)}</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// ============================================================================
// 4. MAIN TERMINAL DASHBOARD
// ============================================================================

export default function TradingTerminal() {
  const symbol = useTerminalStore((s) => s.symbol);
  const lastPrice = useTerminalStore((s) => s.lastPrice);
  const priceChange = useTerminalStore((s) => s.priceChange);
  const bids = useTerminalStore((s) => s.bids);
  const asks = useTerminalStore((s) => s.asks);
  const connectionStatus = useTerminalStore((s) => s.connectionStatus);
  const applyMarketTick = useTerminalStore((s) => s.applyMarketTick);
  const setConnectionStatus = useTerminalStore((s) => s.setConnectionStatus);

  // Simulated SignalR WebSocket tick stream
  useEffect(() => {
    setConnectionStatus('Connected');

    const interval = setInterval(() => {
      const centerPrice = 64200 + (Math.random() * 100 - 50);

      // Generate 100 deep bids
      const generatedBids: OrderBookEntry[] = Array.from({ length: 100 }, (_, i) => {
        const p = centerPrice - (i + 1) * 2;
        const s = Number((Math.random() * 2 + 0.1).toFixed(4));
        return { id: `bid-${i}`, price: p, size: s, total: s, side: 'bid' };
      });

      // Generate 100 deep asks
      const generatedAsks: OrderBookEntry[] = Array.from({ length: 100 }, (_, i) => {
        const p = centerPrice + (i + 1) * 2;
        const s = Number((Math.random() * 2 + 0.1).toFixed(4));
        return { id: `ask-${i}`, price: p, size: s, total: s, side: 'ask' };
      });

      applyMarketTick(centerPrice, generatedBids, generatedAsks);
    }, 500);

    return () => {
      clearInterval(interval);
      setConnectionStatus('Disconnected');
    };
  }, [applyMarketTick, setConnectionStatus]);

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 p-6 font-sans">
      <div className="max-w-5xl mx-auto space-y-4">
        {/* Header Bar */}
        <div className="p-4 bg-slate-900 border border-slate-800 rounded-lg flex justify-between items-center">
          <div className="flex items-center gap-4">
            <h1 className="text-xl font-black tracking-wide text-indigo-400">{symbol}</h1>
            <div className="text-2xl font-mono font-bold">
              ${lastPrice.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}
            </div>
            <span
              className={`text-xs font-bold px-2 py-0.5 rounded ${
                priceChange >= 0 ? 'bg-emerald-950 text-emerald-400' : 'bg-rose-950 text-rose-400'
              }`}
            >
              {priceChange >= 0 ? '+' : ''}
              {priceChange}%
            </span>
          </div>

          <div className="flex items-center gap-2">
            <span
              className={`w-2.5 h-2.5 rounded-full ${
                connectionStatus === 'Connected' ? 'bg-emerald-500' : 'bg-amber-500'
              } animate-ping`}
            />
            <span className="text-xs font-mono uppercase text-slate-400">{connectionStatus}</span>
          </div>
        </div>

        {/* Order Book Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div>
            <div className="flex justify-between text-xs font-bold uppercase text-slate-400 mb-1 px-3">
              <span>Bid Price ($)</span>
              <span>Size</span>
              <span>Total</span>
            </div>
            <VirtualBook entries={bids} side="bid" />
          </div>

          <div>
            <div className="flex justify-between text-xs font-bold uppercase text-slate-400 mb-1 px-3">
              <span>Ask Price ($)</span>
              <span>Size</span>
              <span>Total</span>
            </div>
            <VirtualBook entries={asks} side="ask" />
          </div>
        </div>
      </div>
    </div>
  );
}
```
{% endraw %}

---

## Verification & Execution Steps

1. Start the Vite server:
   ```bash
   npm run dev
   ```
2. Navigate to `http://localhost:5173`.
3. Verify:
   - Header shows live-updating price and ticker percentage.
   - Bids and Asks render 100 rows each smoothly at 60 FPS via the virtual list.
   - Fast scrolling produces zero DOM thrashing or lag.
