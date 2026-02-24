# Phase 3: PWA & Offline Support

## Overview

Transforms the application into an installable Progressive Web App with offline-first capabilities. This phase ensures users can write, edit, and browse entries without an internet connection, with seamless background synchronization when connectivity returns.

**Status**: Not Started  
**Estimated Duration**: 2 weeks  
**Priority**: High  
**Blocked by**: Phase 2 (Editor & UI)

---

## Key Features

### Progressive Web App

- **Installable**: Add to Home Screen on mobile/desktop
- **Standalone Mode**: Runs like a native app
- **App Shell Architecture**: Instant load from cache
- **Auto-Updates**: Silent background updates
- **Update Prompts**: Notify users of new versions

### Offline-First Architecture

- **Local Storage**: IndexedDB for entry persistence
- **Optimistic UI**: Immediate feedback, background sync
- **Conflict Resolution**: Handle simultaneous edits
- **Queue System**: Pending operations when offline
- **Background Sync**: Automatic sync when online

### Data Synchronization

- **Bidirectional Sync**: Client ↔ Server
- **Differential Sync**: Only changed data
- **Conflict Detection**: Timestamp-based resolution
- **Retry Logic**: Exponential backoff for failures
- **Sync Status**: Visual indicators

---

## Dependencies

### New Dependencies

```bash
bun add -D vite-plugin-pwa
bun add workbox-window
bun add idb
bun add -D @types/workbox-window
```

| Package | Purpose | Size |
|---------|---------|------|
| `vite-plugin-pwa` | Vite PWA configuration | Dev only |
| `workbox-window` | Service Worker registration | ~5KB |
| `idb` | IndexedDB promise wrapper | ~3KB |

**Total Bundle Impact**: ~8KB (+ Service Worker generated at build time)

### Existing Dependencies (From Previous Phases)

- `comlink` - Worker communication (Phase 1)
- `better-auth` - Authentication (existing)
- `drizzle-orm` - Database ORM (existing)

---

## Technical Architecture

### PWA Configuration

```typescript
// vite.config.ts
import { VitePWA } from 'vite-plugin-pwa';

export default {
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      injectRegister: 'auto',
      
      manifest: {
        name: 'Journal',
        short_name: 'Journal',
        description: 'Your private, encrypted journal',
        theme_color: '#FAF9F6',
        background_color: '#FAF9F6',
        display: 'standalone',
        orientation: 'portrait',
        scope: '/',
        start_url: '/',
        icons: [
          {
            src: '/icon-192.png',
            sizes: '192x192',
            type: 'image/png'
          },
          {
            src: '/icon-512.png',
            sizes: '512x512',
            type: 'image/png',
            purpose: 'any maskable'
          }
        ]
      },
      
      workbox: {
        globPatterns: [
          '**/*.{js,css,html,ico,png,svg,woff2}'
        ],
        
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/fonts\.googleapis\.com\/.*/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'google-fonts-cache',
              expiration: {
                maxEntries: 10,
                maxAgeSeconds: 60 * 60 * 24 * 365 // 1 year
              }
            }
          },
          {
            urlPattern: /^https:\/\/fonts\.gstatic\.com\/.*/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'google-fonts-cache',
              expiration: {
                maxEntries: 10,
                maxAgeSeconds: 60 * 60 * 24 * 365
              }
            }
          }
        ],
        
        // Don't cache API calls - let app handle offline
        navigateFallback: '/index.html',
        navigateFallbackDenylist: [/^\/api/]
      }
    })
  ]
};
```

### Service Worker Strategy

```
Service Worker Lifecycle:
┌─────────────────┐
│   Install       │── Cache static assets (app shell)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Activate      │── Clean old caches, take control
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Fetch Handler │── Serve from cache or network
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Sync Handler  │── Background sync when online
└─────────────────┘
```

### Offline Data Flow

```
User Action (Offline):
┌─────────────────┐
│  Create Entry   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Encrypt Entry   │── Web Worker
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Save to         │── IndexedDB
│ IndexedDB       │   (local)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Queue for Sync  │── sync_queue table
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Show Optimistic │── UI updates immediately
│ UI Update       │
└─────────────────┘

Connection Restored:
┌─────────────────┐
│ Detect Online   │── 'online' event
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Trigger         │── Background Sync API
│ Background Sync │   or polling fallback
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Process Queue   │── Sequential or parallel
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Send to Server  │── API calls
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Update Status   │── Mark as synced
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Notify User     │── Success or conflict
└─────────────────┘
```

### IndexedDB Schema

```typescript
// IndexedDB stores (client-side)
interface JournalDB {
  // Store: entries
  // Key: entry id
  entries: {
    id: string;
    encryptedContent: string;
    encryptedTitle: string | null;
    nonce: string;
    salt: string;
    wordCount: number;
    mood: number | null;
    tags: string[];
    isDeleted: boolean;
    createdAt: string; // ISO date
    updatedAt: string;
    // Sync metadata
    syncStatus: 'synced' | 'pending' | 'error';
    lastSyncedAt: string | null;
  };
  
  // Store: syncQueue
  // Key: auto-increment
  syncQueue: {
    id: number;
    operation: 'create' | 'update' | 'delete';
    entityType: 'entry';
    entityId: string;
    encryptedPayload: string;
    timestamp: string;
    retryCount: number;
  };
  
  // Store: appState
  // Key: state key
  appState: {
    key: string;
    value: any;
  };
}
```

### Conflict Resolution Strategy

**Approach**: Last-Write-Wins with Optional Manual Merge

```typescript
interface ConflictResolution {
  // Automatic resolution (default)
  strategy: 'last-write-wins';
  
  // When conflicts detected
  // 1. Compare timestamps
  // 2. Keep most recent version
  // 3. Log conflict for user review
}

// Future enhancement:
interface ManualMerge {
  strategy: 'manual-merge';
  // Show both versions side-by-side
  // User selects which to keep
  // Or edits merged version
}
```

---

## Success Metrics

### PWA Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Lighthouse PWA | 100 | Lighthouse |
| Installable | Yes | Chrome DevTools |
| Offline functionality | 100% core features | Manual test |
| App shell load time | <1s | Lighthouse |
| Service Worker registration | 100% success | Analytics |

### Offline Metrics

| Metric | Target | Test |
|--------|--------|------|
| Write entry offline | 100% success | Manual test |
| Edit entry offline | 100% success | Manual test |
| Read entries offline | 100% success | Manual test |
| Data persistence | No data loss | Recovery test |

### Sync Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Sync success rate | >99% | Unit tests |
| Sync latency | <5s for <50 items | Performance test |
| Conflict rate | <1% | Monitoring |
| Retry success | >95% after 3 attempts | Stress test |
| Storage quota usage | <50% warning | Monitoring |

### Performance Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Time to First Byte (TTFB) | <200ms | WebPageTest |
| First Contentful Paint | <1s | Lighthouse |
| Largest Contentful Paint | <2.5s | Lighthouse |
| IndexedDB query time | <50ms | Benchmark |
| Bundle size increase | <50KB | Bundle analyzer |

---

## Test Requirements

### Unit Tests

```typescript
// tests/lib/storage.test.ts
describe('IndexedDB Storage', () => {
  describe('entries', () => {
    it('saves entry to IndexedDB', async () => {
      const entry = createMockEntry();
      await saveEntry(entry);
      const retrieved = await getEntry(entry.id);
      expect(retrieved).toEqual(entry);
    });
    
    it('retrieves all entries', async () => {
      const entries = [entry1, entry2, entry3];
      await Promise.all(entries.map(saveEntry));
      const all = await getAllEntries();
      expect(all).toHaveLength(3);
    });
    
    it('updates existing entry', async () => {
      const entry = createMockEntry();
      await saveEntry(entry);
      entry.wordCount = 100;
      await updateEntry(entry);
      const retrieved = await getEntry(entry.id);
      expect(retrieved.wordCount).toBe(100);
    });
    
    it('soft deletes entry', async () => {
      const entry = createMockEntry();
      await saveEntry(entry);
      await deleteEntry(entry.id);
      const retrieved = await getEntry(entry.id);
      expect(retrieved.isDeleted).toBe(true);
    });
    
    it('handles large entries (>1MB)', async () => {
      const entry = createMockEntry({ content: 'x'.repeat(1000000) });
      await saveEntry(entry);
      const retrieved = await getEntry(entry.id);
      expect(retrieved).toBeDefined();
    });
    
    it('encrypts data before storage', async () => {
      const entry = createMockEntry();
      await saveEntry(entry);
      // Verify raw IndexedDB data is encrypted
      const raw = await getRawEntryFromDB(entry.id);
      expect(isEncrypted(raw.content)).toBe(true);
    });
  });

  describe('syncQueue', () => {
    it('queues operation when offline', async () => {
      mockOffline();
      const operation = createOperation('create');
      await queueOperation(operation);
      const queue = await getSyncQueue();
      expect(queue).toHaveLength(1);
    });
    
    it('processes queue in FIFO order', async () => {
      const op1 = createOperation('create', { id: '1' });
      const op2 = createOperation('update', { id: '2' });
      await queueOperation(op1);
      await queueOperation(op2);
      const queue = await getSyncQueue();
      expect(queue[0].entityId).toBe('1');
      expect(queue[1].entityId).toBe('2');
    });
    
    it('removes operation after successful sync', async () => {
      mockOnline();
      mockApiSuccess();
      const operation = createOperation('create');
      await queueOperation(operation);
      await processSyncQueue();
      const queue = await getSyncQueue();
      expect(queue).toHaveLength(0);
    });
    
    it('increments retry count on failure', async () => {
      mockOnline();
      mockApiFailure();
      const operation = createOperation('create');
      await queueOperation(operation);
      await processSyncQueue();
      const queue = await getSyncQueue();
      expect(queue[0].retryCount).toBe(1);
    });
    
    it('marks operation as failed after max retries', async () => {
      mockOnline();
      mockApiFailure();
      const operation = createOperation('create', { retryCount: 2 });
      await queueOperation(operation);
      await processSyncQueue();
      const queue = await getSyncQueue();
      expect(queue[0].status).toBe('failed');
    });
  });

  describe('storage limits', () => {
    it('warns at 50% storage quota', async () => {
      mockStorageQuota({ used: 50, total: 100 });
      const warning = await checkStorageQuota();
      expect(warning.level).toBe('warning');
    });
    
    it('errors at 80% storage quota', async () => {
      mockStorageQuota({ used: 80, total: 100 });
      const warning = await checkStorageQuota();
      expect(warning.level).toBe('error');
    });
  });
});
```

### Integration Tests

```typescript
// tests/integration/offline.test.ts
describe('Offline Functionality', () => {
  beforeEach(() => {
    // Clear IndexedDB
    // Reset network mocks
  });

  it('saves entry when offline', async () => {
    goOffline();
    
    const { getByRole } = render(<Editor />);
    const editor = getByRole('textbox');
    
    fireEvent.change(editor, { target: { value: 'Test entry' } });
    fireEvent.click(getByRole('button', { name: /save/i }));
    
    // Verify saved to IndexedDB
    const entries = await getAllEntries();
    expect(entries).toHaveLength(1);
    expect(entries[0].encryptedContent).toBeDefined();
    
    // Verify queued for sync
    const queue = await getSyncQueue();
    expect(queue).toHaveLength(1);
  });

  it('syncs queued entries when coming online', async () => {
    // Create entry while offline
    goOffline();
    await createEntry('Offline entry');
    
    // Come back online
    goOnline();
    
    // Wait for sync
    await waitFor(() => {
      return getSyncQueue().then(queue => queue.length === 0);
    }, { timeout: 5000 });
    
    // Verify synced to server
    const serverEntries = await fetchServerEntries();
    expect(serverEntries).toHaveLength(1);
  });

  it('shows sync status indicator', async () => {
    goOffline();
    await createEntry('Test');
    
    const { getByTestId } = render(<App />);
    
    // Should show pending indicator
    expect(getByTestId('sync-status')).toHaveTextContent('Pending');
    
    goOnline();
    
    // Should show syncing then synced
    await waitFor(() => {
      expect(getByTestId('sync-status')).toHaveTextContent('Synced');
    });
  });

  it('handles edit conflicts', async () => {
    // Create entry
    const entry = await createEntry('Original');
    
    // Edit offline
    goOffline();
    await editEntry(entry.id, 'Offline edit');
    
    // Simulate server edit (different user or device)
    await simulateServerEdit(entry.id, 'Server edit');
    
    // Come online and sync
    goOnline();
    await processSyncQueue();
    
    // Should resolve conflict (last write wins)
    const finalEntry = await getEntry(entry.id);
    expect(finalEntry.encryptedContent).toBe('Offline edit');
  });

  it('preserves data across page reloads', async () => {
    goOffline();
    await createEntry('Persistent entry');
    
    // Reload page
    window.location.reload();
    
    // Verify entry still exists
    const entries = await getAllEntries();
    expect(entries).toHaveLength(1);
  });
});

// tests/integration/pwa.test.ts
describe('PWA Functionality', () => {
  it('registers service worker', async () => {
    render(<App />);
    
    await waitFor(() => {
      expect(navigator.serviceWorker.controller).toBeDefined();
    });
  });

  it('shows install prompt when criteria met', async () => {
    mockBeforeInstallPrompt();
    
    const { getByRole } = render(<App />);
    
    await waitFor(() => {
      expect(getByRole('button', { name: /install/i })).toBeInTheDocument();
    });
  });

  it('caches assets for offline use', async () => {
    // Load app
    render(<App />);
    
    // Wait for service worker installation
    await waitForServiceWorkerInstalled();
    
    // Verify caches created
    const caches = await window.caches.keys();
    expect(caches).toContain('workbox-precache');
  });

  it('serves from cache when offline', async () => {
    // Load and cache
    render(<App />);
    await waitForServiceWorkerInstalled();
    
    // Go offline
    goOffline();
    
    // Reload
    window.location.reload();
    
    // App should still load
    expect(document.querySelector('#root')).toBeInTheDocument();
  });

  it('notifies of available update', async () => {
    // Mock new version available
    mockNewVersionAvailable();
    
    const { getByRole } = render(<App />);
    
    await waitFor(() => {
      expect(getByRole('button', { name: /update/i })).toBeInTheDocument();
    });
  });
});
```

### End-to-End Tests

```typescript
// tests/e2e/offline-workflow.test.ts
describe('Complete Offline Workflow', () => {
  it('user can write offline and sync later', async () => {
    // 1. Install PWA
    await page.goto('/');
    await page.evaluate(() => {
      window.dispatchEvent(new Event('beforeinstallprompt'));
    });
    await page.click('[data-testid="install-button"]');
    
    // 2. Go offline
    await page.setOfflineMode(true);
    
    // 3. Create entry
    await page.click('[data-testid="new-entry-button"]');
    await page.fill('[data-testid="editor"]', 'My offline thoughts');
    await page.click('[data-testid="mood-4"]');
    await page.click('[data-testid="save-button"]');
    
    // 4. Verify offline indicator
    await expect(page.locator('[data-testid="offline-badge"]')).toBeVisible();
    
    // 5. Create another entry
    await page.click('[data-testid="new-entry-button"]');
    await page.fill('[data-testid="editor"]', 'More offline thoughts');
    await page.click('[data-testid="save-button"]');
    
    // 6. View entries list (should work offline)
    await page.click('[data-testid="entries-tab"]');
    await expect(page.locator('[data-testid="entry-card"]')).toHaveCount(2);
    
    // 7. Go online
    await page.setOfflineMode(false);
    
    // 8. Wait for sync
    await expect(page.locator('[data-testid="syncing-indicator"]')).toBeVisible();
    await expect(page.locator('[data-testid="synced-indicator"]')).toBeVisible({
      timeout: 10000
    });
    
    // 9. Refresh and verify persistence
    await page.reload();
    await expect(page.locator('[data-testid="entry-card"]')).toHaveCount(2);
  });

  it('handles conflict resolution', async () => {
    // Setup: Create entry while online
    await page.goto('/');
    await page.click('[data-testid="new-entry-button"]');
    await page.fill('[data-testid="editor"]', 'Original content');
    await page.click('[data-testid="save-button"]');
    await expect(page.locator('[data-testid="synced-indicator"]')).toBeVisible();
    
    // Simulate server-side edit
    await simulateServerEdit('Server modified content');
    
    // Go offline and edit
    await page.setOfflineMode(true);
    await page.click('[data-testid="edit-button"]');
    await page.fill('[data-testid="editor"]', 'Offline modified content');
    await page.click('[data-testid="save-button"]');
    
    // Go online
    await page.setOfflineMode(false);
    
    // Verify conflict resolution (last write wins)
    await page.reload();
    await expect(page.locator('[data-testid="entry-content"]')).toContainText(
      'Offline modified content'
    );
  });
});
```

### Performance Tests

```typescript
// tests/performance/sync.test.ts
describe('Sync Performance', () => {
  it('syncs 50 queued items in <5 seconds', async () => {
    // Create 50 operations
    const operations = Array.from({ length: 50 }, (_, i) => ({
      operation: 'create',
      entityId: `entry-${i}`,
      encryptedPayload: generateEncryptedPayload()
    }));
    
    await Promise.all(operations.map(queueOperation));
    
    const start = performance.now();
    await processSyncQueue();
    const duration = performance.now() - start;
    
    expect(duration).toBeLessThan(5000);
  });

  it('handles 1000 entries in IndexedDB', async () => {
    // Create 1000 entries
    const entries = Array.from({ length: 1000 }, createMockEntry);
    await Promise.all(entries.map(saveEntry));
    
    const start = performance.now();
    const all = await getAllEntries();
    const duration = performance.now() - start;
    
    expect(all).toHaveLength(1000);
    expect(duration).toBeLessThan(1000);
  });
});
```

---

## Implementation Checklist

### PWA Configuration

- [ ] Vite PWA plugin setup
  - [ ] Install `vite-plugin-pwa`
  - [ ] Configure manifest.json generation
  - [ ] Set up Workbox configuration
  - [ ] Define runtime caching strategies
  
- [ ] App icons
  - [ ] Generate 192x192 icon
  - [ ] Generate 512x512 icon
  - [ ] Generate maskable icons
  - [ ] Add favicon variants
  
- [ ] Service Worker registration
  - [ ] Register in main.tsx
  - [ ] Handle update logic
  - [ ] Implement update prompts
  
- [ ] Install prompts
  - [ ] Detect beforeinstallprompt event
  - [ ] Show custom install UI
  - [ ] Handle install completion

### Offline Storage

- [ ] IndexedDB setup
  - [ ] Install `idb` library
  - [ ] Create database wrapper
  - [ ] Define object stores
  - [ ] Create indexes for queries
  
- [ ] Entry storage
  - [ ] `saveEntry()` - Create/update
  - [ ] `getEntry()` - Retrieve single
  - [ ] `getAllEntries()` - List all
  - [ ] `deleteEntry()` - Soft delete
  - [ ] `searchEntries()` - Client-side search
  
- [ ] Encryption integration
  - [ ] Encrypt before storing locally
  - [ ] Decrypt after retrieval
  - [ ] Handle encryption errors

### Sync System

- [ ] Sync queue management
  - [ ] `queueOperation()` - Add to queue
  - [ ] `getSyncQueue()` - Retrieve queue
  - [ ] `removeFromQueue()` - Remove after sync
  - [ ] `updateQueueStatus()` - Mark status
  
- [ ] Network detection
  - [ ] Listen for online/offline events
  - [ ] Provide network status hook
  - [ ] Visual offline indicator
  
- [ ] Background sync
  - [ ] Use Background Sync API (if available)
  - [ ] Fallback to periodic sync polling
  - [ ] Process queue when online
  
- [ ] Sync process
  - [ ] Sequential or parallel processing
  - [ ] Error handling and retry
  - [ ] Conflict detection
  - [ ] Success/failure callbacks

### UI Components

- [ ] Network status
  - [ ] `OfflineIndicator` - Banner when offline
  - [ ] `SyncStatus` - Pending/syncing/synced indicator
  
- [ ] PWA components
  - [ ] `InstallPrompt` - Install app UI
  - [ ] `UpdatePrompt` - New version available
  - [ ] `OfflineReady` - Cache ready notification

### Conflict Resolution

- [ ] Conflict detection
  - [ ] Compare timestamps
  - [ ] Detect concurrent edits
  - [ ] Flag conflicts for review
  
- [ ] Resolution strategies
  - [ ] Implement last-write-wins (default)
  - [ ] Log conflicts for audit
  - [ ] Optional manual merge UI

### Testing

- [ ] Unit tests
  - [ ] Storage functions
  - [ ] Sync queue operations
  - [ ] Conflict resolution
  
- [ ] Integration tests
  - [ ] Offline/online transitions
  - [ ] Sync flows
  - [ ] PWA installation
  
- [ ] E2E tests
  - [ ] Complete offline workflow
  - [ ] Conflict scenarios
  - [ ] PWA lifecycle
  
- [ ] Performance tests
  - [ ] Storage limits
  - [ ] Sync speed
  - [ ] Cache efficiency

---

## Deliverables

### Configuration Files

| File | Purpose |
|------|---------|
| `vite.config.ts` (updated) | PWA plugin configuration |
| `public/manifest.json` | Web app manifest |
| `public/icon-192.png` | App icon (192x192) |
| `public/icon-512.png` | App icon (512x512) |
| `public/icon-maskable.png` | Maskable icon |

### Code Files

| File | Purpose |
|------|---------|
| `src/lib/storage.ts` | IndexedDB wrapper |
| `src/lib/sync.ts` | Sync logic and queue management |
| `src/lib/network.ts` | Network status utilities |
| `src/workers/sync.worker.ts` | Background sync worker |
| `src/hooks/useOffline.ts` | Offline status hook |
| `src/hooks/useSync.ts` | Sync status and control |
| `src/hooks/useStorage.ts` | Storage operations hook |
| `src/components/pwa/InstallPrompt.tsx` | Install app UI |
| `src/components/pwa/UpdatePrompt.tsx` | Update available UI |
| `src/components/pwa/OfflineIndicator.tsx` | Offline status banner |
| `src/components/sync/SyncStatus.tsx` | Sync state indicator |
| `src/contexts/SyncContext.tsx` | Global sync state |

### Test Files

| File | Coverage |
|------|----------|
| `tests/lib/storage.test.ts` | IndexedDB operations |
| `tests/lib/sync.test.ts` | Sync logic |
| `tests/integration/offline.test.ts` | Offline functionality |
| `tests/integration/pwa.test.ts` | PWA features |
| `tests/e2e/offline-workflow.test.ts` | Complete workflow |
| `tests/performance/sync.test.ts` | Performance benchmarks |

---

## File Structure

```
src/
├── lib/
│   ├── storage.ts          # IndexedDB operations
│   ├── sync.ts             # Sync queue management
│   └── network.ts          # Network utilities
├── workers/
│   └── sync.worker.ts      # Background sync worker
├── hooks/
│   ├── useOffline.ts       # Network status
│   ├── useSync.ts          # Sync operations
│   └── useStorage.ts       # Storage operations
├── components/
│   ├── pwa/
│   │   ├── InstallPrompt.tsx
│   │   ├── UpdatePrompt.tsx
│   │   └── OfflineIndicator.tsx
│   └── sync/
│       └── SyncStatus.tsx
├── contexts/
│   └── SyncContext.tsx     # Global sync state
└── routes/
    └── __root.tsx          # PWA registration

public/
├── manifest.json           # Web app manifest
├── icon-192.png
├── icon-512.png
└── icon-maskable.png
```

---

## Dependencies on Previous Phases

| Dependency | From | Usage |
|------------|------|-------|
| Encryption utilities | Phase 1 | Encrypt before local storage |
| Entry API | Phase 1 | Sync with server |
| Editor component | Phase 2 | Offline editing |
| Entry list | Phase 2 | Offline browsing |
| Theme system | Phase 2 | PWA theming |

---

## Browser Support

### Required APIs

| API | Support | Fallback |
|-----|---------|----------|
| Service Workers | Chrome 45+, FF 44+, Safari 11.3+ | None (graceful degradation) |
| IndexedDB | All modern browsers | localStorage (limited) |
| Background Sync | Chrome 49+, FF 75+ | Polling fallback |
| Cache API | Chrome 43+, FF 41+, Safari 11.1+ | Network only |

### Storage Quotas

| Browser | Typical Quota |
|---------|---------------|
| Chrome | ~60% of available disk |
| Firefox | 2GB or 50% of disk |
| Safari | ~1GB (prompts after 200MB) |

---

## Known Challenges

### Storage Limits

- Safari: Limited to ~1GB, prompts user after 200MB
- Solution: Monitor usage, warn at 50%, compress data

### Background Sync Limitations

- Not available in all browsers
- Solution: Polling fallback with reasonable intervals

### Conflict Resolution Complexity

- Multiple devices editing simultaneously
- Solution: Start with last-write-wins, add merge UI later

### Service Worker Updates

- Cache invalidation on updates
- Solution: Workbox handles this, implement update prompts

---

## Resources

- [Workbox Documentation](https://developer.chrome.com/docs/workbox/)
- [PWA Checklist](https://web.dev/pwa-checklist/)
- [Background Sync API](https://web.dev/background-sync/)
- [IndexedDB Best Practices](https://dexie.org/docs/Tutorial/Best-Practices)
- [Service Worker Lifecycle](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle)
