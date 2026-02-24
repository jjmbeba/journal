# Phase 1: Foundation & Encryption

## Overview

Establishes the core security architecture and data layer. This phase implements client-side encryption, database schema, authentication flow, and data fetching infrastructure using **TanStack Start Server Functions** and **TanStack Query**.

**Status**: Not Started  
**Estimated Duration**: 2 weeks  
**Priority**: Critical (blocks all other phases)

---

## Key Features

### Client-Side Encryption
- **AES-256-GCM** encryption using Web Crypto API
- **PBKDF2** key derivation with 100,000+ iterations
- Unique nonce for each encryption operation
- Memory-only key storage (never persisted)
- **Zero-knowledge architecture**: Server never sees plaintext

### TanStack Start Server Functions
- Type-safe RPC between client and server
- No manual API route handlers
- Automatic request validation with Zod
- End-to-end TypeScript inference
- Better Auth session integration

### TanStack Query Integration
- Server state management with caching
- Optimistic updates for mutations
- Background refetching
- Loading/error states
- Perfect foundation for offline sync (Phase 3)

### Database Schema
- Extended tables for journal entries, settings, and sync queue
- Encrypted content storage (no plaintext in database)
- Proper indexing for performance

### Authentication Integration
- Leverage existing Better Auth setup
- Master password flow for encryption key derivation
- Session timeout and auto-lock mechanisms

---

## Dependencies

### New Dependencies

```bash
# Core
bun add @tanstack/react-query
bun add comlink

# Development
bun add -D @types/webcrypto
```

| Package | Purpose | Size |
|---------|---------|------|
| `@tanstack/react-query` | Server state management | ~40KB |
| `comlink` | Web Worker communication | ~15KB |
| `@types/webcrypto` | TypeScript definitions | Dev only |

### Existing Dependencies (Already Configured)

- `@tanstack/react-start` - Full-stack framework (includes `createServerFn`)
- `better-auth` - Authentication framework
- `drizzle-orm` - Database ORM
- `@libsql/client` - SQLite client
- `zod` - Schema validation

---

## Technical Architecture

### Server Functions Architecture

**TanStack Start Server Functions** provide type-safe RPC calls between client and server without manual API routes.

```typescript
// Server Function Definition (runs on server)
// src/lib/server/entries.ts
import { createServerFn } from '@tanstack/react-start';
import { z } from 'zod';

export const createEntry = createServerFn({ method: 'POST' })
  .validator(z.object({
    encryptedContent: z.string(),
    encryptedTitle: z.string().optional(),
    nonce: z.string(),
    salt: z.string(),
    wordCount: z.number(),
    mood: z.number().min(1).max(5).optional(),
    tags: z.array(z.string()).optional()
  }))
  .handler(async ({ data, context }) => {
    // Server-side authentication check
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    // Server NEVER decrypts - stores encrypted blob only
    const entry = await db.insert(journalEntry).values({
      userId: session.user.id,
      ...data
    }).returning();
    
    return entry[0];
  });

// Client Usage (type-safe!)
// src/hooks/useEntries.ts
import { useMutation } from '@tanstack/react-query';
import { createEntry } from '@/lib/server/entries';

export function useCreateEntry() {
  return useMutation({
    mutationFn: createEntry,
    onSuccess: () => {
      // Invalidate and refetch entries list
      queryClient.invalidateQueries({ queryKey: ['entries'] });
    }
  });
}
```

### Encryption Flow

```
User enters master password
         ↓
PBKDF2(password, salt, 100000 iterations)
         ↓
256-bit AES key (stored in memory only)
         ↓
Encrypt(plaintext, key) → { ciphertext, nonce, salt }
         ↓
Send to server function (still encrypted!)
         ↓
Server stores encrypted blob in database
```

### Decryption Flow

```
User requests entries (via useQuery)
         ↓
Server function returns encrypted data
         ↓
TanStack Query caches encrypted data
         ↓
Client decrypts with in-memory key
         ↓
Display plaintext (never stored)
```

### Key Management

| Aspect | Implementation |
|--------|----------------|
| Key Derivation | PBKDF2 with 100k iterations |
| Algorithm | AES-256-GCM |
| Key Storage | Memory only (JavaScript variable) |
| Key Lifecycle | Derived on unlock, cleared on lock/logout |
| Auto-lock | Configurable: 5min/15min/30min/never |

---

## Database Schema

### New Tables

```typescript
// journalEntry - Stores encrypted journal entries
{
  id: string (primary key)
  userId: string (foreign key → user.id)
  encryptedContent: string (base64 encoded ciphertext)
  encryptedTitle: string | null (optional encrypted title)
  nonce: string (unique per encryption)
  salt: string (key derivation salt)
  wordCount: number (plaintext metadata for stats)
  mood: number | null (1-5 scale)
  tags: string[] (JSON array)
  isDeleted: boolean (soft delete flag)
  deletedAt: Date | null
  createdAt: Date
  updatedAt: Date
}

// userSettings - User preferences and configuration
{
  userId: string (primary key, foreign key)
  theme: 'light' | 'dark' | 'oled'
  accentColor: string (hex color)
  fontSize: number (default: 16)
  lineHeight: number (default: 1.6)
  autosaveInterval: number (seconds, default: 30)
  encryptionEnabled: boolean (default: true)
  autoLockDuration: number | null (minutes, null = never)
  createdAt: Date
  updatedAt: Date
}

// syncQueue - Pending operations for offline sync
{
  id: string (primary key)
  userId: string (foreign key)
  operation: 'create' | 'update' | 'delete'
  entityType: 'entry' | 'settings'
  entityId: string
  encryptedPayload: string
  timestamp: Date
  status: 'pending' | 'synced' | 'failed'
  retryCount: number
}
```

### Indexes

```sql
-- Performance indexes
CREATE INDEX idx_entry_userId ON journal_entry(userId);
CREATE INDEX idx_entry_createdAt ON journal_entry(createdAt);
CREATE INDEX idx_entry_isDeleted ON journal_entry(isDeleted);
CREATE INDEX idx_sync_userId ON syncQueue(userId);
CREATE INDEX idx_sync_status ON syncQueue(status);
```

---

## Server Functions

### Entry Server Functions

```typescript
// src/lib/server/entries.ts
import { createServerFn } from '@tanstack/react-start';
import { z } from 'zod';
import { db } from '@/db';
import { journalEntry } from '@/db/schema';
import { getSession } from './auth';

// Validation schemas
const createEntrySchema = z.object({
  encryptedContent: z.string().min(1),
  encryptedTitle: z.string().optional(),
  nonce: z.string().min(1),
  salt: z.string().min(1),
  wordCount: z.number().int().min(0),
  mood: z.number().int().min(1).max(5).optional(),
  tags: z.array(z.string()).optional()
});

const updateEntrySchema = z.object({
  id: z.string().uuid(),
  encryptedContent: z.string().optional(),
  encryptedTitle: z.string().optional(),
  nonce: z.string().optional(),
  salt: z.string().optional(),
  wordCount: z.number().int().min(0).optional(),
  mood: z.number().int().min(1).max(5).optional(),
  tags: z.array(z.string()).optional()
});

// Create entry
export const createEntry = createServerFn({ method: 'POST' })
  .validator(createEntrySchema)
  .handler(async ({ data, context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const entry = await db.insert(journalEntry).values({
      userId: session.user.id,
      ...data
    }).returning();
    
    return entry[0];
  });

// Get all entries (returns encrypted data)
export const getEntries = createServerFn({ method: 'GET' })
  .handler(async ({ context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const entries = await db.query.journalEntry.findMany({
      where: (entry, { eq, and }) => and(
        eq(entry.userId, session.user.id),
        eq(entry.isDeleted, false)
      ),
      orderBy: (entry, { desc }) => desc(entry.createdAt)
    });
    
    return entries;
  });

// Get single entry
export const getEntry = createServerFn({ method: 'GET' })
  .validator(z.object({ id: z.string().uuid() }))
  .handler(async ({ data, context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const entry = await db.query.journalEntry.findFirst({
      where: (entry, { eq, and }) => and(
        eq(entry.id, data.id),
        eq(entry.userId, session.user.id)
      )
    });
    
    if (!entry) throw new Error('Entry not found');
    return entry;
  });

// Update entry
export const updateEntry = createServerFn({ method: 'POST' })
  .validator(updateEntrySchema)
  .handler(async ({ data, context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const { id, ...updates } = data;
    
    const entry = await db.update(journalEntry)
      .set({ ...updates, updatedAt: new Date() })
      .where(and(
        eq(journalEntry.id, id),
        eq(journalEntry.userId, session.user.id)
      ))
      .returning();
    
    if (!entry.length) throw new Error('Entry not found');
    return entry[0];
  });

// Soft delete entry
export const deleteEntry = createServerFn({ method: 'POST' })
  .validator(z.object({ id: z.string().uuid() }))
  .handler(async ({ data, context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const entry = await db.update(journalEntry)
      .set({ isDeleted: true, deletedAt: new Date() })
      .where(and(
        eq(journalEntry.id, data.id),
        eq(journalEntry.userId, session.user.id)
      ))
      .returning();
    
    if (!entry.length) throw new Error('Entry not found');
    return { success: true };
  });
```

### Settings Server Functions

```typescript
// src/lib/server/settings.ts
import { createServerFn } from '@tanstack/react-start';
import { z } from 'zod';
import { db } from '@/db';
import { userSettings } from '@/db/schema';
import { getSession } from './auth';

const settingsSchema = z.object({
  theme: z.enum(['light', 'dark', 'oled']).optional(),
  accentColor: z.string().regex(/^#[0-9A-Fa-f]{6}$/).optional(),
  fontSize: z.number().int().min(12).max(24).optional(),
  lineHeight: z.number().min(1).max(2).optional(),
  autosaveInterval: z.number().int().min(5).max(300).optional(),
  autoLockDuration: z.number().int().nullable().optional()
});

export const getSettings = createServerFn({ method: 'GET' })
  .handler(async ({ context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const settings = await db.query.userSettings.findFirst({
      where: eq(userSettings.userId, session.user.id)
    });
    
    // Return defaults if no settings exist
    return settings || {
      userId: session.user.id,
      theme: 'light',
      accentColor: '#8B7355',
      fontSize: 16,
      lineHeight: 1.6,
      autosaveInterval: 30,
      autoLockDuration: null
    };
  });

export const updateSettings = createServerFn({ method: 'POST' })
  .validator(settingsSchema)
  .handler(async ({ data, context }) => {
    const session = await getSession(context.request);
    if (!session) throw new Error('Unauthorized');
    
    const settings = await db.insert(userSettings)
      .values({
        userId: session.user.id,
        ...data
      })
      .onConflictDoUpdate({
        target: userSettings.userId,
        set: { ...data, updatedAt: new Date() }
      })
      .returning();
    
    return settings[0];
  });
```

### Authentication Helper

```typescript
// src/lib/server/auth.ts
import { auth } from '@/lib/auth';

export async function getSession(request: Request) {
  const session = await auth.api.getSession({
    headers: request.headers
  });
  return session;
}
```

---

## TanStack Query Integration

### Query Client Configuration

```typescript
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Data stays fresh for 5 minutes
      staleTime: 1000 * 60 * 5,
      // Keep unused data in cache for 30 minutes
      gcTime: 1000 * 60 * 30,
      // Retry failed queries 3 times
      retry: 3,
      retryDelay: (attemptIndex) => 
        Math.min(1000 * 2 ** attemptIndex, 30000),
      // Refetch on window focus (user returns to app)
      refetchOnWindowFocus: true,
      // Refetch when reconnecting (for offline support)
      refetchOnReconnect: true
    },
    mutations: {
      // Retry mutations twice
      retry: 2,
      // Optimistic by default for Phase 3
      networkMode: 'offlineFirst'
    }
  }
});
```

### React Query Hooks

```typescript
// src/hooks/useEntries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { 
  createEntry, 
  getEntries, 
  getEntry, 
  updateEntry, 
  deleteEntry 
} from '@/lib/server/entries';
import { useEncryption } from './useEncryption';

// Query keys for cache management
const entryKeys = {
  all: ['entries'] as const,
  lists: () => [...entryKeys.all, 'list'] as const,
  list: (filters: string) => [...entryKeys.lists(), filters] as const,
  details: () => [...entryKeys.all, 'detail'] as const,
  detail: (id: string) => [...entryKeys.details(), id] as const
};

// Hook to fetch all entries
export function useEntries() {
  const { decryptEntry } = useEncryption();
  
  return useQuery({
    queryKey: entryKeys.lists(),
    queryFn: getEntries,
    // Transform encrypted data to decrypted for UI
    select: (entries) => entries.map(decryptEntry)
  });
}

// Hook to fetch single entry
export function useEntry(id: string) {
  const { decryptEntry } = useEncryption();
  
  return useQuery({
    queryKey: entryKeys.detail(id),
    queryFn: () => getEntry({ data: { id } }),
    enabled: !!id,
    select: decryptEntry
  });
}

// Hook to create entry
export function useCreateEntry() {
  const queryClient = useQueryClient();
  const { encryptEntry } = useEncryption();
  
  return useMutation({
    mutationFn: async (plaintextData: CreateEntryInput) => {
      // Encrypt before sending to server
      const encryptedData = await encryptEntry(plaintextData);
      return createEntry({ data: encryptedData });
    },
    onSuccess: () => {
      // Invalidate entries list to refetch
      queryClient.invalidateQueries({ queryKey: entryKeys.lists() });
    }
  });
}

// Hook to update entry
export function useUpdateEntry() {
  const queryClient = useQueryClient();
  const { encryptEntry } = useEncryption();
  
  return useMutation({
    mutationFn: async ({ id, ...plaintextData }: UpdateEntryInput & { id: string }) => {
      const encryptedData = await encryptEntry(plaintextData);
      return updateEntry({ data: { id, ...encryptedData } });
    },
    onSuccess: (_, variables) => {
      // Invalidate specific entry and list
      queryClient.invalidateQueries({ queryKey: entryKeys.detail(variables.id) });
      queryClient.invalidateQueries({ queryKey: entryKeys.lists() });
    }
  });
}

// Hook to delete entry
export function useDeleteEntry() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (id: string) => deleteEntry({ data: { id } }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: entryKeys.lists() });
    }
  });
}
```

### Settings Hooks

```typescript
// src/hooks/useSettings.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { getSettings, updateSettings } from '@/lib/server/settings';

const settingsKeys = {
  all: ['settings'] as const
};

export function useSettings() {
  return useQuery({
    queryKey: settingsKeys.all,
    queryFn: getSettings
  });
}

export function useUpdateSettings() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: updateSettings,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.all });
    }
  });
}
```

### Query Provider Setup

```typescript
// src/providers/QueryProvider.tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/query-client';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

---

## Encryption Implementation

### Core Encryption Functions

```typescript
// src/lib/encryption.ts
import { generateSalt, generateNonce, deriveKey, encrypt, decrypt } from './crypto';

export interface EncryptedData {
  encryptedContent: string;
  encryptedTitle?: string;
  nonce: string;
  salt: string;
}

export interface EncryptionKey {
  key: CryptoKey;
  salt: string;
}

// Encrypt entry data before sending to server
export async function encryptEntry(
  data: { content: string; title?: string },
  masterKey: CryptoKey
): Promise<EncryptedData> {
  const salt = generateSalt();
  const nonce = generateNonce();
  
  const [encryptedContent, encryptedTitle] = await Promise.all([
    encrypt(data.content, masterKey, nonce),
    data.title ? encrypt(data.title, masterKey, nonce) : undefined
  ]);
  
  return {
    encryptedContent,
    encryptedTitle,
    nonce,
    salt
  };
}

// Decrypt entry data after receiving from server
export async function decryptEntry(
  entry: JournalEntry,
  masterKey: CryptoKey
): Promise<DecryptedEntry> {
  const [content, title] = await Promise.all([
    decrypt(entry.encryptedContent, masterKey, entry.nonce),
    entry.encryptedTitle 
      ? decrypt(entry.encryptedTitle, masterKey, entry.nonce)
      : Promise.resolve(null)
  ]);
  
  return {
    ...entry,
    content,
    title
  };
}
```

### React Hook for Encryption

```typescript
// src/hooks/useEncryption.ts
import { useCallback } from 'react';
import { useEncryptionContext } from '@/contexts/EncryptionContext';
import { encryptEntry as encryptEntryLib, decryptEntry as decryptEntryLib } from '@/lib/encryption';

export function useEncryption() {
  const { masterKey, isUnlocked } = useEncryptionContext();
  
  const encryptEntry = useCallback(
    async (data: { content: string; title?: string }) => {
      if (!masterKey) throw new Error('Encryption key not available');
      return encryptEntryLib(data, masterKey);
    },
    [masterKey]
  );
  
  const decryptEntry = useCallback(
    async (entry: JournalEntry) => {
      if (!masterKey) {
        // Return encrypted placeholder if locked
        return {
          ...entry,
          content: '[Locked]',
          title: entry.encryptedTitle ? '[Locked]' : null
        };
      }
      return decryptEntryLib(entry, masterKey);
    },
    [masterKey]
  );
  
  return {
    encryptEntry,
    decryptEntry,
    isUnlocked
  };
}
```

---

## Success Metrics

### Performance Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Encryption speed | <100ms for 1000 words | Benchmark test |
| Key derivation time | 100-500ms | Timing test |
| Decryption speed | <50ms for 1000 words | Benchmark test |
| Server function response | <200ms | API timing |
| TanStack Query cache hit | >90% | DevTools profiler |
| Memory footprint | <5MB for cached keys | Memory profiling |

### Security Metrics

| Metric | Target | Verification |
|--------|--------|--------------|
| Key uniqueness | 100% unique salts | Salt collision test |
| Encryption strength | AES-256-GCM | Algorithm verification |
| Key exposure | 0 persisted keys | Security audit |
| Server plaintext access | 0 (zero-knowledge) | Code review |
| Ciphertext uniqueness | 100% unique for same plaintext | Nonce verification |

### Reliability Metrics

| Metric | Target | Test |
|--------|--------|------|
| Roundtrip accuracy | 100% | Decrypt(Encrypt(x)) === x |
| Unicode support | 100% | Test all Unicode ranges |
| Large content | Support 50,000+ words | Stress test |
| Error handling | 100% graceful failures | Error injection tests |

---

## Test Requirements

### Unit Tests

```typescript
// tests/lib/encryption.test.ts

describe('Encryption', () => {
  describe('Key Derivation', () => {
    it('derives identical keys from same password and salt', async () => {
      // Same inputs → same key
    });
    
    it('derives different keys from same password with different salts', async () => {
      // Different salts → different keys
    });
    
    it('takes at least 100ms for security (100k iterations)', async () => {
      // Timing test for iteration count
    });
    
    it('handles Unicode passwords correctly', async () => {
      // Test international characters
    });
  });

  describe('Encryption/Decryption', () => {
    it('encrypts and decrypts text accurately', async () => {
      // Roundtrip test
    });
    
    it('generates unique ciphertexts for identical inputs', async () => {
      // Nonce uniqueness
    });
    
    it('fails decryption with wrong password', async () => {
      // Security test
    });
    
    it('handles special characters and emojis', async () => {
      // Unicode test
    });
    
    it('handles empty strings', async () => {
      // Edge case
    });
    
    it('handles very large texts (50k words)', async () => {
      // Performance edge case
    });
  });
});
```

### Server Function Tests

```typescript
// tests/server/entries.test.ts

describe('Entry Server Functions', () => {
  describe('createEntry', () => {
    it('creates entry with encrypted content', async () => {
      const data = {
        encryptedContent: 'encrypted...',
        nonce: 'nonce...',
        salt: 'salt...',
        wordCount: 100
      };
      
      const result = await createEntry({ data });
      expect(result.encryptedContent).toBe(data.encryptedContent);
      expect(result.userId).toBeDefined();
    });
    
    it('requires authentication', async () => {
      // Mock no session
      await expect(createEntry({ data: mockEntry })).rejects.toThrow('Unauthorized');
    });
    
    it('validates required fields', async () => {
      await expect(createEntry({ data: {} })).rejects.toThrow();
    });
  });

  describe('getEntries', () => {
    it('returns only current user entries', async () => {
      const entries = await getEntries();
      expect(entries.every(e => e.userId === currentUserId)).toBe(true);
    });
    
    it('excludes soft-deleted entries', async () => {
      const entries = await getEntries();
      expect(entries.every(e => !e.isDeleted)).toBe(true);
    });
    
    it('returns encrypted data only', async () => {
      const entries = await getEntries();
      // Verify no plaintext fields
      expect(entries[0]).not.toHaveProperty('content');
      expect(entries[0]).toHaveProperty('encryptedContent');
    });
  });
});
```

### React Query Hook Tests

```typescript
// tests/hooks/useEntries.test.tsx

describe('useEntries', () => {
  it('fetches and decrypts entries', async () => {
    const { result } = renderHook(() => useEntries(), {
      wrapper: createWrapper()
    });
    
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });
    
    expect(result.current.data).toBeDefined();
    expect(result.current.data[0]).toHaveProperty('content'); // Decrypted
  });
  
  it('caches entries', async () => {
    const { result, rerender } = renderHook(() => useEntries(), {
      wrapper: createWrapper()
    });
    
    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    
    // Should not refetch on re-render
    const spy = jest.spyOn(serverFunctions, 'getEntries');
    rerender();
    expect(spy).not.toHaveBeenCalled();
  });
});

describe('useCreateEntry', () => {
  it('encrypts before sending to server', async () => {
    const { result } = renderHook(() => useCreateEntry(), {
      wrapper: createWrapper()
    });
    
    await result.current.mutateAsync({
      content: 'Test content',
      title: 'Test title'
    });
    
    // Verify server received encrypted data
    expect(serverReceivedData.encryptedContent).toBeDefined();
    expect(serverReceivedData).not.toHaveProperty('content');
  });
  
  it('invalidates cache on success', async () => {
    const queryClient = createQueryClient();
    const invalidateSpy = jest.spyOn(queryClient, 'invalidateQueries');
    
    const { result } = renderHook(() => useCreateEntry(), {
      wrapper: ({ children }) => (
        <QueryClientProvider client={queryClient}>
          {children}
        </QueryClientProvider>
      )
    });
    
    await result.current.mutateAsync({ content: 'Test' });
    
    expect(invalidateSpy).toHaveBeenCalledWith({ queryKey: ['entries'] });
  });
});
```

---

## Implementation Checklist

### Core Encryption

- [ ] Web Crypto API wrapper
  - [ ] `deriveKey(password, salt)` function
  - [ ] `encrypt(plaintext, key, nonce)` function
  - [ ] `decrypt(ciphertext, key, nonce)` function
  - [ ] `generateSalt()` function
  - [ ] `generateNonce()` function
  
- [ ] Web Worker implementation
  - [ ] `src/workers/crypto.worker.ts`
  - [ ] Comlink integration
  - [ ] Non-blocking encryption/decryption
  
- [ ] Encryption context
  - [ ] `src/contexts/EncryptionContext.tsx`
  - [ ] Global encryption state
  - [ ] Master key management
  - [ ] Auto-lock timer

### Server Functions

- [ ] Auth helper
  - [ ] `src/lib/server/auth.ts` - Session retrieval
  
- [ ] Entry server functions
  - [ ] `src/lib/server/entries.ts`
  - [ ] `createEntry` with validation
  - [ ] `getEntries` (encrypted)
  - [ ] `getEntry` (encrypted)
  - [ ] `updateEntry` with validation
  - [ ] `deleteEntry` (soft delete)
  
- [ ] Settings server functions
  - [ ] `src/lib/server/settings.ts`
  - [ ] `getSettings`
  - [ ] `updateSettings`

### TanStack Query

- [ ] Query client
  - [ ] `src/lib/query-client.ts` - Configuration
  
- [ ] React hooks
  - [ ] `src/hooks/useEntries.ts` - Entries queries/mutations
  - [ ] `src/hooks/useEntry.ts` - Single entry query
  - [ ] `src/hooks/useSettings.ts` - Settings queries
  - [ ] `src/hooks/useEncryption.ts` - Encryption with React Query
  
- [ ] Provider
  - [ ] `src/providers/QueryProvider.tsx`
  - [ ] Integrate with root layout

### Database

- [ ] Migrations
  - [ ] `drizzle/0002_add_entries.sql`
  - [ ] `drizzle/0003_add_settings.sql`
  - [ ] `drizzle/0004_add_sync_queue.sql`

### UI Components

- [ ] Master password setup
  - [ ] `src/components/auth/MasterPasswordSetup.tsx`
  - [ ] Password strength indicator
  - [ ] Confirmation input
  
- [ ] Master password unlock
  - [ ] `src/components/auth/MasterPasswordUnlock.tsx`
  - [ ] Unlock modal
  - [ ] "Forgot password" warning
  
- [ ] Lock screen
  - [ ] `src/components/auth/LockScreen.tsx`

### Testing

- [ ] Unit tests
  - [ ] Encryption functions
  - [ ] Crypto utilities
  
- [ ] Server function tests
  - [ ] Entry CRUD operations
  - [ ] Settings operations
  - [ ] Auth integration
  
- [ ] React Query tests
  - [ ] Hook tests with MSW
  - [ ] Cache behavior
  - [ ] Mutation invalidation
  
- [ ] Integration tests
  - [ ] Encryption + server functions
  - [ ] End-to-end flow
  
- [ ] Security audit
  - [ ] Verify zero-knowledge
  - [ ] Check for plaintext leaks

---

## Deliverables

### Code Files

| File | Purpose |
|------|---------|
| `src/lib/crypto.ts` | Web Crypto API wrappers |
| `src/lib/encryption.ts` | High-level encryption/decryption |
| `src/lib/server/auth.ts` | Server-side auth helpers |
| `src/lib/server/entries.ts` | Entry server functions |
| `src/lib/server/settings.ts` | Settings server functions |
| `src/lib/query-client.ts` | TanStack Query configuration |
| `src/workers/crypto.worker.ts` | Web Worker for encryption |
| `src/hooks/useEncryption.ts` | Encryption React hook |
| `src/hooks/useEntries.ts` | Entries React Query hooks |
| `src/hooks/useSettings.ts` | Settings React Query hooks |
| `src/contexts/EncryptionContext.tsx` | Global encryption state |
| `src/providers/QueryProvider.tsx` | Query client provider |
| `src/components/auth/MasterPasswordSetup.tsx` | Setup UI |
| `src/components/auth/MasterPasswordUnlock.tsx` | Unlock UI |
| `src/components/auth/LockScreen.tsx` | Lock UI |

### Database Migrations

| File | Description |
|------|-------------|
| `drizzle/0002_add_entries.sql` | Journal entries table |
| `drizzle/0003_add_settings.sql` | User settings table |
| `drizzle/0004_add_sync_queue.sql` | Sync queue table |

### Tests

| File | Coverage |
|------|----------|
| `tests/lib/encryption.test.ts` | Encryption functions |
| `tests/lib/crypto.test.ts` | Web Crypto wrappers |
| `tests/server/entries.test.ts` | Entry server functions |
| `tests/server/settings.test.ts` | Settings server functions |
| `tests/hooks/useEntries.test.tsx` | React Query hooks |
| `tests/e2e/encryption-flow.test.ts` | Full flow |
| `tests/security/zero-knowledge.test.ts` | Security verification |

---

## Security Considerations

### Zero-Knowledge Architecture

| Component | What Server Sees | What Client Handles |
|-----------|------------------|---------------------|
| Database | Encrypted blobs only | Decryption keys |
| Server Functions | Encrypted payloads | Plaintext content |
| Network Traffic | Ciphertext | Plaintext (in memory) |
| Browser Storage | Nothing | Keys in memory only |

### Threat Model

| Threat | Mitigation |
|--------|------------|
| Database breach | Client-side encryption ensures ciphertext only |
| Server compromise | Zero-knowledge: server has no decryption keys |
| Memory dump | Keys cleared on lock/logout |
| XSS attack | No plaintext in DOM; encrypted until user unlocks |
| Man-in-the-middle | HTTPS only; encrypted payloads in transit |
| Brute force | PBKDF2 with high iteration count |
| Key reuse | Unique nonce for every encryption |

### Security Checklist

- [ ] No plaintext journal content in database
- [ ] No plaintext in server function responses
- [ ] Server never has access to decryption keys
- [ ] Encryption keys never leave client memory
- [ ] Automatic lock after inactivity
- [ ] HTTPS enforcement in production
- [ ] Input validation on all server functions
- [ ] Zod schema validation for type safety

---

## Dependencies Diagram

```
Phase 1 Architecture:
┌─────────────────────────────────────────────────────┐
│                    Client (Browser)                  │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │  Encryption     │  │   TanStack Query         │  │
│  │  (Web Crypto)   │  │   - Cache                │  │
│  │  - deriveKey    │  │   - Background fetch     │  │
│  │  - encrypt      │  │   - Optimistic updates   │  │
│  │  - decrypt      │  │   - Loading states       │  │
│  └────────┬────────┘  └───────────┬──────────────┘  │
│           │                       │                  │
│           │  Plaintext            │  Encrypted data  │
│           │  (memory only)        │  (server state)  │
│           ▼                       ▼                  │
│  ┌──────────────────────────────────────────────┐   │
│  │          React Components                     │   │
│  │  - Master password UI                         │   │
│  │  - Lock screen                                │   │
│  └──────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────┘
                        │
                        │ Type-safe RPC
                        │ (encrypted payloads)
                        ▼
┌─────────────────────────────────────────────────────┐
│                    Server (Node.js)                  │
│  ┌──────────────────────────────────────────────┐   │
│  │     TanStack Start Server Functions          │   │
│  │  - createServerFn                            │   │
│  │  - Zod validation                            │   │
│  │  - Better Auth session                       │   │
│  └─────────────────────┬────────────────────────┘   │
│                        │                            │
│                        ▼                            │
│  ┌──────────────────────────────────────────────┐   │
│  │              Database (SQLite)                │   │
│  │  - journal_entry (encrypted)                  │   │
│  │  - user_settings                              │   │
│  │  - sync_queue                                 │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## Next Phase Dependencies

**Phase 2: Editor & UI** depends on:
- ✅ Encryption context (this phase)
- ✅ useEntries hook with decryption (this phase)
- ✅ useCreateEntry mutation (this phase)
- ✅ Master password flow (this phase)
- ✅ TanStack Query client (this phase)

---

## Notes

### Why Server Functions + TanStack Query?

1. **Type Safety**: End-to-end TypeScript without manual API contracts
2. **Developer Experience**: Call functions directly, no fetch/axios boilerplate
3. **Zero-Knowledge**: Server never sees plaintext (encrypted payloads)
4. **Performance**: TanStack Query caching reduces server calls
5. **Future-Proof**: Easy transition to offline-first (Phase 3) with Query persistence

### Browser Compatibility

- **Required**: Modern browsers with Web Crypto API support
- **Chrome**: 37+
- **Firefox**: 34+
- **Safari**: 7+
- **Edge**: 12+

### Performance Considerations

- Web Worker prevents UI blocking during encryption
- TanStack Query caches decrypted data (with proper invalidation)
- Chunk large texts for progressive encryption
- Use requestIdleCallback for non-critical encryption tasks

### Known Limitations

- Master password cannot be recovered (by design)
- Encryption adds ~33% overhead to storage size (base64 encoding)
- Large entries (>10MB) may have performance implications
- Mobile devices may have slower key derivation times

---

## Resources

- [TanStack Start Server Functions](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)
- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Optimistic Updates Guide](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates)
- [Web Crypto API Spec](https://www.w3.org/TR/WebCryptoAPI/)
- [Better Auth Server-Side Usage](https://www.better-auth.com/docs/integrations/next)
- [Comlink Documentation](https://github.com/GoogleChromeLabs/comlink)
