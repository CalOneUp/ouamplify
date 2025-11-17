# Amplifier Platform - Final Implementation Plan

## Decision: Full External Hosting (Recommended)

**Why move away from HubSpot:**
- ❌ Need external database anyway (Firebase)
- ❌ Secrets management is easier on Netlify/Vercel
- ❌ HubSpot serverless functions add unnecessary complexity
- ❌ Can use Firebase directly from frontend
- ❌ No HubSpot limitations or vendor lock-in
- ✅ Simpler architecture
- ✅ Better developer experience
- ✅ More control

---

## Recommended Architecture

```
Next.js/React App (Netlify/Vercel)
    ↓
Firebase Firestore (Database)
Firebase Auth (Authentication)
Firebase Functions (Optional - for admin operations)
```

**Stack:**
- **Frontend:** Next.js + React (or Vite + React)
- **Hosting:** Netlify or Vercel (free tier)
- **Database:** Firebase Firestore
- **Auth:** Firebase Authentication
- **Styling:** Tailwind CSS (already using)
- **Deployment:** GitHub → Netlify/Vercel (automatic)

---

## Why This Makes Sense

### 1. Simpler Architecture
- No HubSpot serverless functions as middleman
- Firebase SDK works directly in frontend
- Fewer moving parts

### 2. Better Secrets Management
- Environment variables in Netlify/Vercel
- No HubSpot secrets limitations
- Easier to manage

### 3. Cost
- **Netlify:** Free tier (100GB bandwidth, 300 build minutes)
- **Vercel:** Free tier (100GB bandwidth, unlimited builds)
- **Firebase:** Free tier (generous limits)
- **Total:** $0/month for MVP

### 4. Developer Experience
- Modern React/Next.js
- Hot reload
- Better debugging
- TypeScript support
- Better tooling

### 5. Flexibility
- Can add features HubSpot can't support
- No vendor lock-in
- Easy to scale

---

## Project Structure

```
amplifier-platform/
├── src/
│   ├── components/
│   │   ├── Login.jsx
│   │   ├── DropCard.jsx
│   │   ├── Leaderboard.jsx
│   │   ├── AdminDashboard.jsx
│   │   └── ...
│   ├── pages/
│   │   ├── index.jsx          # Main app
│   │   ├── admin.jsx          # Admin dashboard
│   │   └── recap.jsx          # Drop recap
│   ├── lib/
│   │   ├── firebase.js        # Firebase config
│   │   ├── auth.js            # Auth helpers
│   │   └── db.js              # Firestore helpers
│   ├── hooks/
│   │   ├── useAuth.js
│   │   ├── useDrops.js
│   │   └── useLeaderboard.js
│   └── App.jsx
├── public/
│   └── ...
├── package.json
├── .env.local                 # Firebase config (gitignored)
├── netlify.toml              # Netlify config
└── README.md
```

---

## Setup Steps

### Step 1: Create Next.js/React App

**Option A: Next.js (Recommended)**
```bash
npx create-next-app@latest amplifier-platform
cd amplifier-platform
npm install firebase tailwindcss
```

**Option B: Vite + React**
```bash
npm create vite@latest amplifier-platform -- --template react
cd amplifier-platform
npm install
npm install firebase tailwindcss
```

### Step 2: Set Up Firebase

1. Create Firebase project
2. Enable Firestore
3. Enable Authentication (Email/Password)
4. Get Firebase config
5. Add to `.env.local`:

```env
NEXT_PUBLIC_FIREBASE_API_KEY=your-api-key
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your-project-id
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
NEXT_PUBLIC_FIREBASE_APP_ID=your-app-id
```

### Step 3: Firebase Configuration

**`src/lib/firebase.js`**
```javascript
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
export const db = getFirestore(app);
```

### Step 4: Deploy to Netlify

1. Push to GitHub
2. Connect Netlify to GitHub repo
3. Add environment variables in Netlify dashboard
4. Deploy automatically

**`netlify.toml`**
```toml
[build]
  command = "npm run build"
  publish = ".next"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

---

## Key Components

### Authentication Hook

**`src/hooks/useAuth.js`**
```javascript
import { useState, useEffect } from 'react';
import { onAuthStateChanged, signInWithEmailAndPassword, signOut } from 'firebase/auth';
import { doc, getDoc, setDoc } from 'firebase/firestore';
import { auth, db } from '../lib/firebase';

export function useAuth() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (firebaseUser) => {
      if (firebaseUser) {
        // Get user data from Firestore
        const userDoc = await getDoc(doc(db, 'users', firebaseUser.email));
        if (userDoc.exists()) {
          setUser({ ...userDoc.data(), email: firebaseUser.email });
        }
      } else {
        setUser(null);
      }
      setLoading(false);
    });

    return unsubscribe;
  }, []);

  const login = async (email, password) => {
    // Verify domain first
    const domain = email.split('@')[1];
    const domainDoc = await getDoc(doc(db, 'allowedDomains', domain));
    
    if (!domainDoc.exists() || domainDoc.data().status !== 'approved') {
      throw new Error('Domain not approved');
    }

    // Sign in with Firebase
    const userCredential = await signInWithEmailAndPassword(auth, email, password);
    
    // Get or create user document
    const userRef = doc(db, 'users', email);
    const userDoc = await getDoc(userRef);
    
    if (!userDoc.exists()) {
      await setDoc(userRef, {
        email,
        name: email.split('@')[0],
        department: 'Unassigned',
        domain,
        isAdmin: false,
        isVerified: true,
        participatedDrops: [],
        totalHype: 0,
        createdAt: new Date(),
        lastLogin: new Date()
      });
    } else {
      await setDoc(userRef, { lastLogin: new Date() }, { merge: true });
    }
  };

  const logout = () => signOut(auth);

  return { user, loading, login, logout };
}
```

### Real-time Drops Hook

**`src/hooks/useDrops.js`**
```javascript
import { useState, useEffect } from 'react';
import { collection, query, where, onSnapshot } from 'firebase/firestore';
import { db } from '../lib/firebase';

export function useDrops(status = 'active') {
  const [drops, setDrops] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const q = query(
      collection(db, 'drops'),
      where('status', '==', status)
    );

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const dropsData = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setDrops(dropsData);
      setLoading(false);
    });

    return unsubscribe;
  }, [status]);

  return { drops, loading };
}
```

### Real-time Leaderboard Hook

**`src/hooks/useLeaderboard.js`**
```javascript
import { useState, useEffect } from 'react';
import { collection, query, orderBy, onSnapshot } from 'firebase/firestore';
import { db } from '../lib/firebase';

export function useLeaderboard() {
  const [leaderboard, setLeaderboard] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const q = query(
      collection(db, 'users'),
      orderBy('totalHype', 'desc')
    );

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const leaderboardData = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setLeaderboard(leaderboardData);
      setLoading(false);
    });

    return unsubscribe;
  }, []);

  return { leaderboard, loading };
}
```

---

## Migration from HubSpot Module

### What to Keep
- ✅ UI design and styling
- ✅ Component structure
- ✅ Business logic
- ✅ Tailwind CSS classes

### What to Change
- 🔄 Convert HubL to React components
- 🔄 Replace client-side state with Firebase
- 🔄 Add React hooks for data fetching
- 🔄 Implement proper routing

### Migration Steps

1. **Extract HTML/CSS from HubSpot module**
2. **Convert to React components**
3. **Add Firebase integration**
4. **Set up routing**
5. **Deploy to Netlify**

---

## Deployment

### Netlify Setup

1. **Create `netlify.toml`:**
```toml
[build]
  command = "npm run build"
  publish = ".next"

[build.environment]
  NODE_VERSION = "18"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

2. **Add Environment Variables:**
   - Go to Netlify Dashboard → Site Settings → Environment Variables
   - Add all `NEXT_PUBLIC_FIREBASE_*` variables

3. **Connect GitHub:**
   - Netlify → Add new site → Import from Git
   - Select repository
   - Deploy automatically on push

### Vercel Setup (Alternative)

1. **Install Vercel CLI:**
```bash
npm i -g vercel
```

2. **Deploy:**
```bash
vercel
```

3. **Add Environment Variables:**
   - Vercel Dashboard → Project Settings → Environment Variables

---

## Cost Breakdown

### Free Tier (MVP)
- **Netlify/Vercel:** Free
  - 100GB bandwidth
  - Unlimited builds (Vercel) or 300 min/month (Netlify)
- **Firebase:** Free
  - 1GB storage
  - 50K reads/day
  - 20K writes/day
  - 50K MAU

### Total: **$0/month** for MVP

### If You Scale
- **Netlify Pro:** $19/month (if needed)
- **Firebase Blaze:** Pay as you go (still very cheap)

---

## Timeline: 6-8 weeks

- **Week 1:** Set up Next.js project, Firebase, basic structure
- **Week 2-3:** Build authentication, user management
- **Week 4-5:** Build drops, performance tracking
- **Week 6:** Admin dashboard
- **Week 7:** Real-time features, polish
- **Week 8:** Testing, deployment, launch

---

## Next Steps

1. ✅ **Create Next.js project**
2. ✅ **Set up Firebase**
3. ✅ **Build authentication**
4. ✅ **Migrate UI from HubSpot module**
5. ✅ **Add Firebase integration**
6. ✅ **Deploy to Netlify**

---

## Recommendation

**Go with full external hosting:**
- ✅ Simpler architecture
- ✅ Better developer experience
- ✅ Easier secrets management
- ✅ More flexibility
- ✅ Still free for MVP
- ✅ Can reuse most of your existing code

The HubSpot module was a great prototype, but for production with Firebase, external hosting makes much more sense.


