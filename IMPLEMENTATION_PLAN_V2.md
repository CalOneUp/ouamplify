# Amplifier Platform - Implementation Plan (No HubDB Access)

## Updated Recommendation: External Database + HubSpot Serverless Functions

Since you don't have HubDB access, here are your options:

---

## Option 1: Hybrid Approach ⭐ **RECOMMENDED**

**Architecture:**
- **Frontend:** HubSpot Module (keep existing)
- **Backend:** HubSpot Serverless Functions
- **Database:** External (Supabase, MongoDB Atlas, or PlanetScale)
- **Auth:** Custom JWT via serverless functions

**Pros:**
- ✅ Keep existing HubSpot module
- ✅ No need to rebuild frontend
- ✅ Flexible database choice
- ✅ Lower cost than full external hosting
- ✅ Can still leverage HubSpot infrastructure

**Cons:**
- ❌ Need external database service
- ❌ More complex than HubDB
- ❌ Additional service to manage

**Cost:** ~$25-50/month (database hosting)

---

## Option 2: Full External Hosting

**Architecture:**
- **Frontend:** Next.js/React on Netlify/Vercel
- **Backend:** API routes (Next.js) or serverless functions
- **Database:** Supabase/PostgreSQL or MongoDB
- **Auth:** Clerk, Auth0, or Supabase Auth

**Pros:**
- ✅ Full control
- ✅ Modern tech stack
- ✅ Better scalability
- ✅ No HubSpot limitations

**Cons:**
- ❌ Need to rebuild frontend
- ❌ More setup time
- ❌ Higher complexity

**Cost:** ~$50-100/month (hosting + database + auth)

---

## Recommended: Option 1 (Hybrid)

### Why Hybrid?
1. **Faster to market** - Keep existing module
2. **Lower cost** - Just database hosting
3. **Flexibility** - Can migrate later if needed
4. **Leverage HubSpot** - Still use HubSpot hosting

---

## Implementation Plan: Hybrid Approach

### Phase 1: Database Setup (Week 1)

#### 1.1 Choose Database Provider

**Recommended: Supabase (PostgreSQL)**
- Free tier: 500MB database, 2GB bandwidth
- Easy to use
- Built-in auth (can use later)
- PostgreSQL (industry standard)

**Alternative: MongoDB Atlas**
- Free tier: 512MB storage
- NoSQL (more flexible)
- Easy scaling

**Alternative: PlanetScale (MySQL)**
- Free tier available
- Serverless MySQL
- Great for relational data

#### 1.2 Database Schema

**Table: `users`**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255),
  department VARCHAR(255),
  domain VARCHAR(255),
  is_admin BOOLEAN DEFAULT FALSE,
  is_verified BOOLEAN DEFAULT FALSE,
  participated_drops JSONB DEFAULT '[]',
  total_hype INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP
);
```

**Table: `allowed_domains`**
```sql
CREATE TABLE allowed_domains (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  domain VARCHAR(255) UNIQUE NOT NULL,
  verified_by VARCHAR(255),
  verified_at TIMESTAMP,
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Table: `drops`**
```sql
CREATE TABLE drops (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  objective TEXT,
  status VARCHAR(50) DEFAULT 'draft',
  instructions JSONB,
  visual_assets JSONB DEFAULT '[]',
  content_options JSONB DEFAULT '[]',
  post_link TEXT,
  points INTEGER DEFAULT 1000,
  created_by VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  start_date TIMESTAMP,
  end_date TIMESTAMP
);
```

**Table: `drop_performance`**
```sql
CREATE TABLE drop_performance (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  drop_id UUID REFERENCES drops(id),
  user_email VARCHAR(255) REFERENCES users(email),
  clicks_generated INTEGER DEFAULT 0,
  participation_date TIMESTAMP DEFAULT NOW(),
  unique_link TEXT
);
```

**Table: `drop_analytics`**
```sql
CREATE TABLE drop_analytics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  drop_id UUID REFERENCES drops(id) UNIQUE,
  total_clicks INTEGER DEFAULT 0,
  total_participants INTEGER DEFAULT 0,
  team_breakdown JSONB,
  mvp_leaderboard JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Phase 2: Serverless Functions (Week 2-3)

#### 2.1 Database Connection

**`/functions/lib/db.js`**
```javascript
const { createClient } = require('@supabase/supabase-js');

const supabaseUrl = process.env.SUPABASE_URL;
const supabaseKey = process.env.SUPABASE_KEY;

const supabase = createClient(supabaseUrl, supabaseKey);

module.exports = { supabase };
```

#### 2.2 Authentication Functions

**`/functions/api/auth/login.js`**
- Validates email domain
- Creates/updates user in database
- Generates JWT token
- Returns session data

**`/functions/api/auth/verify-domain.js`**
- Checks if domain is allowed
- Sends verification email if new
- Creates pending domain entry

**`/functions/api/auth/check-session.js`**
- Validates JWT token
- Returns user data

#### 2.3 Drop Management Functions

**`/functions/api/drops/create.js`** (Admin only)
- Creates drop in database
- Validates admin permissions
- Returns drop ID

**`/functions/api/drops/list.js`**
- Fetches active drops
- Filters by status

**`/functions/api/drops/participate.js`**
- Records participation
- Generates unique link
- Updates performance data

#### 2.4 Performance Functions

**`/functions/api/performance/record-click.js`**
- Records click event
- Updates performance table
- Calculates hype points

**`/functions/api/performance/leaderboard.js`**
- Calculates scores
- Returns sorted leaderboard

### Phase 3: Frontend Integration (Week 4)

#### 3.1 Update Module to Use API

Replace client-side state with API calls:
- Login → `/api/auth/login`
- Get drops → `/api/drops/list`
- Participate → `/api/drops/participate`
- Record clicks → `/api/performance/record-click`

#### 3.2 Session Management

- Store JWT in httpOnly cookie
- Check session on page load
- Redirect to login if expired

### Phase 4: Admin Dashboard (Week 5)

#### 4.1 Admin Interface

Create admin module or separate page:
- Create/edit drops
- Manage users
- View analytics
- Domain management

---

## Quick Start: Supabase Setup

### Step 1: Create Supabase Project

1. Go to [supabase.com](https://supabase.com)
2. Create new project
3. Note your project URL and API key

### Step 2: Set Up Database

Run the SQL schema above in Supabase SQL editor

### Step 3: Add Secrets to HubSpot

```bash
hs secrets add SUPABASE_URL
hs secrets add SUPABASE_KEY
hs secrets add JWT_SECRET
```

### Step 4: Create First Serverless Function

**`/functions/api/test-db.js`**
```javascript
const { supabase } = require('../lib/db');

exports.main = async (event, callback) => {
  const { data, error } = await supabase
    .from('users')
    .select('*')
    .limit(5);
  
  callback({
    statusCode: 200,
    body: JSON.stringify({ data, error })
  });
};
```

---

## Cost Comparison

### Hybrid Approach (Recommended)
- **Supabase Free Tier:** $0/month (up to 500MB)
- **Supabase Pro (if needed):** $25/month
- **HubSpot:** $0 (using existing)
- **Total:** $0-25/month

### Full External Hosting
- **Netlify Pro:** $19/month
- **Supabase:** $25/month
- **Clerk Auth:** $25/month (or free tier)
- **Total:** $69/month minimum

---

## Timeline

### Hybrid Approach: 5 weeks
- Week 1: Database setup & schema
- Week 2-3: Serverless functions
- Week 4: Frontend integration
- Week 5: Admin dashboard & testing

### Full External: 8-10 weeks
- Additional time to rebuild frontend

---

## Next Steps

1. **Choose database provider** (Supabase recommended)
2. **Set up Supabase project**
3. **Create database schema**
4. **Create first serverless function** (test connection)
5. **Update module to use API**

---

## Migration Path

If you later get HubDB access or want to move:
- Export data from external database
- Import to HubDB or new system
- Update API endpoints
- No frontend changes needed

---

## Recommendation Summary

**Go with Hybrid Approach:**
- ✅ Fastest to market
- ✅ Lowest cost
- ✅ Keep existing module
- ✅ Flexible for future changes

**Start with Supabase:**
- Free tier is generous
- Easy to use
- PostgreSQL (standard)
- Can scale as needed


