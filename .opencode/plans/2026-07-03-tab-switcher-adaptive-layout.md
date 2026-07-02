# Tab Switcher Adaptive Layout Implementation Plan

> **For agentic workers:** Implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the Alt+Q tab switcher overlay to adapt its layout based on tab count — horizontal strip for ≤7 tabs, responsive grid for 7–20 tabs, and grid + search input for >20 tabs — while preserving keyboard navigation, CSS isolation, and Shadow DOM rendering.

**Architecture:** Controller in `useTabSwitcher.tsx` manages lifecycle + search query state. Overlay component switches between strip/grid rendering based on `getLayoutMode(tabs.length)`. Grid spatial navigation is a pure function computed from column count + selected index. All styling stays inline + scoped `<style>` in Shadow DOM. No new dependencies.

**Tech Stack:** React 19, WXT 0.20, TypeScript, Shadow DOM (closed mode), inline CSS.

---

## File Structure

| File | Status | Responsibility |
|------|--------|----------------|
| `entrypoints/tab-switcher/types.ts` | Modify | Add `searchQuery`, `layoutMode` types |
| `entrypoints/tab-switcher/TabSwitcherOverlay.tsx` | Rewrite | Layout router (strip/grid/grid+search), keyboard dispatch, search input, footer |
| `entrypoints/tab-switcher/TabPreviewCard.tsx` | Modify | Two-line title clamp, dynamic card sizing, enhanced selected glow, dim non-selected |
| `entrypoints/tab-switcher/useTabSwitcher.tsx` | Modify | Add `searchQuery`/`layoutMode` to state, handle search change |

---

### Task 1: Update `types.ts` — add search and layout types

**Files:**
- Modify: `entrypoints/tab-switcher/types.ts`

- [ ] **Step 1: Replace file content with new types**

Replace the content of `entrypoints/tab-switcher/types.ts`:

```typescript
export type LayoutMode = 'strip' | 'grid'

export interface SwitcherState {
  phase: 'idle' | 'open'
  tabs: chrome.tabs.Tab[]
  selectedIndex: number
  thumbnails: Record<number, string>
  searchQuery: string
  layoutMode: LayoutMode
}

export interface SwitcherController {
  advance: (direction: 1 | -1) => void
  select: () => void
  dismiss: () => void
  setSearchQuery: (query: string) => void
  state: SwitcherState
}
```

- [ ] **Step 2: Build to type-check**

```bash
cd packages/browseros-agent/apps/app
bun run typecheck
```

Expected: type errors in `TabSwitcherOverlay.tsx` and `useTabSwitcher.tsx` (missing fields). Resolved in later tasks.

---

### Task 2: Update `TabPreviewCard.tsx` — two-line title, dynamic sizing, enhanced visual state

**Files:**
- Modify: `entrypoints/tab-switcher/TabPreviewCard.tsx`

- [ ] **Step 1: Replace file content**

Changes from current:
- Accept `cardWidth` and `cardHeight` props (dynamic sizing)
- Title changes from single-line truncation to **two-line clamp** using `display: -webkit-box; -webkit-line-clamp: 2`
- Selected card: stronger glow (`box-shadow` blur increased to 28px)
- Non-selected: slight dim via `opacity: 0.75`
- Added `title` attribute on the title span for tooltip on hover
- Thumbnail area height dynamically computed as `cardHeight - 48`
- favicon size reduced slightly to fit smaller cards (12px instead of 14px)

```typescript
import type { FC } from 'react'
import { getFavicons } from '@/lib/getFavicons'

interface TabPreviewCardProps {
  tab: chrome.tabs.Tab
  isSelected: boolean
  thumbnail?: string
  cardWidth: number
  cardHeight: number
}

function getHost(url?: string): string {
  try {
    return new URL(url ?? '').hostname
  } catch {
    return ''
  }
}

function getDomain(url?: string): string {
  const host = getHost(url)
  return host.startsWith('www.') ? host.slice(4) : host
}

export const TabPreviewCard: FC<TabPreviewCardProps> = ({
  tab,
  isSelected,
  thumbnail,
  cardWidth,
  cardHeight,
}) => {
  const thumbnailAreaHeight = cardHeight - 48

  return (
    <div
      style={{
        display: 'flex',
        flexDirection: 'column',
        width: `${cardWidth}px`,
        height: `${cardHeight}px`,
        borderRadius: '12px',
        border: isSelected
          ? '2px solid rgba(255, 160, 50, 0.9)'
          : '1px solid rgba(255, 255, 255, 0.1)',
        padding: '8px',
        cursor: 'pointer',
        background: isSelected
          ? 'rgba(255, 255, 255, 0.12)'
          : 'rgba(255, 255, 255, 0.04)',
        backdropFilter: 'blur(20px)',
        WebkitBackdropFilter: 'blur(20px)',
        opacity: isSelected ? 1 : 0.75,
        boxShadow: isSelected
          ? '0 0 28px rgba(255, 160, 50, 0.4), 0 8px 32px rgba(0, 0, 0, 0.4)'
          : '0 2px 8px rgba(0, 0, 0, 0.2)',
        transform: isSelected ? 'scale(1.08)' : 'scale(1)',
        transition: 'all 0.15s ease-out',
        boxSizing: 'border-box',
      }}
    >
      <div
        style={{
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          width: '100%',
          height: `${thumbnailAreaHeight}px`,
          overflow: 'hidden',
          borderRadius: '8px',
          background: 'rgba(0, 0, 0, 0.2)',
          flexShrink: 0,
        }}
      >
        {thumbnail ? (
          <img
            src={thumbnail}
            alt=""
            style={{
              width: '100%',
              height: '100%',
              objectFit: 'cover',
            }}
          />
        ) : (
          <img
            src={getFavicons(getHost(tab.url))}
            alt=""
            style={{
              width: '28px',
              height: '28px',
            }}
          />
        )}
      </div>

      <div
        style={{
          display: 'flex',
          alignItems: 'flex-start',
          gap: '6px',
          width: '100%',
          marginTop: '6px',
          minHeight: '0',
          flex: 1,
        }}
      >
        <img
          src={tab.favIconUrl ?? getFavicons(getHost(tab.url))}
          alt=""
          style={{
            width: '12px',
            height: '12px',
            borderRadius: '2px',
            flexShrink: 0,
            marginTop: '2px',
          }}
        />
        <span
          title={tab.title ?? getDomain(tab.url)}
          style={{
            fontSize: '10px',
            fontWeight: 500,
            lineHeight: 1.3,
            color: isSelected ? '#fff' : 'rgba(255, 255, 255, 0.8)',
            display: '-webkit-box',
            WebkitLineClamp: 2,
            WebkitBoxOrient: 'vertical',
            overflow: 'hidden',
            wordBreak: 'break-word',
          }}
        >
          {tab.title ?? getDomain(tab.url)}
        </span>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Build to type-check**

```bash
cd packages/browseros-agent/apps/app
bun run typecheck
```

Expected: only errors in `TabSwitcherOverlay.tsx` (not updated yet) and `useTabSwitcher.tsx` (not updated yet).

---

### Task 3: Rewrite `TabSwitcherOverlay.tsx` — adaptive layout + spatial navigation

**Files:**
- Rewrite: `entrypoints/tab-switcher/TabSwitcherOverlay.tsx`

- [ ] **Step 1: Replace file with adaptive overlay**

The overlay has three modes dispatched in the render:
- **Strip mode** (≤7 tabs): Horizontal flex row with `overflow-x: auto; scrollbar-width: none`. Selected card centered via `useLayoutEffect` + `scrollIntoView({ inline: 'center', behavior: 'smooth', block: 'nearest' })`.
- **Grid mode** (7–20 tabs): CSS Grid. Column count = `Math.floor(containerWidth / (cardWidth + gap))`. Cards fill row by row in MRU order.
- **Search mode** (>20 tabs): Same grid + `<input>` at top with `autoFocus`. Filtered tabs computed by lowercasing title/URL match against `searchQuery`.

Spatial navigation (`getSpatialIndex`):
- Left: decrement, wrapping to previous row's last card if at column 0
- Right: increment, wrapping to row start if at last column or last card
- Up: subtract `columns`, clamp to 0
- Down: add `columns`, wrapping to column 0 if at row's last card

Tab/Shift+Tab still calls `onAdvance` (linear MRU cycling, unchanged).

```typescript
import { type FC, useLayoutEffect, useRef, useState, useCallback, useMemo } from 'react'
import { TabPreviewCard } from './TabPreviewCard'
import type { SwitcherState } from './types'

interface TabSwitcherOverlayProps {
  state: SwitcherState
  onSelectTab: (index: number) => void
  onAdvance: (direction: 1 | -1) => void
  onSelect: () => void
  onDismiss: () => void
  onSearchChange: (query: string) => void
}

const CARD_WIDTH_STRIP = 200
const CARD_HEIGHT_STRIP = 140
const CARD_WIDTH_GRID = 176
const CARD_HEIGHT_GRID = 118
const GAP = 12

function getCardDimensions(layoutMode: 'strip' | 'grid'): {
  width: number
  height: number
} {
  return layoutMode === 'strip'
    ? { width: CARD_WIDTH_STRIP, height: CARD_HEIGHT_STRIP }
    : { width: CARD_WIDTH_GRID, height: CARD_HEIGHT_GRID }
}

function getColumns(containerWidth: number, cardWidth: number): number {
  const available = containerWidth - GAP
  const cell = cardWidth + GAP
  return Math.max(2, Math.floor(available / cell))
}

function clampIndex(index: number, length: number): number {
  return ((index % length) + length) % length
}

export const TabSwitcherOverlay: FC<TabSwitcherOverlayProps> = ({
  state,
  onSelectTab,
  onAdvance,
  onSelect,
  onDismiss,
  onSearchChange,
}) => {
  const { tabs, selectedIndex, thumbnails, searchQuery, layoutMode } = state
  const totalTabs = tabs.length
  if (totalTabs === 0) return null

  const stripRef = useRef<HTMLDivElement>(null)
  const containerRef = useRef<HTMLDivElement>(null)
  const searchInputRef = useRef<HTMLInputElement>(null)
  const [containerWidth, setContainerWidth] = useState(0)

  const showSearch = totalTabs > 20

  const filteredTabs = useMemo(() => {
    if (!showSearch || !searchQuery.trim()) return tabs
    const q = searchQuery.toLowerCase()
    return tabs.filter(
      (t) =>
        (t.title && t.title.toLowerCase().includes(q)) ||
        (t.url && t.url.toLowerCase().includes(q)),
    )
  }, [tabs, searchQuery, showSearch])

  const filteredSelectedIndex = useMemo(() => {
    if (!showSearch || !searchQuery.trim()) return selectedIndex
    const currentTab = tabs[selectedIndex]
    if (!currentTab) return 0
    const idx = filteredTabs.indexOf(currentTab)
    return idx >= 0 ? idx : 0
  }, [tabs, filteredTabs, selectedIndex, showSearch, searchQuery])

  const activeIndex = showSearch && searchQuery.trim()
    ? filteredSelectedIndex
    : selectedIndex
  const activeTabs = showSearch && searchQuery.trim()
    ? filteredTabs
    : tabs

  const { width: cardWidth, height: cardHeight } = getCardDimensions(layoutMode)

  const columns = layoutMode === 'strip'
    ? activeTabs.length
    : getColumns(containerWidth, cardWidth)

  const selectedCardRef = useRef<HTMLDivElement>(null)

  const getSpatialIndex = useCallback(
    (current: number, direction: 'up' | 'down' | 'left' | 'right'): number => {
      const col = current % columns
      const row = Math.floor(current / columns)

      switch (direction) {
        case 'left':
          if (col === 0) {
            const target = row * columns - 1
            return clampIndex(target >= 0 ? target : activeTabs.length - 1, activeTabs.length)
          }
          return clampIndex(current - 1, activeTabs.length)
        case 'right':
          if (col === columns - 1 || current === activeTabs.length - 1) {
            return clampIndex(row * columns, activeTabs.length)
          }
          return clampIndex(current + 1, activeTabs.length)
        case 'up': {
          const target = current - columns
          if (target < 0) {
            return clampIndex(current, activeTabs.length)
          }
          return clampIndex(target, activeTabs.length)
        }
        case 'down': {
          const target = current + columns
          const lastInRow = Math.min(row * columns + columns - 1, activeTabs.length - 1)
          if (current === lastInRow || target >= activeTabs.length) {
            return clampIndex(col, activeTabs.length)
          }
          return clampIndex(target, activeTabs.length)
        }
      }
    },
    [columns, activeTabs.length],
  )

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'Tab':
        e.preventDefault()
        e.stopPropagation()
        onAdvance(e.shiftKey ? -1 : 1)
        break
      case 'ArrowLeft':
        e.preventDefault()
        {
          const spatial = getSpatialIndex(activeIndex, 'left')
          const realIndex = showSearch && searchQuery.trim()
            ? tabs.indexOf(activeTabs[spatial])
            : spatial
          onSelectTab(realIndex >= 0 ? realIndex : spatial)
        }
        break
      case 'ArrowRight':
        e.preventDefault()
        {
          const spatial = getSpatialIndex(activeIndex, 'right')
          const realIndex = showSearch && searchQuery.trim()
            ? tabs.indexOf(activeTabs[spatial])
            : spatial
          onSelectTab(realIndex >= 0 ? realIndex : spatial)
        }
        break
      case 'ArrowUp':
        e.preventDefault()
        {
          const spatial = getSpatialIndex(activeIndex, 'up')
          const realIndex = showSearch && searchQuery.trim()
            ? tabs.indexOf(activeTabs[spatial])
            : spatial
          onSelectTab(realIndex >= 0 ? realIndex : spatial)
        }
        break
      case 'ArrowDown':
        e.preventDefault()
        {
          const spatial = getSpatialIndex(activeIndex, 'down')
          const realIndex = showSearch && searchQuery.trim()
            ? tabs.indexOf(activeTabs[spatial])
            : spatial
          onSelectTab(realIndex >= 0 ? realIndex : spatial)
        }
        break
      case 'Enter':
        e.preventDefault()
        onSelect()
        break
      case 'Escape':
        e.preventDefault()
        if (showSearch && searchQuery) {
          onSearchChange('')
        } else {
          onDismiss()
        }
        break
    }
  }

  useLayoutEffect(() => {
    if (layoutMode === 'strip' && selectedCardRef.current) {
      selectedCardRef.current.scrollIntoView({
        inline: 'center',
        behavior: 'smooth',
        block: 'nearest',
      })
    }
  }, [selectedIndex, layoutMode])

  useLayoutEffect(() => {
    if (!containerRef.current) return
    const observer = new ResizeObserver((entries) => {
      for (const entry of entries) {
        setContainerWidth(entry.contentRect.width)
      }
    })
    observer.observe(containerRef.current)
    return () => observer.disconnect()
  }, [])

  useLayoutEffect(() => {
    if (showSearch && searchInputRef.current) {
      searchInputRef.current.focus()
    }
  }, [showSearch])

  const renderSearchInput = () => {
    if (!showSearch) return null
    return (
      <input
        ref={searchInputRef}
        type="text"
        placeholder="Search tabs by title or URL..."
        value={searchQuery}
        onChange={(e) => onSearchChange(e.target.value)}
        style={{
          width: '100%',
          maxWidth: '480px',
          padding: '10px 16px',
          borderRadius: '8px',
          border: '1px solid rgba(255, 255, 255, 0.15)',
          background: 'rgba(255, 255, 255, 0.08)',
          color: '#fff',
          fontSize: '14px',
          outline: 'none',
          fontFamily: 'inherit',
        }}
        onKeyDown={(e) => {
          if (e.key === 'Escape') {
            e.stopPropagation()
            if (searchQuery) {
              onSearchChange('')
            } else {
              onDismiss()
            }
          }
          if (e.key === 'ArrowDown' || e.key === 'ArrowRight') {
            e.preventDefault()
            const spatial = getSpatialIndex(0, 'right')
            const realIndex = tabs.indexOf(activeTabs[spatial])
            onSelectTab(realIndex >= 0 ? realIndex : spatial)
          }
        }}
      />
    )
  }

  const renderCards = () => {
    const isStrip = layoutMode === 'strip'

    return (
      <div
        ref={containerRef}
        style={{
          display: isStrip ? 'flex' : 'grid',
          flexDirection: isStrip ? 'row' : undefined,
          gridTemplateColumns: isStrip ? undefined : `repeat(${columns}, ${cardWidth}px)`,
          gap: `${GAP}px`,
          overflowX: isStrip ? 'auto' : 'visible',
          overflowY: isStrip ? 'visible' : 'auto',
          padding: '8px 16px',
          maxWidth: isStrip ? '90vw' : 'min(90vw, 1200px)',
          maxHeight: '70vh',
          scrollbarWidth: 'none',
          justifyContent: isStrip ? 'flex-start' : 'center',
          justifyItems: 'center',
        }}
      >
        {activeTabs.map((tab, index) => {
          const isSelected = index === activeIndex
          return (
            <div
              key={tab.id}
              ref={isSelected ? selectedCardRef : undefined}
              onClick={() => {
                const realIndex = showSearch && searchQuery.trim()
                  ? tabs.indexOf(tab)
                  : index
                if (realIndex >= 0) onSelectTab(realIndex)
              }}
            >
              <TabPreviewCard
                tab={tab}
                isSelected={isSelected}
                thumbnail={thumbnails[tab.id!]}
                cardWidth={cardWidth}
                cardHeight={cardHeight}
              />
            </div>
          )
        })}
      </div>
    )
  }

  return (
    <div
      onKeyDown={handleKeyDown}
      style={{
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        gap: '16px',
        outline: 'none',
        width: '90vw',
        maxWidth: '1200px',
      }}
    >
      {renderSearchInput()}
      {renderCards()}
      <div
        style={{
          display: 'flex',
          alignItems: 'center',
          gap: '16px',
          fontSize: '12px',
          color: 'rgba(255, 255, 255, 0.5)',
        }}
      >
        <span>
          {filteredTabs.length === tabs.length
            ? `${selectedIndex + 1} / ${totalTabs} tabs`
            : `${activeIndex + 1} / ${filteredTabs.length} (filtered from ${totalTabs})`}
        </span>
        <span
          style={{
            width: '4px',
            height: '4px',
            borderRadius: '50%',
            background: 'rgba(255, 255, 255, 0.3)',
          }}
        />
        <span>Tab/arrows to navigate</span>
        <span
          style={{
            width: '4px',
            height: '4px',
            borderRadius: '50%',
            background: 'rgba(255, 255, 255, 0.3)',
          }}
        />
        <span>Esc to {showSearch && searchQuery ? 'clear search' : 'cancel'}</span>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Build to type-check**

```bash
cd packages/browseros-agent/apps/app
bun run typecheck
```

Expected: only error is `setSearchQuery` not yet plumbed in `useTabSwitcher.tsx`.

---

### Task 4: Update `useTabSwitcher.tsx` — search state, layout mode, setSearchQuery

**Files:**
- Modify: `entrypoints/tab-switcher/useTabSwitcher.tsx`

- [ ] **Step 1: Update controller with search state and layout mode**

Changes:
- Add `searchQuery: ''` and `layoutMode: 'strip'` to initial state
- Add `getLayoutMode()` helper: `tabs.length <= 7 ? 'strip' : 'grid'`
- Compute `layoutMode` in `open()` based on `tabs.length`
- Add `setSearchQuery(query)` method that updates `_state.searchQuery` and re-renders
- Pass `onSearchChange` prop to `<TabSwitcherOverlay>`
- Reset to initial values in `dismiss()`

```typescript
import { createRoot, type Root } from 'react-dom/client'
import type { ContentScriptContext } from 'wxt/utils/content-script-context'
import { sendSwitcherMessage } from '@/lib/messaging/switcher/switcherMessages'
import { TabSwitcherOverlay } from './TabSwitcherOverlay'
import type { LayoutMode, SwitcherState } from './types'

export class TabSwitcherController {
  private _state: SwitcherState = {
    phase: 'idle',
    tabs: [],
    selectedIndex: 0,
    thumbnails: {},
    searchQuery: '',
    layoutMode: 'strip',
  }

  private dialogEl: HTMLDialogElement | null = null
  private root: Root | null = null
  private mountPoint: HTMLDivElement | null = null
  private ctx: ContentScriptContext

  constructor(ctx: ContentScriptContext) {
    this.ctx = ctx
  }

  get state(): SwitcherState {
    return this._state
  }

  private handleDialogClick = (e: MouseEvent) => {
    const path = e.composedPath()
    if (path[0] === this.dialogEl || path[0] === this.dialogEl?.firstChild) {
      this.dismiss()
    }
  }

  private getLayoutMode(tabCount: number): LayoutMode {
    if (tabCount <= 7) return 'strip'
    return 'grid'
  }

  async open(direction: 1 | -1) {
    const response = await sendSwitcherMessage('tab-switcher:get-tabs')
    const tabs = response.tabs
    if (tabs.length === 0) return

    const thumbnails = response.thumbnails

    const activeIdx = tabs.findIndex((t) => t.active)
    const excludeActive = activeIdx >= 0

    let selectedIndex: number
    if (direction > 0) {
      selectedIndex = excludeActive ? (activeIdx + 1) % tabs.length : 0
    } else {
      selectedIndex = excludeActive
        ? (activeIdx - 1 + tabs.length) % tabs.length
        : tabs.length - 1
    }

    this._state = {
      phase: 'open',
      tabs,
      selectedIndex,
      thumbnails,
      searchQuery: '',
      layoutMode: this.getLayoutMode(tabs.length),
    }

    if (!this.dialogEl) {
      this.createDialog()
    }

    this.render()
  }

  advance(direction: 1 | -1) {
    if (this._state.phase !== 'open') return

    const len = this._state.tabs.length
    if (len === 0) {
      this.dismiss()
      return
    }

    const next = (this._state.selectedIndex + direction + len) % len
    this._state = { ...this._state, selectedIndex: next }
    this.render()
  }

  setSearchQuery(query: string) {
    if (this._state.phase !== 'open') return
    this._state = { ...this._state, searchQuery: query }
    this.render()
  }

  async select() {
    if (this._state.phase !== 'open') return

    const tab = this._state.tabs[this._state.selectedIndex]
    if (tab?.id && tab.windowId) {
      await sendSwitcherMessage('tab-switcher:switch', {
        tabId: tab.id,
        windowId: tab.windowId,
      })
    }

    this.dismiss()
  }

  async selectTab(index: number) {
    if (this._state.phase !== 'open') return
    this._state = { ...this._state, selectedIndex: index }
    await this.select()
  }

  dismiss() {
    if (this._state.phase === 'idle') return

    this._state = {
      phase: 'idle',
      tabs: [],
      selectedIndex: 0,
      thumbnails: {},
      searchQuery: '',
      layoutMode: 'strip',
    }

    if (this.root) {
      this.root.unmount()
      this.root = null
    }
    this.mountPoint = null

    if (this.dialogEl) {
      this.dialogEl.removeEventListener('click', this.handleDialogClick)
      this.dialogEl.close()
      this.dialogEl.remove()
      this.dialogEl = null
    }
  }

  private createDialog() {
    const dialog = document.createElement('dialog')
    dialog.style.cssText = [
      'all: unset',
      'display: flex',
      'position: fixed',
      'inset: 0',
      'width: 100vw',
      'height: 100vh',
      'max-width: 100vw',
      'max-height: 100vh',
      'border: none',
      'padding: 0',
      'margin: 0',
      'background: transparent',
      'overflow: hidden',
      'align-items: center',
      'justify-content: center',
    ].join('; ')

    const backdrop = document.createElement('div')
    backdrop.style.cssText = [
      'position: fixed',
      'inset: 0',
      'background: rgba(0, 0, 0, 0.6)',
      'backdrop-filter: blur(4px)',
      'z-index: 0',
      'animation: bos-backdrop-in 0.15s ease-out',
    ].join('; ')
    dialog.appendChild(backdrop)

    const host = document.createElement('div')
    host.style.cssText = [
      'position: relative',
      'z-index: 1',
      'display: flex',
      'align-items: center',
      'justify-content: center',
    ].join('; ')
    dialog.appendChild(host)

    const shadow = host.attachShadow({ mode: 'closed' })

    const styleEl = document.createElement('style')
    styleEl.textContent = [
      '#bos-switcher-root {',
      '  all: initial;',
      '  display: flex;',
      '  align-items: center;',
      '  justify-content: center;',
      '  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;',
      '  color: #fff;',
      '}',
      '@keyframes bos-backdrop-in {',
      '  from { opacity: 0; }',
      '  to { opacity: 1; }',
      '}',
      '@keyframes bos-card-enter {',
      '  from { opacity: 0; transform: translateY(16px) scale(0.85); }',
      '  to { opacity: 1; transform: translateY(0) scale(1); }',
      '}',
    ].join('\n')
    shadow.appendChild(styleEl)

    const mount = document.createElement('div')
    mount.id = 'bos-switcher-root'
    shadow.appendChild(mount)

    document.documentElement.appendChild(dialog)
    dialog.showModal()

    dialog.addEventListener('click', this.handleDialogClick)

    this.dialogEl = dialog
    this.mountPoint = mount

    if (!this.root) {
      this.root = createRoot(mount)
    }
  }

  private render() {
    if (!this.root || !this.mountPoint) return

    this.root.render(
      <TabSwitcherOverlay
        state={this._state}
        onSelectTab={(index) => this.selectTab(index)}
        onAdvance={(dir) => this.advance(dir)}
        onSelect={() => this.select()}
        onDismiss={() => this.dismiss()}
        onSearchChange={(query) => this.setSearchQuery(query)}
      />,
    )
  }
}
```

- [ ] **Step 2: Build to type-check**

```bash
cd packages/browseros-agent/apps/app
bun run typecheck
```

Expected: clean, no errors.

---

### Task 5: Build and verify

**Files:**
- No changes — run build and manual test

- [ ] **Step 1: Production build**

```bash
cd packages/browseros-agent/apps/app
bun --env-file=.env.development wxt build
```

Expected: Build succeeds. Output bundle contains new adaptive layout logic. No `jsxDEV` in output.

- [ ] **Step 2: Manual test — strip mode (≤7 tabs)**

Open 5 tabs. Press `Alt+Q`. Verify:
- Horizontal strip layout, cards centered, no visible scrollbar
- Selected card has orange glow border + slight scale-up
- Non-selected cards are slightly dimmed (opacity ~0.75)
- Title shows max 2 lines
- Footer shows "N / 5 tabs"
- Tab/Arrow keys cycle selection; Enter selects; Escape dismisses
- Alt release selects highlighted tab

- [ ] **Step 3: Manual test — grid mode (8–20 tabs)**

Open 12 tabs. Press `Alt+Q`. Verify:
- Grid layout with rows and columns
- All 12 cards visible without horizontal scrolling
- Arrow keys move spatially: Left/Right = adjacent, Up/Down = adjacent row
- Tab/Shift+Tab cycle MRU linearly
- Selected card has glow; non-selected is dimmed
- Thumbnails and favicons render correctly
- Footer shows "N / 12 tabs"

- [ ] **Step 4: Manual test — search mode (>20 tabs)**

Open 25 tabs. Press `Alt+Q`. Verify:
- Search input at top, auto-focused
- Type to filter by title or URL; results update live
- Arrow keys navigate filtered results
- Esc with text clears search; Esc without text closes overlay
- Footer shows "N / M (filtered from 25)" during search

- [ ] **Step 5: Edge cases**

- 0 tabs: overlay doesn't open (already handled)
- 1 tab: opens, shows single card (strip mode)
- 7 tabs: strip mode boundary
- 8 tabs: grid mode boundary
- 20 tabs: grid mode (no search)
- 21 tabs: grid mode + search

---

## Spec Coverage Check

| Requirement | Task |
|---|---|
| ≤7 tabs: horizontal strip, large previews, center selected | Task 3 (strip rendering + scrollIntoView) |
| >7 tabs: responsive grid layout | Task 3 (CSS Grid + dynamic columns) |
| >20 tabs: search input at top | Task 3 (showSearch conditional + search input) |
| Tab/Shift+Tab cycle MRU order | Task 3 (handleKeyDown, Tab case → onAdvance) |
| Arrow keys: spatial grid movement | Task 3 (getSpatialIndex — left/right/up/down) |
| Enter selects, Alt release selects, Esc closes | Already implemented, unchanged |
| Selected card: scale-up + glow | Task 2 (isSelected styling: box-shadow, transform) |
| Non-selected: dimmed | Task 2 (opacity: 0.75) |
| Two-line title clamping | Task 2 (WebkitLineClamp: 2) |
| Thumbnails and favicons preserved | Task 2 (unchanged from current) |
| Tab count display: "5 / 18 tabs" | Task 3 (footer) |
| Hybrid layout evaluation | Rejected in favor of pure grid (see analysis) |
| Responsiveness (laptop ↔ ultrawide) | Task 3 (ResizeObserver, dynamic columns, max-width) |
| No horizontal scrollbar | Task 3 (strip: `scrollbar-width: none` + CSS ; grid: `overflow: visible`) |
