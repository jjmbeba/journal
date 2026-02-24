# Phase 4: Features & Analytics

## Overview

Adds value-adding features like mood tracking, writing statistics, and optional on-device AI analysis. This phase focuses on insights derived from local data without compromising privacy. All analytics are computed client-side.

**Status**: Not Started  
**Estimated Duration**: 2 weeks  
**Priority**: Medium  
**Blocked by**: Phase 2 (Editor & UI)

---

## Key Features

### Mood Tracking

- 5-point mood scale (1-5) with emoji representations
- Mood selector in editor
- Mood stored with each entry
- Mood trends over time
- Mood correlation with writing patterns

### Writing Statistics

- **Basic Stats**: Total words, entries, days active
- **Streak Tracking**: Current and longest streaks
- **Writing Patterns**: Time of day, day of week
- **Goals**: Daily word count goals with progress
- **Trends**: Visual charts and insights

### Search Functionality

- Full-text search across entry content
- Search entry titles
- Tag-based filtering
- Fuzzy matching for typos
- Real-time results

### Entry Tagging

- Create custom tags
- Assign multiple tags to entries
- Filter entries by tag
- Tag cloud/popular tags

### Export Functionality

- **Encrypted JSON**: Full backup with encryption
- **Markdown**: Clean text export
- **Plain Text**: Simple text format
- **PDF**: Formatted document (optional)
- Select entries or full journal

### Optional: On-Device AI

**Approach**: Statistical analysis (Phase 1) + Optional ML (Phase 2)

- **Always On**: Word counts, patterns, trends (no AI)
- **Optional**: Sentiment analysis using Transformers.js
- **Privacy**: 100% client-side, no data leaves device

---

## Dependencies

### New Dependencies

```bash
# Core dependencies
bun add date-fns
bun add fuse.js
bun add file-saver
bun add recharts

# Optional: On-device ML
bun add @xenova/transformers
```

| Package | Purpose | Size | Required |
|---------|---------|------|----------|
| `date-fns` | Date calculations | ~20KB (tree-shakeable) | Yes |
| `fuse.js` | Fuzzy search | ~15KB | Yes |
| `file-saver` | File downloads | ~5KB | Yes |
| `recharts` | Data visualization | ~70KB | Yes |
| `@xenova/transformers` | On-device ML | ~3MB | No (optional) |

**Required Bundle Impact**: ~110KB  
**With Optional ML**: ~3.1MB

### Existing Dependencies

- `date-fns` - May already be installed in Phase 2
- `lucide-react` - Icons
- `tailwindcss` - Styling

---

## Technical Architecture

### Mood System

```typescript
// Mood definitions
const MOODS = {
  1: { 
    value: 1, 
    label: 'Rough',
    emoji: '😞',
    color: '#EF4444' // Red
  },
  2: { 
    value: 2, 
    label: 'Difficult',
    emoji: '😕',
    color: '#F59E0B' // Orange
  },
  3: { 
    value: 3, 
    label: 'Okay',
    emoji: '😐',
    color: '#6B7280' // Gray
  },
  4: { 
    value: 4, 
    label: 'Good',
    emoji: '🙂',
    color: '#10B981' // Green
  },
  5: { 
    value: 5, 
    label: 'Great',
    emoji: '😊',
    color: '#3B82F6' // Blue
  }
} as const;

// Mood stored with entry
interface JournalEntry {
  // ... other fields
  mood: 1 | 2 | 3 | 4 | 5 | null;
}
```

### Statistics Engine

```typescript
interface WritingStats {
  // Basic counts
  totalWords: number;
  totalEntries: number;
  totalDaysActive: number;
  
  // Streaks
  currentStreak: number; // days
  longestStreak: number;
  
  // Averages
  averageWordsPerDay: number;
  averageWordsPerEntry: number;
  averageEntriesPerDay: number;
  
  // Patterns
  writingTimeDistribution: number[]; // 24 hours, count per hour
  dayOfWeekDistribution: number[]; // 7 days, count per day
  
  // Mood (if tracked)
  averageMood: number | null;
  moodTrends: {
    date: string;
    mood: number;
    wordCount: number;
  }[];
  
  // Goals
  dailyGoal: number;
  goalProgress: {
    date: string;
    wordsWritten: number;
    goalMet: boolean;
  }[];
}

// Calculation functions
function calculateStats(entries: JournalEntry[]): WritingStats;
function calculateStreak(entries: JournalEntry[]): { current: number; longest: number };
function calculatePatterns(entries: JournalEntry[]): PatternData;
```

### Search Implementation

```typescript
// Search indexing (client-side)
interface SearchIndex {
  entries: FuseIndex<SearchableEntry>;
  lastUpdated: Date;
}

interface SearchableEntry {
  id: string;
  title: string;
  content: string;
  tags: string[];
  date: string;
}

// Search results
interface SearchResult {
  entry: JournalEntry;
  matches: {
    field: 'title' | 'content' | 'tags';
    snippet: string;
    score: number;
    indices: [number, number][];
  }[];
  score: number;
}

// Search function
function searchEntries(
  query: string,
  options?: SearchOptions
): SearchResult[];
```

### Export System

```typescript
type ExportFormat = 'json' | 'markdown' | 'txt' | 'pdf';

interface ExportOptions {
  format: ExportFormat;
  entries?: string[]; // Specific entry IDs, or all if undefined
  dateRange?: { start: Date; end: Date };
  includeTags?: boolean;
  includeMood?: boolean;
  encrypt?: boolean; // For JSON export
}

// Export functions
async function exportEntries(options: ExportOptions): Promise<Blob>;

// Format generators
function generateJSON(entries: JournalEntry[], encrypt: boolean): string;
function generateMarkdown(entries: JournalEntry[]): string;
function generatePlainText(entries: JournalEntry[]): string;
function generatePDF(entries: JournalEntry[]): Promise<Blob>; // Optional
```

### Tag System

```typescript
interface Tag {
  id: string;
  name: string;
  color?: string;
  createdAt: Date;
  entryCount: number;
}

// Tag operations
function createTag(name: string, color?: string): Promise<Tag>;
function getAllTags(): Promise<Tag[]>;
function updateTag(id: string, updates: Partial<Tag>): Promise<Tag>;
function deleteTag(id: string): Promise<void>;
function getEntriesByTag(tagId: string): Promise<JournalEntry[]>;
```

### On-Device AI (Optional)

```typescript
// Sentiment analysis
interface SentimentResult {
  label: 'positive' | 'negative' | 'neutral';
  score: number; // 0-1 confidence
}

async function analyzeSentiment(text: string): Promise<SentimentResult>;

// Usage
async function analyzeEntry(entry: JournalEntry): Promise<void> {
  const sentiment = await analyzeSentiment(decryptedContent);
  entry.sentiment = sentiment;
  await saveEntry(entry);
}
```

**Model**: `Xenova/distilbert-base-uncased-finetuned-sst-2-english`  
**Size**: ~65MB  
**Device**: Runs in Web Worker  
**Privacy**: 100% local, no network calls

---

## Success Metrics

### Mood Tracking

| Metric | Target | Method |
|--------|--------|--------|
| Mood selection rate | >50% of entries | Analytics |
| Mood accuracy | User-reported alignment | Survey |
| UI interaction time | <2 seconds | Timing test |

### Statistics

| Metric | Target | Test |
|--------|--------|------|
| Calculation accuracy | 100% | Unit tests |
| Streak calculation | Correct edge cases | Test suite |
| Pattern detection | Accurate trends | Validation |
| Calculation speed | <100ms for 1000 entries | Benchmark |

### Search

| Metric | Target | Tool |
|--------|--------|------|
| Search response time | <100ms | Performance test |
| Result relevance | Top 3 relevant | Manual review |
| Fuzzy matching | >80% typo tolerance | Test cases |
| Index build time | <1s for 1000 entries | Benchmark |

### Export

| Metric | Target | Test |
|--------|--------|------|
| Export speed | <3s for 100 entries | Performance test |
| Data integrity | 100% accuracy | Comparison test |
| File size | Reasonable compression | File analysis |

### AI (Optional)

| Metric | Target | Method |
|--------|--------|--------|
| Sentiment accuracy | >75% vs manual labeling | Validation set |
| Inference time | <500ms per entry | Benchmark |
| Model load time | <5s first time | Performance test |
| Memory usage | <200MB | Memory profiling |

---

## Test Requirements

### Mood Tests

```typescript
// tests/features/mood.test.ts
describe('Mood Tracking', () => {
  it('associates mood with entry', async () => {
    const entry = await createEntry({
      content: 'Test',
      mood: 4
    });
    
    expect(entry.mood).toBe(4);
    
    const retrieved = await getEntry(entry.id);
    expect(retrieved.mood).toBe(4);
  });
  
  it('displays mood in entry list', () => {
    const entries = [
      { id: '1', mood: 5 },
      { id: '2', mood: 3 }
    ];
    
    const { getByTestId } = render(<EntryList entries={entries} />);
    
    expect(getByTestId('entry-1-mood')).toHaveTextContent('😊');
    expect(getByTestId('entry-2-mood')).toHaveTextContent('😐');
  });
  
  it('calculates average mood over time', () => {
    const entries = [
      { mood: 4 },
      { mood: 5 },
      { mood: 3 },
      { mood: 4 }
    ];
    
    const avg = calculateAverageMood(entries);
    expect(avg).toBe(4);
  });
  
  it('shows mood trends in chart', () => {
    const moodData = [
      { date: '2024-01-01', mood: 3 },
      { date: '2024-01-02', mood: 4 },
      { date: '2024-01-03', mood: 5 }
    ];
    
    const { container } = render(<MoodChart data={moodData} />);
    expect(container.querySelector('.recharts-line')).toBeInTheDocument();
  });
  
  it('handles entries without mood', () => {
    const entries = [
      { mood: 4 },
      { mood: null },
      { mood: 5 }
    ];
    
    const avg = calculateAverageMood(entries);
    expect(avg).toBe(4.5); // Ignores null
  });
});
```

### Statistics Tests

```typescript
// tests/features/stats.test.ts
describe('Writing Statistics', () => {
  describe('Streaks', () => {
    it('calculates current streak correctly', () => {
      const entries = createEntriesForDays([
        '2024-01-05',
        '2024-01-04',
        '2024-01-03',
        '2024-01-01' // Gap on Jan 2
      ]);
      
      const stats = calculateStats(entries);
      expect(stats.currentStreak).toBe(3); // Jan 3-5
    });
    
    it('calculates longest streak correctly', () => {
      const entries = createEntriesForDays([
        '2024-01-01',
        '2024-01-02',
        '2024-01-03',
        '2024-01-05',
        '2024-01-06'
      ]);
      
      const stats = calculateStats(entries);
      expect(stats.longestStreak).toBe(3); // Jan 1-3
    });
    
    it('handles streak breaks', () => {
      const entries = createEntriesForDays([
        '2024-01-01',
        '2024-01-02',
        '2024-01-05',
        '2024-01-06'
      ]);
      
      const stats = calculateStats(entries);
      expect(stats.currentStreak).toBe(2); // Jan 5-6
      expect(stats.longestStreak).toBe(2); // Two streaks of 2
    });
    
    it('handles empty journal', () => {
      const stats = calculateStats([]);
      expect(stats.currentStreak).toBe(0);
      expect(stats.longestStreak).toBe(0);
    });
  });

  describe('Patterns', () => {
    it('calculates writing time distribution', () => {
      const entries = [
        { createdAt: new Date('2024-01-01T09:00:00') },
        { createdAt: new Date('2024-01-02T09:30:00') },
        { createdAt: new Date('2024-01-03T14:00:00') }
      ];
      
      const patterns = calculatePatterns(entries);
      expect(patterns.writingTimeDistribution[9]).toBe(2); // 9am
      expect(patterns.writingTimeDistribution[14]).toBe(1); // 2pm
    });
    
    it('calculates day of week distribution', () => {
      const entries = [
        { createdAt: new Date('2024-01-01') }, // Monday
        { createdAt: new Date('2024-01-08') }, // Monday
        { createdAt: new Date('2024-01-02') }  // Tuesday
      ];
      
      const patterns = calculatePatterns(entries);
      expect(patterns.dayOfWeekDistribution[1]).toBe(2); // Monday
      expect(patterns.dayOfWeekDistribution[2]).toBe(1); // Tuesday
    });
  });

  describe('Goals', () => {
    it('tracks daily goal progress', () => {
      const entries = [
        { createdAt: new Date('2024-01-01'), wordCount: 300 },
        { createdAt: new Date('2024-01-02'), wordCount: 600 }
      ];
      
      const goal = 500;
      const progress = calculateGoalProgress(entries, goal);
      
      expect(progress[0].goalMet).toBe(false); // 300 < 500
      expect(progress[1].goalMet).toBe(true);  // 600 >= 500
    });
    
    it('aggregates multiple entries per day', () => {
      const entries = [
        { createdAt: new Date('2024-01-01T09:00'), wordCount: 300 },
        { createdAt: new Date('2024-01-01T14:00'), wordCount: 300 }
      ];
      
      const goal = 500;
      const progress = calculateGoalProgress(entries, goal);
      
      expect(progress[0].wordsWritten).toBe(600);
      expect(progress[0].goalMet).toBe(true);
    });
  });
});
```

### Search Tests

```typescript
// tests/features/search.test.ts
describe('Search', () => {
  beforeEach(() => {
    // Build search index
    const entries = [
      { id: '1', title: 'Morning Thoughts', content: 'Today was a good day' },
      { id: '2', title: 'Work Update', content: 'Finished the project' },
      { id: '3', title: 'Evening Reflection', content: 'Today was challenging' }
    ];
    buildSearchIndex(entries);
  });

  it('searches entry content', () => {
    const results = searchEntries('good');
    expect(results).toHaveLength(1);
    expect(results[0].entry.id).toBe('1');
  });
  
  it('searches entry titles', () => {
    const results = searchEntries('morning');
    expect(results).toHaveLength(1);
    expect(results[0].entry.id).toBe('1');
  });
  
  it('supports fuzzy matching', () => {
    const results = searchEntries('thoghts'); // typo
    expect(results).toHaveLength(1);
    expect(results[0].entry.id).toBe('1');
  });
  
  it('ranks results by relevance', () => {
    const results = searchEntries('today');
    expect(results).toHaveLength(2);
    // Both entries 1 and 3 contain 'today'
    expect(results[0].score).toBeGreaterThan(results[1].score);
  });
  
  it('returns highlighted snippets', () => {
    const results = searchEntries('good');
    expect(results[0].matches[0].snippet).toContain('<mark>good</mark>');
  });
  
  it('handles empty search', () => {
    const results = searchEntries('');
    expect(results).toHaveLength(0);
  });
  
  it('handles no matches', () => {
    const results = searchEntries('xyz123');
    expect(results).toHaveLength(0);
  });
  
  it('searches in real-time', async () => {
    const { getByRole } = render(<SearchInterface />);
    const input = getByRole('searchbox');
    
    fireEvent.change(input, { target: { value: 'good' } });
    
    await waitFor(() => {
      expect(screen.getByText('Morning Thoughts')).toBeInTheDocument();
    });
  });
  
  it('respects encrypted content boundaries', () => {
    // Ensure search only works on decrypted content
    // Verify no plaintext leakage
  });
});
```

### Tag Tests

```typescript
// tests/features/tags.test.ts
describe('Tag System', () => {
  it('creates new tag', async () => {
    const tag = await createTag('Personal', '#3B82F6');
    expect(tag.name).toBe('Personal');
    expect(tag.color).toBe('#3B82F6');
  });
  
  it('prevents duplicate tag names', async () => {
    await createTag('Work');
    await expect(createTag('Work')).rejects.toThrow('Tag already exists');
  });
  
  it('assigns tags to entry', async () => {
    const tag = await createTag('Ideas');
    const entry = await createEntry({
      content: 'New idea',
      tags: [tag.id]
    });
    
    expect(entry.tags).toContain(tag.id);
  });
  
  it('filters entries by tag', async () => {
    const workTag = await createTag('Work');
    const personalTag = await createTag('Personal');
    
    await createEntry({ content: 'Work entry', tags: [workTag.id] });
    await createEntry({ content: 'Personal entry', tags: [personalTag.id] });
    await createEntry({ content: 'Both', tags: [workTag.id, personalTag.id] });
    
    const workEntries = await getEntriesByTag(workTag.id);
    expect(workEntries).toHaveLength(2);
  });
  
  it('updates tag color', async () => {
    const tag = await createTag('Test', '#000000');
    const updated = await updateTag(tag.id, { color: '#FFFFFF' });
    
    expect(updated.color).toBe('#FFFFFF');
  });
  
  it('updates entry count when entries added/removed', async () => {
    const tag = await createTag('Test');
    expect(tag.entryCount).toBe(0);
    
    await createEntry({ tags: [tag.id] });
    const updatedTag = await getTag(tag.id);
    expect(updatedTag.entryCount).toBe(1);
  });
});
```

### Export Tests

```typescript
// tests/features/export.test.ts
describe('Export', () => {
  const entries = [
    {
      id: '1',
      encryptedContent: '<p>Entry 1</p>',
      title: 'First Entry',
      tags: ['tag1'],
      mood: 4,
      createdAt: new Date('2024-01-01'),
      wordCount: 100
    },
    {
      id: '2',
      encryptedContent: '<p>Entry 2</p>',
      title: 'Second Entry',
      tags: ['tag2'],
      mood: 5,
      createdAt: new Date('2024-01-02'),
      wordCount: 200
    }
  ];

  it('exports to JSON', async () => {
    const blob = await exportEntries({
      format: 'json',
      entries: entries.map(e => e.id)
    });
    
    const content = await blob.text();
    const data = JSON.parse(content);
    
    expect(data.entries).toHaveLength(2);
    expect(data.entries[0].id).toBe('1');
  });
  
  it('exports to Markdown', async () => {
    const blob = await exportEntries({
      format: 'markdown',
      entries: entries.map(e => e.id)
    });
    
    const content = await blob.text();
    
    expect(content).toContain('# First Entry');
    expect(content).toContain('Entry 1');
    expect(content).toContain('---'); // Separator
  });
  
  it('exports to plain text', async () => {
    const blob = await exportEntries({
      format: 'txt',
      entries: entries.map(e => e.id)
    });
    
    const content = await blob.text();
    
    expect(content).toContain('First Entry');
    expect(content).not.toContain('#'); // No markdown
  });
  
  it('exports encrypted JSON', async () => {
    const blob = await exportEntries({
      format: 'json',
      entries: entries.map(e => e.id),
      encrypt: true
    });
    
    const content = await blob.text();
    const data = JSON.parse(content);
    
    expect(data.encrypted).toBe(true);
    expect(data.entries[0].encryptedContent).toBeDefined();
  });
  
  it('exports all entries when no IDs specified', async () => {
    const blob = await exportEntries({ format: 'json' });
    const content = await blob.text();
    const data = JSON.parse(content);
    
    expect(data.entries.length).toBeGreaterThan(0);
  });
  
  it('filters by date range', async () => {
    const blob = await exportEntries({
      format: 'json',
      dateRange: {
        start: new Date('2024-01-01'),
        end: new Date('2024-01-01')
      }
    });
    
    const content = await blob.text();
    const data = JSON.parse(content);
    
    expect(data.entries).toHaveLength(1);
  });
  
  it('generates correct file name', async () => {
    const blob = await exportEntries({ format: 'json' });
    expect(blob.type).toBe('application/json');
  });
});
```

### AI Tests (Optional)

```typescript
// tests/features/sentiment.test.ts (optional)
describe('Sentiment Analysis', () => {
  beforeAll(async () => {
    // Load model (slow)
    await loadSentimentModel();
  }, 30000);

  it('detects positive sentiment', async () => {
    const result = await analyzeSentiment('Today was amazing! I am so happy.');
    expect(result.label).toBe('positive');
    expect(result.score).toBeGreaterThan(0.7);
  });
  
  it('detects negative sentiment', async () => {
    const result = await analyzeSentiment('Everything went wrong today.');
    expect(result.label).toBe('negative');
    expect(result.score).toBeGreaterThan(0.7);
  });
  
  it('handles neutral text', async () => {
    const result = await analyzeSentiment('I went to the store today.');
    expect(result.label).toBe('neutral');
  });
  
  it('completes in reasonable time', async () => {
    const start = performance.now();
    await analyzeSentiment('Short text');
    const duration = performance.now() - start;
    expect(duration).toBeLessThan(500);
  });
  
  it('runs in web worker', async () => {
    // Verify it doesn't block main thread
  });
});
```

---

## Implementation Checklist

### Mood Tracking

- [ ] Mood data model
  - [ ] Mood type definitions
  - [ ] Mood constants (emojis, colors, labels)
  - [ ] Database schema update
  
- [ ] Mood selector component
  - [ ] 5-point scale UI
  - [ ] Touch-friendly design
  - [ ] Accessibility (ARIA labels)
  - [ ] Optional indicator in entry list
  
- [ ] Mood analytics
  - [ ] Average mood calculation
  - [ ] Mood trends over time
  - [ ] Mood correlation with word count
  - [ ] Visual charts

### Statistics

- [ ] Stats calculation engine
  - [ ] Basic counts (words, entries, days)
  - [ ] Streak calculation algorithm
  - [ ] Writing patterns (time, day of week)
  - [ ] Goal tracking
  
- [ ] Dashboard UI
  - [ ] Stats cards
  - [ ] Streak display
  - [ ] Writing goal progress
  - [ ] Pattern charts
  
- [ ] Visualization
  - [ ] Line charts for trends
  - [ ] Bar charts for patterns
  - [ ] Heatmaps for activity
  - [ ] Responsive charts

### Search

- [ ] Search indexing
  - [ ] Index builder
  - [ ] Incremental updates
  - [ ] Index persistence (optional)
  
- [ ] Search interface
  - [ ] Search input component
  - [ ] Real-time results
  - [ ] Highlighted snippets
  - [ ] Filter by date/tag (optional)
  
- [ ] Search algorithm
  - [ ] Fuse.js integration
  - [ ] Fuzzy matching
  - [ ] Relevance scoring
  - [ ] Performance optimization

### Tags

- [ ] Tag data model
  - [ ] Tag schema
  - [ ] Entry-tag relationship
  - [ ] Color support
  
- [ ] Tag management UI
  - [ ] Create tag dialog
  - [ ] Tag selector in editor
  - [ ] Tag list/filter view
  - [ ] Tag cloud (optional)
  
- [ ] Tag API
  - [ ] CRUD operations
  - [ ] Get entries by tag
  - [ ] Tag statistics

### Export

- [ ] Export formats
  - [ ] JSON (encrypted and plaintext)
  - [ ] Markdown
  - [ ] Plain text
  - [ ] PDF (optional - requires jspdf or similar)
  
- [ ] Export UI
  - [ ] Export dialog
  - [ ] Format selection
  - [ ] Entry selection (all or specific)
  - [ ] Date range filter
  - [ ] Progress indicator
  
- [ ] Export logic
  - [ ] Format generators
  - [ ] File generation
  - [ ] Download trigger
  - [ ] Error handling

### AI (Optional)

- [ ] Model setup
  - [ ] Transformers.js integration
  - [ ] Model download/cache
  - [ ] Web Worker implementation
  
- [ ] Sentiment analysis
  - [ ] Analyze entry content
  - [ ] Store sentiment score
  - [ ] Visualize sentiment trends
  
- [ ] Privacy safeguards
  - [ ] 100% local processing
  - [ ] No network calls
  - [ ] User opt-in
  - [ ] Model size warnings

### Testing

- [ ] Unit tests for all functions
- [ ] Component tests for UI
- [ ] Integration tests for flows
- [ ] Performance benchmarks
- [ ] Visual regression tests

---

## Deliverables

### Code Files

| File | Purpose |
|------|---------|
| `src/lib/stats.ts` | Statistics calculation engine |
| `src/lib/search.ts` | Search implementation |
| `src/lib/export.ts` | Export utilities |
| `src/lib/tags.ts` | Tag management |
| `src/lib/sentiment.ts` | AI sentiment analysis (optional) |
| `src/hooks/useStats.ts` | Statistics hook |
| `src/hooks/useSearch.ts` | Search hook |
| `src/hooks/useTags.ts` | Tags hook |
| `src/components/dashboard/` | Dashboard components |
│   ├── StatsCard.tsx
│   ├── StreakDisplay.tsx
│   ├── GoalProgress.tsx
│   ├── WritingPatterns.tsx
│   └── MoodChart.tsx
| `src/components/search/` | Search components |
│   ├── SearchInterface.tsx
│   ├── SearchResults.tsx
│   └── SearchHighlight.tsx
| `src/components/tags/` | Tag components |
│   ├── TagManager.tsx
│   ├── TagSelector.tsx
│   └── TagFilter.tsx
| `src/components/export/` | Export components |
│   └── ExportDialog.tsx
| `src/workers/sentiment.worker.ts` | AI worker (optional) |
| `src/routes/dashboard.tsx` | Dashboard page |
| `src/routes/search.tsx` | Search page |

### Test Files

| File | Coverage |
|------|----------|
| `tests/features/mood.test.ts` | Mood tracking |
| `tests/features/stats.test.ts` | Statistics |
| `tests/features/search.test.ts` | Search |
| `tests/features/tags.test.ts` | Tags |
| `tests/features/export.test.ts` | Export |
| `tests/features/sentiment.test.ts` | AI (optional) |
| `tests/components/dashboard/*.test.tsx` | Dashboard |

---

## File Structure

```
src/
├── lib/
│   ├── stats.ts
│   ├── search.ts
│   ├── export.ts
│   ├── tags.ts
│   └── sentiment.ts (optional)
├── hooks/
│   ├── useStats.ts
│   ├── useSearch.ts
│   └── useTags.ts
├── components/
│   ├── dashboard/
│   │   ├── StatsCard.tsx
│   │   ├── StreakDisplay.tsx
│   │   ├── GoalProgress.tsx
│   │   ├── WritingPatterns.tsx
│   │   └── MoodChart.tsx
│   ├── search/
│   │   ├── SearchInterface.tsx
│   │   ├── SearchResults.tsx
│   │   └── SearchHighlight.tsx
│   ├── tags/
│   │   ├── TagManager.tsx
│   │   ├── TagSelector.tsx
│   │   └── TagFilter.tsx
│   └── export/
│       └── ExportDialog.tsx
├── workers/
│   └── sentiment.worker.ts (optional)
└── routes/
    ├── dashboard.tsx
    └── search.tsx
```

---

## Dependencies on Previous Phases

| Dependency | From | Usage |
|------------|------|-------|
| Entry data | Phase 1-3 | Statistics, search, export |
| Encryption | Phase 1 | Decrypt for search/analysis |
| IndexedDB | Phase 3 | Local search index |
| Editor | Phase 2 | Mood selector integration |

---

## Privacy & Security Considerations

### Client-Side Only

- All analytics computed locally
- No analytics services (Google Analytics, etc.)
- No data sent to external services
- Search index stored locally

### Encryption Boundaries

- Search only works on decrypted content (after unlock)
- Export respects encryption settings
- Never expose plaintext in logs/errors

### AI Privacy (Optional)

- 100% on-device processing
- No cloud API calls
- Model downloaded to device
- User must explicitly enable

---

## Performance Considerations

### Statistics

- Calculate incrementally (don't reprocess all entries)
- Cache calculated stats
- Invalidate cache on entry change

### Search

- Build index on app load
- Incremental index updates
- Debounce search input
- Limit results (pagination)

### Export

- Stream large exports
- Show progress for many entries
- Compress large files
- Memory-efficient processing

### AI (Optional)

- Load model on demand (not at startup)
- Cache model in IndexedDB
- Process in Web Worker
- Batch processing for multiple entries

---

## Resources

- [Fuse.js Documentation](https://fusejs.io/)
- [Recharts Examples](https://recharts.org/en-US/examples)
- [Transformers.js](https://huggingface.co/docs/transformers.js/)
- [FileSaver.js](https://github.com/eligrey/FileSaver.js/)
- [date-fns Documentation](https://date-fns.org/)
