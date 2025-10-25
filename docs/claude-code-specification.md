# Voice Pipeline App - Claude Code Development Specification

## Project Overview

**Purpose:** Voice-enabled mobile app for mortgage loan officers to query loan status from ClickUp via speech, with visual fallback interface.

**Target Users:** 6 loan officers initially, scalable to 500+

**Core Value Proposition:** Eliminate 1-2 hours/day of phone calls between LOs and processors by providing instant voice-activated loan status updates.

---

## Technical Stack

### Mobile Application
- **Framework:** React Native (v0.72+) with Expo (SDK 50+)
- **Language:** TypeScript (strict mode)
- **Navigation:** Expo Router (file-based routing)
- **State Management:** React Query for server state, Zustand for local state
- **UI Components:** React Native Paper + Custom components
- **Voice:** 
  - STT: Wispr Flow SDK (with fallback to Expo Speech)
  - TTS: ElevenLabs API (with fallback to Expo Speech)
- **Authentication:** Okta React Native SDK (OIDC flow)
- **Storage:** 
  - Secure: expo-secure-store (tokens, keys)
  - Cache: AsyncStorage (loan data)
- **Network:** Axios with retry logic

### Backend API
- **Runtime:** Node.js v20+
- **Framework:** Fastify v4+ with TypeScript
- **Authentication:** Okta JWT validation
- **Validation:** Zod schemas
- **Database Client:** Drizzle ORM
- **Logging:** Pino (JSON structured logs)
- **Environment:** dotenv for local, Railway env vars for production

### Database
- **Type:** PostgreSQL 15+
- **Hosting:** Supabase (free tier → Pro)
- **Migrations:** Drizzle Kit
- **Indexes:** Optimized for loan officer queries

### External Integrations
- **ClickUp API:** v2 REST API (100 req/min limit)
- **Wispr Flow:** STT service
- **ElevenLabs:** TTS service
- **Encompass:** (Future) ICE Mortgage Technology API

### Infrastructure
- **Backend Hosting:** Railway (US region)
- **Database:** Supabase (US region)
- **Caching:** Upstash Redis (serverless)
- **CI/CD:** GitHub Actions
- **Secrets Management:** Railway environment variables

---

## Project Structure

### Mobile App (`/mobile`)
```
mobile/
├── app/                           # Expo Router pages
│   ├── (auth)/
│   │   ├── login.tsx             # Okta login screen
│   │   └── _layout.tsx           # Auth layout wrapper
│   ├── (tabs)/
│   │   ├── _layout.tsx           # Bottom tab navigation
│   │   ├── index.tsx             # Voice interface (home)
│   │   ├── pipeline.tsx          # List view of loans
│   │   └── settings.tsx          # User preferences
│   ├── loan/
│   │   └── [id].tsx              # Loan detail screen
│   └── _layout.tsx               # Root layout
├── components/
│   ├── VoiceButton.tsx           # Tap-to-talk + wake word
│   ├── LoanCard.tsx              # Loan summary card
│   ├── ConditionsList.tsx        # Conditions checklist
│   ├── AudioWaveform.tsx         # Visual feedback
│   └── EmptyState.tsx            # Empty states
├── services/
│   ├── api.ts                    # Backend API client
│   ├── auth.ts                   # Okta authentication
│   ├── voice.ts                  # STT/TTS wrapper
│   ├── offline.ts                # Cache management
│   └── analytics.ts              # Event tracking
├── hooks/
│   ├── useLoans.ts               # Loan data queries
│   ├── useVoiceQuery.ts          # Voice command handling
│   └── useAuth.ts                # Auth state
├── utils/
│   ├── intentParser.ts           # Parse voice commands
│   ├── formatters.ts             # Format data for display/TTS
│   └── constants.ts              # App constants
├── types/
│   └── index.ts                  # TypeScript types
├── app.json                      # Expo config
├── package.json
└── tsconfig.json
```

### Backend API (`/backend`)
```
backend/
├── src/
│   ├── routes/
│   │   ├── auth.ts               # POST /auth/login, /auth/refresh
│   │   ├── loans.ts              # GET /loans, GET /loans/:id
│   │   ├── voice.ts              # POST /voice/query
│   │   ├── sync.ts               # POST /sync/clickup (webhook)
│   │   └── health.ts             # GET /health
│   ├── services/
│   │   ├── clickup.ts            # ClickUp API wrapper
│   │   ├── encompass.ts          # Encompass API (future)
│   │   ├── voice/
│   │   │   ├── stt.ts            # Wispr Flow STT
│   │   │   ├── tts.ts            # ElevenLabs TTS
│   │   │   └── intent.ts         # Intent parser
│   │   ├── cache.ts              # Redis caching
│   │   └── database.ts           # Drizzle queries
│   ├── middleware/
│   │   ├── auth.ts               # JWT validation
│   │   ├── rateLimit.ts          # Rate limiting
│   │   ├── errorHandler.ts       # Global error handler
│   │   └── audit.ts              # Audit logging
│   ├── db/
│   │   ├── schema.ts             # Drizzle schema
│   │   └── migrations/           # SQL migrations
│   ├── types/
│   │   └── index.ts              # TypeScript types
│   ├── utils/
│   │   ├── logger.ts             # Pino logger
│   │   ├── errors.ts             # Custom errors
│   │   └── validation.ts         # Zod schemas
│   └── server.ts                 # Fastify server setup
├── tests/
│   ├── integration/
│   └── unit/
├── package.json
├── tsconfig.json
├── drizzle.config.ts
└── .env.example
```

---

## Database Schema

```typescript
// src/db/schema.ts (Drizzle ORM)

import { pgTable, uuid, text, timestamp, integer, jsonb, date, index } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  oktaUserId: text('okta_user_id').unique().notNull(),
  email: text('email').unique().notNull(),
  fullName: text('full_name'),
  role: text('role', { enum: ['loan_officer', 'processor', 'admin'] }).notNull(),
  createdAt: timestamp('created_at').defaultNow(),
}, (table) => ({
  oktaUserIdIdx: index('users_okta_user_id_idx').on(table.oktaUserId),
}));

export const loans = pgTable('loans', {
  id: uuid('id').primaryKey().defaultRandom(),
  clickupTaskId: text('clickup_task_id').unique().notNull(),
  borrowerName: text('borrower_name').notNull(),
  propertyAddress: text('property_address'),
  loanStage: text('loan_stage').notNull(),
  stageUpdatedAt: timestamp('stage_updated_at'),
  closingDate: date('closing_date'),
  conditionsCount: integer('conditions_count').default(0),
  loanOfficerId: uuid('loan_officer_id').references(() => users.id),
  processorId: uuid('processor_id').references(() => users.id),
  lastSyncedAt: timestamp('last_synced_at').defaultNow(),
  metadata: jsonb('metadata'), // Custom fields from ClickUp
}, (table) => ({
  borrowerNameIdx: index('loans_borrower_name_idx').on(table.borrowerName),
  loanOfficerIdx: index('loans_loan_officer_idx').on(table.loanOfficerId),
  loanStageIdx: index('loans_loan_stage_idx').on(table.loanStage),
  closingDateIdx: index('loans_closing_date_idx').on(table.closingDate),
}));

export const conditions = pgTable('conditions', {
  id: uuid('id').primaryKey().defaultRandom(),
  loanId: uuid('loan_id').references(() => loans.id, { onDelete: 'cascade' }).notNull(),
  description: text('description').notNull(),
  status: text('status', { enum: ['pending', 'submitted', 'cleared'] }).notNull(),
  dueDate: date('due_date'),
  clearedAt: timestamp('cleared_at'),
}, (table) => ({
  loanIdIdx: index('conditions_loan_id_idx').on(table.loanId),
}));

export const loanActivities = pgTable('loan_activities', {
  id: uuid('id').primaryKey().defaultRandom(),
  loanId: uuid('loan_id').references(() => loans.id, { onDelete: 'cascade' }).notNull(),
  activityType: text('activity_type').notNull(), // 'comment', 'stage_change', 'doc_upload'
  description: text('description').notNull(),
  createdBy: uuid('created_by').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow(),
}, (table) => ({
  loanIdCreatedAtIdx: index('loan_activities_loan_id_created_at_idx').on(table.loanId, table.createdAt),
}));

export const auditLogs = pgTable('audit_logs', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').references(() => users.id),
  action: text('action').notNull(), // 'view_loan', 'query_voice', 'update_stage'
  resourceId: text('resource_id'),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  createdAt: timestamp('created_at').defaultNow(),
}, (table) => ({
  userIdCreatedAtIdx: index('audit_logs_user_id_created_at_idx').on(table.userId, table.createdAt),
}));
```

---

## API Endpoints

### Authentication
```typescript
POST /api/auth/login
Body: { code: string } // Okta auth code
Response: { accessToken, refreshToken, user }

POST /api/auth/refresh
Body: { refreshToken: string }
Response: { accessToken }
```

### Loans
```typescript
GET /api/loans
Query: { loanOfficerId?: string, stage?: string }
Response: { loans: Loan[] }

GET /api/loans/:id
Response: { loan: Loan, conditions: Condition[], activities: Activity[] }

PUT /api/loans/:id/stage
Body: { stage: string }
Response: { loan: Loan }
```

### Voice
```typescript
POST /api/voice/query
Body: { audioBase64: string, userId: string }
Response: { 
  transcription: string,
  intent: { action: string, params: any },
  responseText: string,
  audioUrl?: string // TTS audio
}

POST /api/voice/tts
Body: { text: string }
Response: { audioUrl: string }
```

### Sync
```typescript
POST /api/sync/clickup
Body: { event: 'taskUpdated' | 'taskCreated', task: ClickUpTask }
Response: { success: boolean }

POST /api/sync/manual
Response: { synced: number, errors: number }
```

---

## Voice Intent Parser

### Supported Commands (Phase 1)

```typescript
// utils/intentParser.ts

interface Intent {
  action: 'status' | 'conditions' | 'closing_date' | 'pipeline' | 'updates' | 'unknown';
  params: {
    borrowerName?: string;
    loanId?: string;
    timeframe?: string;
  };
}

// Examples:
// "What's my pipeline?" → { action: 'pipeline', params: {} }
// "Status of Johnson loan" → { action: 'status', params: { borrowerName: 'Johnson' } }
// "What conditions does Smith need?" → { action: 'conditions', params: { borrowerName: 'Smith' } }
// "When does Garcia close?" → { action: 'closing_date', params: { borrowerName: 'Garcia' } }
// "What's new today?" → { action: 'updates', params: { timeframe: 'today' } }

const INTENT_PATTERNS = [
  {
    pattern: /(?:what's|show me|list) (?:my )?pipeline/i,
    action: 'pipeline'
  },
  {
    pattern: /(?:status|update) (?:of |on |for )?(?:the )?(\w+)(?: loan)?/i,
    action: 'status',
    extractBorrower: 1
  },
  {
    pattern: /(?:what conditions|conditions needed|what does|does) (?:the )?(\w+)(?: need| have)/i,
    action: 'conditions',
    extractBorrower: 1
  },
  {
    pattern: /when (?:does|is) (?:the )?(\w+)(?: loan)?(?: close| closing)/i,
    action: 'closing_date',
    extractBorrower: 1
  },
  {
    pattern: /(?:what's new|any updates|recent activity)(?: today)?/i,
    action: 'updates',
    timeframe: 'today'
  }
];
```

### Response Templates

```typescript
// utils/formatters.ts

const TTS_TEMPLATES = {
  pipeline: (data: { total: number, byStage: Record<string, number> }) => 
    `You have ${data.total} active loans. ${data.byStage.processing || 0} in processing, ${data.byStage.underwriting || 0} in underwriting, and ${data.byStage.clear_to_close || 0} clear to close.`,
  
  status: (loan: Loan) => 
    `${loan.borrowerName} loan at ${loan.propertyAddress} is in ${loan.loanStage}. Currently at day ${getDaysInStage(loan)}, with ${loan.conditionsCount} outstanding conditions. Closing date is ${formatDate(loan.closingDate)}.`,
  
  conditions: (loan: Loan, conditions: Condition[]) => {
    const pending = conditions.filter(c => c.status === 'pending');
    if (pending.length === 0) return `${loan.borrowerName} loan has all conditions cleared.`;
    return `${loan.borrowerName} needs ${pending.length} conditions: ${pending.map(c => c.description).join(', ')}.`;
  },
  
  closing_date: (loan: Loan) => 
    `${loan.borrowerName} loan closes on ${formatDate(loan.closingDate)}, in ${getDaysUntil(loan.closingDate)} days.`,
  
  updates: (activities: Activity[]) => {
    if (activities.length === 0) return "No updates today.";
    return `You have ${activities.length} updates today: ${activities.map(a => a.description).join('. ')}`;
  }
};
```

---

## ClickUp Integration

### Custom Field Mapping

```typescript
// services/clickup.ts

const CUSTOM_FIELD_MAP = {
  'Closing Date': 'closing_date',
  'Loan Officer': 'loan_officer_name',
  'Processor': 'processor_name',
  'Property Address': 'property_address',
  'Conditions Count': 'conditions_count',
  'Client Phone': 'client_phone',
  'Client Email': 'client_email',
  'Loan Type': 'loan_type',
  'Loan Amount': 'loan_amount'
};

function mapClickUpTaskToLoan(task: ClickUpTask): Partial<Loan> {
  const customFields = task.custom_fields.reduce((acc, field) => {
    const mappedKey = CUSTOM_FIELD_MAP[field.name];
    if (mappedKey) {
      acc[mappedKey] = field.value;
    }
    return acc;
  }, {} as Record<string, any>);

  return {
    clickupTaskId: task.id,
    borrowerName: extractBorrowerName(task.name),
    propertyAddress: customFields.property_address || extractAddress(task.name),
    loanStage: task.status.status,
    stageUpdatedAt: new Date(parseInt(task.date_updated)),
    closingDate: customFields.closing_date ? new Date(customFields.closing_date) : null,
    conditionsCount: customFields.conditions_count || 0,
    metadata: customFields
  };
}

// Sync strategy: Poll every 15 minutes + webhook for instant updates
async function syncLoansFromClickUp() {
  const tasks = await clickup.getTasks({
    list_id: process.env.CLICKUP_LIST_ID,
    statuses: ['Processing', 'Underwriting', 'Clear to Close', 'Docs Out']
  });
  
  for (const task of tasks) {
    await upsertLoan(mapClickUpTaskToLoan(task));
    await syncConditions(task);
  }
}
```

### Webhook Handler

```typescript
// routes/sync.ts

app.post('/sync/clickup', async (req, res) => {
  const { event, task_id, webhook_id } = req.body;
  
  // Verify webhook signature
  const signature = req.headers['x-signature'];
  if (!verifyClickUpSignature(signature, req.body)) {
    return res.status(401).send({ error: 'Invalid signature' });
  }
  
  switch (event) {
    case 'taskUpdated':
    case 'taskCreated':
      const task = await clickup.getTask(task_id);
      await upsertLoan(mapClickUpTaskToLoan(task));
      break;
    case 'taskDeleted':
      await deleteLoan(task_id);
      break;
  }
  
  return res.send({ success: true });
});
```

---

## Authentication Flow

### Okta OIDC Setup

```typescript
// services/auth.ts (Mobile)

import { OktaAuth } from '@okta/okta-react-native';

const oktaAuth = new OktaAuth({
  issuer: 'https://your-domain.okta.com/oauth2/default',
  clientId: process.env.OKTA_CLIENT_ID!,
  redirectUri: 'com.truenorth.pipeline://callback',
  scopes: ['openid', 'profile', 'email', 'offline_access'],
  requireHardwareBackedKeyStore: false // For development
});

export async function login() {
  try {
    const tokens = await oktaAuth.signInWithBrowser();
    await SecureStore.setItemAsync('access_token', tokens.accessToken);
    await SecureStore.setItemAsync('refresh_token', tokens.refreshToken!);
    return tokens;
  } catch (error) {
    console.error('Login failed:', error);
    throw error;
  }
}

export async function getAccessToken() {
  const token = await SecureStore.getItemAsync('access_token');
  if (!token) throw new Error('Not authenticated');
  
  // Check if token is expired
  const decoded = jwtDecode(token);
  if (decoded.exp! * 1000 < Date.now()) {
    // Refresh token
    return await refreshAccessToken();
  }
  
  return token;
}
```

### Backend JWT Validation

```typescript
// middleware/auth.ts (Backend)

import { FastifyRequest, FastifyReply } from 'fastify';
import { OktaJwtVerifier } from '@okta/jwt-verifier';

const oktaVerifier = new OktaJwtVerifier({
  issuer: process.env.OKTA_ISSUER!,
  clientId: process.env.OKTA_CLIENT_ID!
});

export async function authenticate(req: FastifyRequest, res: FastifyReply) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).send({ error: 'Missing or invalid authorization header' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const jwt = await oktaVerifier.verifyAccessToken(token, 'api://default');
    
    // Attach user to request
    req.user = {
      oktaUserId: jwt.claims.uid,
      email: jwt.claims.sub
    };
  } catch (error) {
    return res.status(401).send({ error: 'Invalid token' });
  }
}
```

---

## Offline Caching Strategy

```typescript
// services/offline.ts (Mobile)

import AsyncStorage from '@react-native-async-storage/async-storage';
import * as Crypto from 'expo-crypto';

const CACHE_KEY = 'loans_cache';
const CACHE_DURATION = 48 * 60 * 60 * 1000; // 48 hours
const ENCRYPTION_KEY = 'user-specific-key-from-keychain';

interface CachedData {
  loans: Loan[];
  timestamp: number;
}

export async function cacheLoans(loans: Loan[]) {
  const data: CachedData = {
    loans,
    timestamp: Date.now()
  };
  
  // Encrypt before storing (HIPAA/GLBA compliance)
  const encrypted = await Crypto.digestStringAsync(
    Crypto.CryptoDigestAlgorithm.SHA256,
    JSON.stringify(data) + ENCRYPTION_KEY
  );
  
  await AsyncStorage.setItem(CACHE_KEY, JSON.stringify({ data, hash: encrypted }));
}

export async function getCachedLoans(): Promise<Loan[] | null> {
  const cached = await AsyncStorage.getItem(CACHE_KEY);
  if (!cached) return null;
  
  const { data, hash }: { data: CachedData, hash: string } = JSON.parse(cached);
  
  // Verify integrity
  const expectedHash = await Crypto.digestStringAsync(
    Crypto.CryptoDigestAlgorithm.SHA256,
    JSON.stringify(data) + ENCRYPTION_KEY
  );
  
  if (hash !== expectedHash) {
    console.warn('Cache integrity check failed');
    await AsyncStorage.removeItem(CACHE_KEY);
    return null;
  }
  
  // Check if stale
  if (Date.now() - data.timestamp > CACHE_DURATION) {
    await AsyncStorage.removeItem(CACHE_KEY);
    return null;
  }
  
  return data.loans;
}

export async function syncWithBackend() {
  try {
    const response = await api.get('/loans');
    await cacheLoans(response.data.loans);
    return response.data.loans;
  } catch (error) {
    console.error('Sync failed, using cached data:', error);
    return await getCachedLoans();
  }
}
```

---

## Environment Variables

### Mobile App (`.env`)
```bash
OKTA_ISSUER=https://your-domain.okta.com/oauth2/default
OKTA_CLIENT_ID=your_client_id
API_BASE_URL=https://your-backend.railway.app
WISPR_API_KEY=your_wispr_key
ELEVENLABS_API_KEY=your_elevenlabs_key
```

### Backend (`.env`)
```bash
# Database
DATABASE_URL=postgresql://user:pass@db.xxx.supabase.co:5432/postgres

# Redis
REDIS_URL=redis://default:xxx@xxx.upstash.io:6379

# Okta
OKTA_ISSUER=https://your-domain.okta.com/oauth2/default
OKTA_CLIENT_ID=your_client_id

# ClickUp
CLICKUP_API_KEY=pk_xxx
CLICKUP_LIST_ID=123456789
CLICKUP_WEBHOOK_SECRET=your_webhook_secret

# Voice Services
WISPR_API_KEY=your_wispr_key
ELEVENLABS_API_KEY=your_elevenlabs_key

# App
NODE_ENV=production
PORT=3000
LOG_LEVEL=info
```

---

## Error Handling

### Global Error Handler (Backend)

```typescript
// middleware/errorHandler.ts

import { FastifyError, FastifyReply, FastifyRequest } from 'fastify';

export class AppError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public isOperational = true
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export async function errorHandler(
  error: FastifyError,
  req: FastifyRequest,
  res: FastifyReply
) {
  // Log error
  req.log.error({
    err: error,
    req: {
      method: req.method,
      url: req.url,
      headers: req.headers,
      body: req.body
    }
  });
  
  // Audit log for security-related errors
  if (error.statusCode === 401 || error.statusCode === 403) {
    await auditLog({
      action: 'auth_error',
      userId: req.user?.id,
      error: error.message,
      ipAddress: req.ip
    });
  }
  
  // Send response
  const statusCode = error.statusCode || 500;
  const message = error.isOperational 
    ? error.message 
    : 'Internal server error';
  
  return res.status(statusCode).send({
    error: message,
    statusCode
  });
}
```

### Mobile Error Handling

```typescript
// services/api.ts (Mobile)

axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token expired, try refresh
      try {
        await refreshAccessToken();
        return axios.request(error.config);
      } catch (refreshError) {
        // Refresh failed, logout
        await logout();
        throw new Error('Session expired. Please login again.');
      }
    }
    
    if (error.response?.status === 503) {
      // ClickUp down
      return {
        data: {
          error: 'ClickUp is currently unavailable. Please try again later.',
          useCache: true
        }
      };
    }
    
    throw error;
  }
);
```

---

## Testing Strategy

### Unit Tests (Backend)
```typescript
// tests/unit/intentParser.test.ts

import { parseIntent } from '../src/utils/intentParser';

describe('Intent Parser', () => {
  test('parses pipeline query', () => {
    const intent = parseIntent("What's my pipeline?");
    expect(intent.action).toBe('pipeline');
  });
  
  test('parses status query with borrower name', () => {
    const intent = parseIntent("Status of Johnson loan");
    expect(intent.action).toBe('status');
    expect(intent.params.borrowerName).toBe('Johnson');
  });
  
  test('handles unknown queries gracefully', () => {
    const intent = parseIntent("random nonsense");
    expect(intent.action).toBe('unknown');
  });
});
```

### Integration Tests (Backend)
```typescript
// tests/integration/loans.test.ts

import { build } from '../src/server';

describe('Loans API', () => {
  let app;
  
  beforeAll(async () => {
    app = await build();
  });
  
  afterAll(async () => {
    await app.close();
  });
  
  test('GET /api/loans requires authentication', async () => {
    const response = await app.inject({
      method: 'GET',
      url: '/api/loans'
    });
    
    expect(response.statusCode).toBe(401);
  });
  
  test('GET /api/loans returns user loans', async () => {
    const token = await getTestToken();
    
    const response = await app.inject({
      method: 'GET',
      url: '/api/loans',
      headers: {
        authorization: `Bearer ${token}`
      }
    });
    
    expect(response.statusCode).toBe(200);
    expect(response.json()).toHaveProperty('loans');
  });
});
```

---

## Deployment Instructions

### Backend Deployment (Railway)

1. **Connect GitHub repository:**
   ```bash
   railway login
   railway init
   railway link
   ```

2. **Add environment variables in Railway dashboard:**
   - DATABASE_URL (from Supabase)
   - All other env vars from `.env.example`

3. **Deploy:**
   ```bash
   railway up
   ```

4. **Run migrations:**
   ```bash
   railway run npm run db:migrate
   ```

### Mobile App Deployment (Expo)

1. **Build for iOS:**
   ```bash
   eas build --platform ios --profile production
   ```

2. **Build for Android:**
   ```bash
   eas build --platform android --profile production
   ```

3. **Submit to stores:**
   ```bash
   eas submit --platform ios
   eas submit --platform android
   ```

4. **Over-the-air updates:**
   ```bash
   eas update --branch production --message "Bug fixes"
   ```

---

## Development Workflow

### Initial Setup
```bash
# Clone repository
git clone <repo-url>
cd voice-pipeline-app

# Install dependencies
cd mobile && npm install
cd ../backend && npm install

# Set up environment
cp .env.example .env
# Fill in all required variables

# Start Supabase local (optional)
npx supabase start

# Run migrations
npm run db:migrate

# Start backend
npm run dev

# Start mobile app (in new terminal)
cd mobile
npx expo start
```

### Daily Development
```bash
# Backend: Auto-reload on changes
cd backend && npm run dev

# Mobile: Expo dev server
cd mobile && npx expo start

# Database changes
npm run db:generate  # Generate migration
npm run db:migrate   # Apply migration
```

---

## Phase 1 MVP Checklist

### Week 1: Foundation
- [ ] Initialize Expo project with TypeScript
- [ ] Set up Fastify backend with TypeScript
- [ ] Create Supabase database and apply schema
- [ ] Set up Okta authentication (dev tenant)
- [ ] Test ClickUp API connection
- [ ] Deploy backend to Railway staging

### Week 2: ClickUp Integration
- [ ] Build ClickUp sync service
- [ ] Implement custom field mapping
- [ ] Create loan upsert logic
- [ ] Set up 15-minute polling
- [ ] Add webhook endpoint
- [ ] Test data sync end-to-end

### Week 3: Voice Interface
- [ ] Integrate Wispr Flow STT
- [ ] Integrate ElevenLabs TTS
- [ ] Build intent parser
- [ ] Create voice button UI
- [ ] Implement 5 core voice queries
- [ ] Test voice accuracy

### Week 4: UI & Polish
- [ ] Build loan list screen
- [ ] Build loan detail screen
- [ ] Implement offline caching
- [ ] Add loading states
- [ ] Error handling & fallbacks
- [ ] Internal testing with 2 users

---

## Critical Implementation Notes for Claude Code

### Priority Order
1. **Start with backend API** - Mobile depends on it
2. **Then database schema** - Backend depends on it
3. **ClickUp integration next** - Data must flow before voice works
4. **Voice last** - Builds on top of everything else

### Watch Out For
- **ClickUp rate limits:** Cache aggressively, use webhooks
- **Voice accuracy:** Test with actual LO voices, not just your own
- **Offline conflicts:** Last-write-wins is acceptable for MVP
- **Token refresh:** Implement before app goes to production
- **Audit logging:** Log EVERYTHING for compliance

### Success Criteria
- ✅ Voice queries return results in <3 seconds
- ✅ 90%+ voice transcription accuracy on borrower names
- ✅ App works offline for 48 hours
- ✅ Zero authentication errors after token refresh
- ✅ All API endpoints return < 500ms (p95)

---

## Next Steps

This specification is complete and ready for Claude Code to:
1. Scaffold the entire project structure
2. Implement all core features
3. Set up integrations
4. Deploy to staging

**Suggested Claude Code prompts:**

```
"Initialize a React Native Expo app with TypeScript using the structure 
defined in this specification. Set up navigation, authentication, and 
the voice interface screen."

"Build the Fastify backend API with all routes, middleware, and services 
defined in the specification. Use Drizzle ORM for database access."

"Implement the ClickUp integration service with sync logic, webhook handler, 
and custom field mapping as specified."

"Create the voice intent parser and TTS response formatter with all 
supported commands from the specification."
```

---

**This specification contains everything Claude Code needs to build the entire application autonomously.**
