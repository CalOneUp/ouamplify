# Amplifier Platform - Production Implementation Plan

## Executive Summary

This document outlines the plan to transform the Amplifier Platform from a prototype into a production-ready social thunderclap platform with authentication, admin controls, and persistent data storage.

## Architecture Decision: HubSpot vs. External Hosting

### Option 1: HubSpot-Native (Recommended for MVP)

**Pros:**
- ✅ Already integrated with your HubSpot infrastructure
- ✅ No additional hosting costs
- ✅ HubDB for database (included with CMS Professional/Enterprise)
- ✅ Serverless functions for backend logic
- ✅ HubSpot CRM integration for user management
- ✅ Built-in analytics and tracking
- ✅ Easy deployment via HubSpot CLI
- ✅ No separate infrastructure to manage

**Cons:**
- ❌ Limited to HubSpot's ecosystem
- ❌ Requires CMS Professional/Enterprise for HubDB
- ❌ Less flexibility for complex custom auth flows
- ❌ HubSpot API rate limits
- ❌ Vendor lock-in

**Best For:** Internal company tool, MVP, quick deployment

### Option 2: External Hosting (GitHub + Netlify/Vercel)

**Pros:**
- ✅ Full control over infrastructure
- ✅ Modern tech stack (React, Next.js, etc.)
- ✅ Flexible authentication (Auth0, Clerk, Supabase Auth)
- ✅ Better scalability
- ✅ No vendor lock-in
- ✅ Can integrate with any database (PostgreSQL, MongoDB, etc.)
- ✅ Better developer experience

**Cons:**
- ❌ Additional hosting costs
- ❌ Need to rebuild from HubSpot module
- ❌ Separate infrastructure to manage
- ❌ More complex deployment
- ❌ Need to handle security, backups, etc.

**Best For:** Public SaaS product, complex requirements, maximum flexibility

## ⚠️ UPDATE: No HubDB Access

**Current Situation:** You don't have access to HubDB (requires CMS Professional/Enterprise)

**Impact:** Cannot use HubSpot's native database solution

**New Recommendation: External Database + HubSpot Serverless Functions OR Full External Hosting**

### Option A: Hybrid Approach (HubSpot Module + External DB) ⭐ RECOMMENDED
- Keep HubSpot module for frontend
- Use external database (Supabase, MongoDB Atlas, etc.)
- HubSpot serverless functions connect to external DB
- Best of both worlds: HubSpot hosting + flexible database

### Option B: Full External Hosting
- Move to GitHub + Netlify/Vercel
- Complete control and flexibility
- More setup required

---

## Implementation Plan: External Database + HubSpot (Recommended)

Since HubDB is not available, we'll use an external database with HubSpot serverless functions.

### Phase 1: Authentication & User Management (Week 1-2)

#### 1.1 Email-Based Authentication
- **Implementation:**
  - Use HubSpot Memberships (requires CMS Enterprise) OR
  - Custom authentication via serverless functions
  - Email domain verification
  - Session management with JWT tokens stored in cookies

- **Database Schema:**
  - **HubDB Table: `amplifier_users`**
    - `email` (text, unique)
    - `name` (text)
    - `department` (text)
    - `domain` (text) - extracted from email
    - `is_admin` (boolean)
    - `is_verified` (boolean)
    - `created_at` (datetime)
    - `last_login` (datetime)
    - `participated_drops` (text) - JSON array of drop IDs

#### 1.2 Domain Verification
- **Flow:**
  1. User signs in with work email
  2. Extract domain from email (e.g., `@company.com`)
  3. Check against allowed domains table
  4. If domain not in list, send verification email to domain admin
  5. Admin approves domain → all users from that domain can access

- **HubDB Table: `allowed_domains`**
  - `domain` (text, unique) - e.g., "company.com"
  - `verified_by` (text) - admin email who verified
  - `verified_at` (datetime)
  - `status` (choice) - "pending", "approved", "rejected"

#### 1.3 Admin Access Control
- **HubDB Table: `admin_users`**
  - `email` (text, unique)
  - `permissions` (text) - JSON: `["create_drops", "manage_users", "view_analytics"]`
  - `created_by` (text)
  - `created_at` (datetime)

- **Serverless Function: `check-admin-access`**
  - Validates if user has admin permissions
  - Returns permission level

### Phase 2: Database Architecture (Week 2-3)

#### 2.1 HubDB Tables Structure

**Table 1: `drops`**
```
- id (text, unique)
- name (text)
- objective (text)
- status (choice): "draft", "active", "completed", "archived"
- instructions (text) - JSON array
- visual_assets (text) - JSON array
- content_options (text) - JSON array
- post_link (text)
- points (number)
- created_by (text) - admin email
- created_at (datetime)
- start_date (datetime)
- end_date (datetime)
```

**Table 2: `crew_members`**
```
- email (text, unique)
- name (text)
- department (text)
- participated_drops (text) - JSON array
- total_hype (number)
- created_at (datetime)
```

**Table 3: `drop_performance`**
```
- drop_id (text)
- user_email (text)
- clicks_generated (number)
- participation_date (datetime)
- unique_link (text)
```

**Table 4: `drop_analytics`** (for recap data)
```
- drop_id (text)
- total_clicks (number)
- total_participants (number)
- team_breakdown (text) - JSON object
- mvp_leaderboard (text) - JSON array
- created_at (datetime)
```

#### 2.2 Data Relationships
- `drops` → `drop_performance` (one-to-many)
- `crew_members` → `drop_performance` (one-to-many)
- `drops` → `drop_analytics` (one-to-one)

### Phase 3: Serverless Functions (Week 3-4)

#### 3.1 Authentication Functions

**`/api/auth/login`**
- Validates email domain
- Creates/updates user session
- Returns JWT token

**`/api/auth/logout`**
- Invalidates session
- Clears cookies

**`/api/auth/verify-domain`**
- Sends verification email to domain admin
- Creates pending domain entry

**`/api/auth/check-session`**
- Validates current session
- Returns user data

#### 3.2 Drop Management Functions

**`/api/drops/create`** (Admin only)
- Creates new drop in HubDB
- Validates admin permissions
- Returns drop ID

**`/api/drops/update`** (Admin only)
- Updates existing drop
- Validates admin permissions

**`/api/drops/list`**
- Returns active drops
- Filters by status

**`/api/drops/get/:id`**
- Returns single drop details

**`/api/drops/participate`**
- Records user participation
- Generates unique tracking link
- Updates performance data

#### 3.3 Performance Functions

**`/api/performance/record-click`**
- Records click event
- Updates drop_performance table
- Calculates hype points

**`/api/performance/get-leaderboard`**
- Calculates crew scores
- Returns sorted leaderboard

**`/api/performance/get-recap/:dropId`**
- Aggregates performance data
- Returns recap statistics

#### 3.4 AI Integration Functions

**`/api/ai/suggest-copy`**
- Calls Gemini API (or OpenAI)
- Generates copy suggestions
- Returns formatted options

**`/api/ai/generate-summary`**
- Generates drop recap summary
- Uses performance data as context

### Phase 4: Frontend Updates (Week 4-5)

#### 4.1 Authentication UI
- Replace name-based login with email login
- Add domain verification flow
- Add admin dashboard access

#### 4.2 Admin Dashboard
- Create drop interface
- Manage users
- View analytics
- Domain management

#### 4.3 Data Integration
- Replace client-side state with API calls
- Real-time leaderboard updates
- Persistent session management

### Phase 5: Security & Performance (Week 5-6)

#### 5.1 Security
- JWT token validation
- CORS configuration
- Rate limiting on API endpoints
- Input sanitization
- XSS protection

#### 5.2 Performance
- Cache frequently accessed data
- Optimize HubDB queries
- Lazy load components
- Image optimization

---

## Technical Stack (HubSpot-Native)

### Frontend
- HubSpot Module (existing)
- Vanilla JavaScript (or React if using React modules)
- Tailwind CSS (already included)

### Backend
- HubSpot Serverless Functions (Node.js)
- HubDB for database
- HubSpot CRM for user profiles (optional)

### Authentication
- HubSpot Memberships (if CMS Enterprise) OR
- Custom JWT-based auth via serverless functions

### External Services
- Google Gemini API (for AI copy generation)
- Email service (HubSpot Email API or SendGrid)

---

## Migration Path to External Hosting (Future)

If you decide to move to external hosting later:

### Step 1: Export Data
- Export all HubDB tables to JSON/CSV
- Export user data
- Export performance metrics

### Step 2: Rebuild Frontend
- Convert HubSpot module to React/Next.js
- Recreate UI components
- Implement routing

### Step 3: Set Up Backend
- Deploy to Netlify/Vercel
- Set up database (PostgreSQL/Supabase)
- Migrate serverless functions to API routes

### Step 4: Authentication
- Implement Auth0/Clerk/Supabase Auth
- Migrate user accounts
- Set up domain verification

### Step 5: Data Migration
- Import HubDB data to new database
- Map relationships
- Verify data integrity

---

## Cost Analysis

### HubSpot-Native
- **HubSpot CMS Professional/Enterprise:** $300-900/month (if not already owned)
- **HubSpot API calls:** Included (with rate limits)
- **Serverless functions:** Included
- **HubDB:** Included
- **Total Additional Cost:** $0 (if already have CMS Professional+)

### External Hosting
- **Netlify Pro:** $19/month
- **Database (Supabase):** $25/month (or free tier)
- **Auth (Clerk):** $25/month (or free tier)
- **Email (SendGrid):** $15/month (free tier available)
- **Total:** ~$84/month minimum

---

## Timeline

### MVP (HubSpot-Native): 6 weeks
- Week 1-2: Authentication & User Management
- Week 3-4: Database & Serverless Functions
- Week 5: Frontend Integration
- Week 6: Testing & Deployment

### Full Production: 8-10 weeks
- Additional 2-4 weeks for:
  - Advanced features
  - Analytics dashboard
  - Email notifications
  - Performance optimization

---

## Next Steps

1. **Decide on architecture** (HubSpot vs. External)
2. **Set up HubDB tables** (if HubSpot-native)
3. **Create first serverless function** (authentication)
4. **Build admin interface** (drop creation)
5. **Test with small user group**
6. **Iterate based on feedback**

---

## Questions to Answer

1. Do you have HubSpot CMS Professional/Enterprise? (Required for HubDB)
2. How many users will use the platform initially?
3. Do you need multi-tenant support? (Multiple companies)
4. What's your budget for hosting?
5. Timeline for launch?

---

## Recommended Approach

**Start with HubSpot-Native MVP:**
1. Faster to market
2. Lower cost
3. Leverages existing infrastructure
4. Can migrate later if needed

**Consider External Hosting if:**
- You need to support multiple companies
- You want to sell this as a SaaS product
- You need features HubSpot can't provide
- You have budget for additional infrastructure

