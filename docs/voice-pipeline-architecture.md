# Voice Pipeline App - Technical Architecture

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    MOBILE APP (React Native)                 │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │  Voice UI  │  │  Visual List │  │  Loan Detail     │    │
│  │  "Pipeline"│  │  View        │  │  View            │    │
│  │  Tap-Talk  │  │  Swipe/Tap   │  │  Timeline/Docs   │    │
│  └─────┬──────┘  └──────┬───────┘  └────────┬─────────┘    │
│        │                │                    │               │
│        └────────────────┴────────────────────┘               │
│                         │                                    │
│                    ┌────▼─────┐                             │
│                    │ Auth     │ (Okta OIDC)                 │
│                    │ Layer    │                             │
│                    └────┬─────┘                             │
│                         │                                    │
│                    ┌────▼─────┐                             │
│                    │ Local    │ (Offline cache)             │
│                    │ Storage  │ (Last 48hrs data)           │
│                    └──────────┘                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  BACKEND API (Node.js/Fastify)               │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Voice Processing Layer                  │   │
│  │  ┌────────────┐    ┌──────────────┐                │   │
│  │  │ STT Engine │───▶│ Intent Parse │                │   │
│  │  │ (Wispr)    │    │ (Keywords)   │                │   │
│  │  └────────────┘    └──────────────┘                │   │
│  │                           │                          │   │
│  │                    ┌──────▼────────┐                │   │
│  │                    │ Query Builder │                │   │
│  │                    │ (SQL/Filter)  │                │   │
│  │                    └───────────────┘                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Data Aggregation Layer                │   │
│  │  ┌──────────────┐  ┌──────────────┐                │   │
│  │  │ ClickUp      │  │ Local Cache  │                │   │
│  │  │ Connector    │  │ (PostgreSQL) │                │   │
│  │  └──────┬───────┘  └──────┬───────┘                │   │
│  │         │                  │                         │   │
│  │         └──────────┬───────┘                         │   │
│  │                    │                                  │   │
│  │            ┌───────▼────────┐                        │   │
│  │            │ Loan Formatter │                        │   │
│  │            │ (TTS-ready)    │                        │   │
│  │            └────────────────┘                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Future: Encompass Layer                 │   │
│  │  (OAuth, eFolder API, Document Pull)                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         ┌────▼─────┐            ┌─────▼──────┐
         │ ClickUp  │            │ PostgreSQL │
         │ API      │            │ (Supabase) │
         └──────────┘            └────────────┘
```

## Data Flow Examples

### Voice Query Flow
```
User: "Hey Pipeline, what's the status of the Johnson loan?"
  │
  ├─▶ STT (Wispr): Transcribe audio
  │
  ├─▶ Intent Parser: Extract {action: "status", borrower: "Johnson"}
  │
  ├─▶ Query Builder: Search ClickUp tasks WHERE name LIKE "%Johnson%"
  │
  ├─▶ Data Formatter: 
  │     "Johnson loan, 123 Main St, is in underwriting.
  │      Currently at day 12, waiting on 3 conditions.
  │      Closing date is November 15th.
  │      Last update: VOE received yesterday."
  │
  └─▶ TTS: Speak response + show visual card
```

### Visual Browse Flow
```
User: Opens app ─▶ Sees list of all active loans
  │
  ├─▶ Tap "Johnson Loan" ─▶ Detail view opens
  │
  └─▶ Shows:
       • Timeline (milestones)
       • Conditions checklist
       • Recent updates (from ClickUp comments)
       • Quick actions (Call client, Update stage)
```

## Component Breakdown

### Mobile App (React Native + Expo)
```
src/
├── app/
│   ├── (auth)/
│   │   └── login.tsx              # Okta login flow
│   ├── (tabs)/
│   │   ├── index.tsx              # Voice interface (main)
│   │   ├── pipeline.tsx           # List view of all loans
│   │   └── settings.tsx           # User preferences
│   └── loan/[id].tsx              # Loan detail screen
├── components/
│   ├── VoiceButton.tsx            # Tap-to-talk + "Pipeline" wake
│   ├── LoanCard.tsx               # Visual loan summary
│   ├── ConditionsList.tsx         # Checklist UI
│   └── AudioWaveform.tsx          # Visual feedback during speech
├── services/
│   ├── voice.ts                   # STT/TTS wrapper
│   ├── api.ts                     # Backend API client
│   ├── okta.ts                    # Auth flow
│   └── offline.ts                 # Local cache management
└── utils/
    ├── intentParser.ts            # Parse voice commands locally
    └── formatters.ts              # Format data for TTS
```

### Backend API (Node.js + Fastify)
```
src/
├── routes/
│   ├── auth.ts                    # Okta token validation
│   ├── loans.ts                   # GET /loans, GET /loans/:id
│   ├── voice.ts                   # POST /voice/query
│   └── sync.ts                    # POST /sync (ClickUp webhook)
├── services/
│   ├── clickup.ts                 # ClickUp API wrapper
│   ├── encompass.ts               # Future: Encompass integration
│   ├── voice/
│   │   ├── stt.ts                 # Speech-to-text
│   │   ├── tts.ts                 # Text-to-speech
│   │   └── intent.ts              # Natural language understanding
│   └── cache.ts                   # PostgreSQL data layer
├── middleware/
│   ├── auth.ts                    # JWT validation
│   ├── rateLimit.ts               # Prevent abuse
│   └── audit.ts                   # Log all access (GLBA)
└── utils/
    ├── logger.ts                  # Structured logging
    └── errors.ts                  # Error handling
```

### Database Schema (PostgreSQL)
```sql
-- Cached loan data (synced from ClickUp every 15 min)
CREATE TABLE loans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clickup_task_id TEXT UNIQUE NOT NULL,
  borrower_name TEXT NOT NULL,
  property_address TEXT,
  loan_stage TEXT NOT NULL,
  stage_updated_at TIMESTAMPTZ,
  closing_date DATE,
  conditions_count INT DEFAULT 0,
  loan_officer_id UUID REFERENCES users(id),
  processor_id UUID REFERENCES users(id),
  last_synced_at TIMESTAMPTZ DEFAULT NOW(),
  metadata JSONB -- Custom fields from ClickUp
);

-- Conditions tracking
CREATE TABLE conditions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  loan_id UUID REFERENCES loans(id) ON DELETE CASCADE,
  description TEXT NOT NULL,
  status TEXT CHECK (status IN ('pending', 'submitted', 'cleared')),
  due_date DATE,
  cleared_at TIMESTAMPTZ
);

-- Activity timeline (from ClickUp comments/updates)
CREATE TABLE loan_activities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  loan_id UUID REFERENCES loans(id) ON DELETE CASCADE,
  activity_type TEXT NOT NULL, -- 'comment', 'stage_change', 'doc_upload'
  description TEXT NOT NULL,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users (synced from Okta)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  okta_user_id TEXT UNIQUE NOT NULL,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  role TEXT CHECK (role IN ('loan_officer', 'processor', 'admin')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Audit log (GLBA compliance)
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  action TEXT NOT NULL, -- 'view_loan', 'query_voice', 'update_stage'
  resource_id TEXT, -- loan_id or task_id
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_loans_borrower ON loans(borrower_name);
CREATE INDEX idx_loans_officer ON loans(loan_officer_id);
CREATE INDEX idx_loans_stage ON loans(loan_stage);
CREATE INDEX idx_activities_loan ON loan_activities(loan_id, created_at DESC);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id, created_at DESC);
```

## Voice Command Examples

### Supported Queries (Phase 1)
```
"Pipeline" or "Hey Pipeline" → Activates listening

"What's my pipeline?" / "Show me my loans"
  ↳ Reads summary: "You have 12 active loans. 3 in processing, 
    5 in underwriting, 4 clear to close."

"Status of Johnson loan" / "Johnson loan update"
  ↳ Reads: Stage, days in stage, conditions, closing date

"What conditions does Smith need?" / "Smith conditions"
  ↳ Reads outstanding conditions list

"When does Garcia close?" / "Garcia closing date"
  ↳ Reads: "Garcia loan closes November 15th, in 21 days"

"What loans close this week?"
  ↳ Lists all loans closing in next 7 days

"Any updates today?" / "What's new?"
  ↳ Reads recent activities across all user's loans
```

### Future Commands (Phase 2+)
```
"Move Johnson to underwriting" → Updates ClickUp stage
"Add note to Smith loan: VOE received" → Creates comment
"Call Garcia" → Dials borrower phone number
"Send me Johnson conditions" → Texts/emails conditions list
```

## Integration Details

### ClickUp API Integration
```typescript
// services/clickup.ts
interface ClickUpTask {
  id: string;
  name: string; // "Johnson, Mike & Sarah - 123 Main St"
  status: { status: string }; // "In Processing"
  custom_fields: Array<{
    name: string;
    value: any;
  }>;
  date_updated: string;
  tags: Array<{ name: string }>;
}

// Map custom fields to our schema
const CUSTOM_FIELD_MAP = {
  'Closing Date': 'closing_date',
  'Loan Officer': 'loan_officer_name',
  'Processor': 'processor_name',
  'Property Address': 'property_address',
  'Conditions Count': 'conditions_count',
  'Client Phone': 'client_phone',
  'Client Email': 'client_email'
};

async function syncLoansFromClickUp() {
  const tasks = await clickup.getTasks({
    list_id: PIPELINE_LIST_ID,
    statuses: ['Processing', 'Underwriting', 'CTC', 'Docs Out']
  });
  
  for (const task of tasks) {
    await upsertLoan(mapTaskToLoan(task));
  }
}
```

### Okta Authentication Flow
```typescript
// services/okta.ts
import { OktaAuth } from '@okta/okta-react-native';

const oktaAuth = new OktaAuth({
  issuer: 'https://your-domain.okta.com/oauth2/default',
  clientId: process.env.OKTA_CLIENT_ID,
  redirectUri: 'com.truenorth.pipeline://callback',
  scopes: ['openid', 'profile', 'email', 'offline_access']
});

// On app launch
async function authenticate() {
  const tokens = await oktaAuth.signInWithBrowser();
  // Store access token securely
  await SecureStore.setItemAsync('auth_token', tokens.accessToken);
  return tokens;
}
```

## Offline Strategy

**Problem:** LOs in areas with spotty coverage need data access.

**Solution:**
1. **Sync on connect:** When app opens with internet, download all user's loans
2. **48-hour cache:** Store last 2 days of data locally (encrypted)
3. **Smart refresh:** Background sync every 15 minutes when connected
4. **Offline indicator:** Show "Last updated X minutes ago" badge
5. **Queue writes:** If user tries to update offline, queue until reconnected

```typescript
// services/offline.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import * as Crypto from 'expo-crypto';

const CACHE_KEY = 'loans_cache';
const CACHE_DURATION = 48 * 60 * 60 * 1000; // 48 hours

async function cacheLoans(loans: Loan[]) {
  const encrypted = await Crypto.encryptAsync(
    JSON.stringify({ loans, timestamp: Date.now() }),
    await getEncryptionKey()
  );
  await AsyncStorage.setItem(CACHE_KEY, encrypted);
}

async function getCachedLoans(): Promise<Loan[] | null> {
  const encrypted = await AsyncStorage.getItem(CACHE_KEY);
  if (!encrypted) return null;
  
  const decrypted = await Crypto.decryptAsync(encrypted, await getEncryptionKey());
  const { loans, timestamp } = JSON.parse(decrypted);
  
  // Check if cache is stale
  if (Date.now() - timestamp > CACHE_DURATION) {
    return null; // Force refresh
  }
  
  return loans;
}
```

