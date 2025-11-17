# Amplifier Platform - Firebase Implementation Plan

## Architecture: HubSpot Module + Firebase

**Stack:**
- **Frontend:** HubSpot Module (existing)
- **Backend:** HubSpot Serverless Functions
- **Database:** Firebase Firestore (NoSQL)
- **Auth:** Firebase Authentication
- **Real-time:** Firestore real-time listeners (for live leaderboards)

---

## Why Firebase?

✅ **Free Tier is Generous:**
- 1GB storage
- 10GB/month network egress
- 50K reads/day, 20K writes/day
- Firebase Auth: 50K MAU free

✅ **Perfect for This Use Case:**
- Real-time updates (leaderboard changes instantly)
- Built-in authentication
- Easy to integrate with serverless functions
- NoSQL (flexible schema)

✅ **Google Infrastructure:**
- Reliable and scalable
- Global CDN
- Automatic backups

---

## Firebase Setup

### Step 1: Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click "Add project"
3. Name it "amplifier-platform"
4. Enable Google Analytics (optional)
5. Note your project ID

### Step 2: Enable Services

1. **Firestore Database:**
   - Go to Firestore Database
   - Click "Create database"
   - Start in **production mode** (we'll add security rules)
   - Choose location (us-central1 recommended)

2. **Authentication:**
   - Go to Authentication
   - Click "Get started"
   - Enable **Email/Password** provider

### Step 3: Get Credentials

1. Go to Project Settings (gear icon)
2. Scroll to "Your apps"
3. Click "Web" icon (`</>`)
4. Register app name: "Amplifier Platform"
5. Copy the Firebase config object

### Step 4: Create Service Account

1. Go to Project Settings → Service Accounts
2. Click "Generate new private key"
3. Download JSON file (keep secure!)
4. This is used by serverless functions

---

## Database Schema (Firestore Collections)

### Collection: `users`

**Document Structure:**
```javascript
{
  email: "user@company.com",
  name: "John Doe",
  department: "Marketing",
  domain: "company.com",
  isAdmin: false,
  isVerified: true,
  participatedDrops: ["drop-id-1", "drop-id-2"],
  totalHype: 5000,
  createdAt: Timestamp,
  lastLogin: Timestamp
}
```

**Document ID:** Use email as document ID for easy lookup

### Collection: `allowedDomains`

**Document Structure:**
```javascript
{
  domain: "company.com",
  verifiedBy: "admin@company.com",
  verifiedAt: Timestamp,
  status: "approved", // "pending", "approved", "rejected"
  createdAt: Timestamp
}
```

**Document ID:** Use domain as document ID

### Collection: `drops`

**Document Structure:**
```javascript
{
  name: "The Q3 Platform Drop",
  objective: "Drive awareness and sign-ups",
  status: "active", // "draft", "active", "completed", "archived"
  instructions: [
    "1. Download your preferred visual asset.",
    "2. Choose a copy option..."
  ],
  visualAssets: [
    {
      type: "image",
      url: "https://...",
      filename: "drop_banner.png"
    }
  ],
  contentOptions: [
    {
      type: "Statement",
      text: "It's here! So excited..."
    }
  ],
  postLink: "https://www.yourcompany.com/new-product",
  points: 1000,
  createdBy: "admin@company.com",
  createdAt: Timestamp,
  startDate: Timestamp,
  endDate: Timestamp
}
```

### Collection: `dropPerformance`

**Document Structure:**
```javascript
{
  dropId: "drop-id-123",
  userEmail: "user@company.com",
  clicksGenerated: 150,
  participationDate: Timestamp,
  uniqueLink: "https://...?utm_source=amplifier&utm_content=user"
}
```

**Document ID:** Auto-generated (Firestore handles this)

**Indexes Needed:**
- `dropId` (for querying by drop)
- `userEmail` (for querying by user)

### Collection: `dropAnalytics`

**Document Structure:**
```javascript
{
  dropId: "drop-id-123",
  totalClicks: 5000,
  totalParticipants: 25,
  teamBreakdown: {
    "Marketing": 2000,
    "Sales": 1500,
    "Engineering": 1500
  },
  mvpLeaderboard: [
    { name: "Alice", clicks: 500 },
    { name: "Bob", clicks: 400 }
  ],
  createdAt: Timestamp,
  updatedAt: Timestamp
}
```

**Document ID:** Use `dropId` as document ID (one-to-one with drops)

---

## Security Rules

**`firestore.rules`**
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Users can read their own data
    match /users/{userId} {
      allow read: if request.auth != null && request.auth.token.email == userId;
      allow write: if request.auth != null && request.auth.token.email == userId;
    }
    
    // Admins can manage domains
    match /allowedDomains/{domain} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && 
        get(/databases/$(database)/documents/users/$(request.auth.token.email)).data.isAdmin == true;
    }
    
    // Everyone can read active drops
    match /drops/{dropId} {
      allow read: if request.auth != null;
      allow create, update: if request.auth != null && 
        get(/databases/$(database)/documents/users/$(request.auth.token.email)).data.isAdmin == true;
    }
    
    // Users can read their own performance
    match /dropPerformance/{performanceId} {
      allow read: if request.auth != null && 
        resource.data.userEmail == request.auth.token.email;
      allow create: if request.auth != null;
    }
    
    // Everyone can read analytics
    match /dropAnalytics/{analyticsId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && 
        get(/databases/$(database)/documents/users/$(request.auth.token.email)).data.isAdmin == true;
    }
  }
}
```

---

## HubSpot Serverless Functions

### Project Structure

```
social-thunderclap/
├── functions/
│   ├── lib/
│   │   └── firebase.js          # Firebase admin SDK setup
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login.js
│   │   │   ├── logout.js
│   │   │   ├── verify-domain.js
│   │   │   └── check-session.js
│   │   ├── drops/
│   │   │   ├── create.js
│   │   │   ├── list.js
│   │   │   ├── get.js
│   │   │   └── participate.js
│   │   └── performance/
│   │       ├── record-click.js
│   │       ├── leaderboard.js
│   │       └── recap.js
│   └── serverless.json
```

### Firebase Admin Setup

**`functions/lib/firebase.js`**
```javascript
const admin = require('firebase-admin');

// Initialize Firebase Admin
if (!admin.apps.length) {
  const serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT);
  
  admin.initializeApp({
    credential: admin.credential.cert(serviceAccount)
  });
}

const db = admin.firestore();
const auth = admin.auth();

module.exports = { admin, db, auth };
```

### Authentication Functions

**`functions/api/auth/login.js`**
```javascript
const { db, auth } = require('../../lib/firebase');

exports.main = async (event, callback) => {
  const { email, password } = JSON.parse(event.body || '{}');
  
  try {
    // 1. Verify email domain
    const domain = email.split('@')[1];
    const domainDoc = await db.collection('allowedDomains').doc(domain).get();
    
    if (!domainDoc.exists || domainDoc.data().status !== 'approved') {
      return callback({
        statusCode: 403,
        body: JSON.stringify({ error: 'Domain not approved' })
      });
    }
    
    // 2. Get or create Firebase Auth user
    let firebaseUser;
    try {
      firebaseUser = await auth.getUserByEmail(email);
    } catch (error) {
      // User doesn't exist, create them
      firebaseUser = await auth.createUser({
        email: email,
        emailVerified: false
      });
    }
    
    // 3. Create custom token for client
    const customToken = await auth.createCustomToken(firebaseUser.uid);
    
    // 4. Update/create user document in Firestore
    const userRef = db.collection('users').doc(email);
    const userDoc = await userRef.get();
    
    if (!userDoc.exists) {
      await userRef.set({
        email: email,
        name: email.split('@')[0], // Default to email prefix
        department: 'Unassigned',
        domain: domain,
        isAdmin: false,
        isVerified: true,
        participatedDrops: [],
        totalHype: 0,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        lastLogin: admin.firestore.FieldValue.serverTimestamp()
      });
    } else {
      await userRef.update({
        lastLogin: admin.firestore.FieldValue.serverTimestamp()
      });
    }
    
    // 5. Return token and user data
    const userData = (await userRef.get()).data();
    
    callback({
      statusCode: 200,
      body: JSON.stringify({
        token: customToken,
        user: userData
      })
    });
    
  } catch (error) {
    callback({
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    });
  }
};
```

**`functions/api/auth/verify-domain.js`**
```javascript
const { db } = require('../../lib/firebase');

exports.main = async (event, callback) => {
  const { domain, adminEmail } = JSON.parse(event.body || '{}');
  
  try {
    const domainRef = db.collection('allowedDomains').doc(domain);
    const domainDoc = await domainRef.get();
    
    if (domainDoc.exists) {
      return callback({
        statusCode: 400,
        body: JSON.stringify({ error: 'Domain already exists' })
      });
    }
    
    // Create pending domain
    await domainRef.set({
      domain: domain,
      status: 'pending',
      verifiedBy: adminEmail,
      createdAt: admin.firestore.FieldValue.serverTimestamp()
    });
    
    // TODO: Send verification email to domain admin
    
    callback({
      statusCode: 200,
      body: JSON.stringify({ message: 'Domain verification requested' })
    });
    
  } catch (error) {
    callback({
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    });
  }
};
```

### Drop Management Functions

**`functions/api/drops/create.js`**
```javascript
const { db } = require('../../lib/firebase');

exports.main = async (event, callback) => {
  const { dropData, userEmail } = JSON.parse(event.body || '{}');
  
  try {
    // Verify admin
    const userDoc = await db.collection('users').doc(userEmail).get();
    if (!userDoc.exists || !userDoc.data().isAdmin) {
      return callback({
        statusCode: 403,
        body: JSON.stringify({ error: 'Admin access required' })
      });
    }
    
    // Create drop
    const dropRef = db.collection('drops').doc();
    await dropRef.set({
      ...dropData,
      createdBy: userEmail,
      createdAt: admin.firestore.FieldValue.serverTimestamp()
    });
    
    callback({
      statusCode: 200,
      body: JSON.stringify({ dropId: dropRef.id })
    });
    
  } catch (error) {
    callback({
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    });
  }
};
```

**`functions/api/drops/list.js`**
```javascript
const { db } = require('../../lib/firebase');

exports.main = async (event, callback) => {
  const { status } = event.queryStringParameters || {};
  
  try {
    let query = db.collection('drops');
    
    if (status) {
      query = query.where('status', '==', status);
    }
    
    const snapshot = await query.get();
    const drops = snapshot.docs.map(doc => ({
      id: doc.id,
      ...doc.data()
    }));
    
    callback({
      statusCode: 200,
      body: JSON.stringify({ drops })
    });
    
  } catch (error) {
    callback({
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    });
  }
};
```

### Performance Functions

**`functions/api/performance/leaderboard.js`**
```javascript
const { db } = require('../../lib/firebase');

exports.main = async (event, callback) => {
  try {
    const usersSnapshot = await db.collection('users').get();
    const users = [];
    
    for (const userDoc of usersSnapshot.docs) {
      const userData = userDoc.data();
      let totalScore = 0;
      
      // Calculate score from participated drops
      for (const dropId of userData.participatedDrops || []) {
        const dropDoc = await db.collection('drops').doc(dropId).get();
        if (dropDoc.exists) {
          totalScore += dropDoc.data().points || 0;
        }
        
        // Add performance bonus
        const perfSnapshot = await db.collection('dropPerformance')
          .where('dropId', '==', dropId)
          .where('userEmail', '==', userData.email)
          .get();
        
        perfSnapshot.forEach(perfDoc => {
          const clicks = perfDoc.data().clicksGenerated || 0;
          totalScore += clicks * 10; // CLICK_MULTIPLIER
        });
      }
      
      users.push({
        ...userData,
        score: totalScore
      });
    }
    
    // Sort by score
    users.sort((a, b) => b.score - a.score);
    
    callback({
      statusCode: 200,
      body: JSON.stringify({ leaderboard: users })
    });
    
  } catch (error) {
    callback({
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    });
  }
};
```

---

## Frontend Integration

### Update Module to Use Firebase

**In `module.html`, add Firebase SDK:**
```html
<script src="https://www.gstatic.com/firebasejs/10.7.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.0/firebase-auth.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.7.0/firebase-firestore.js"></script>

<script>
  const firebaseConfig = {
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_PROJECT_ID.appspot.com",
    messagingSenderId: "YOUR_SENDER_ID",
    appId: "YOUR_APP_ID"
  };
  
  firebase.initializeApp(firebaseConfig);
  const auth = firebase.auth();
  const db = firebase.firestore();
</script>
```

### Real-time Leaderboard

**Use Firestore real-time listeners:**
```javascript
// Listen to leaderboard changes in real-time
db.collection('users')
  .orderBy('totalHype', 'desc')
  .limit(10)
  .onSnapshot((snapshot) => {
    const leaderboard = snapshot.docs.map(doc => ({
      id: doc.id,
      ...doc.data()
    }));
    renderRoster(leaderboard);
  });
```

---

## HubSpot Secrets Setup

Add Firebase credentials as secrets:

```bash
# Add Firebase service account JSON (as string)
hs secrets add FIREBASE_SERVICE_ACCOUNT

# Or add individual values
hs secrets add FIREBASE_PROJECT_ID
hs secrets add FIREBASE_PRIVATE_KEY
hs secrets add FIREBASE_CLIENT_EMAIL
```

---

## Deployment Steps

1. **Set up Firebase project**
2. **Create Firestore collections** (manually or via script)
3. **Add security rules**
4. **Create service account** and add to HubSpot secrets
5. **Create serverless functions**
6. **Update module** to use Firebase SDK
7. **Test authentication flow**
8. **Deploy functions**

---

## Cost Estimate

**Firebase Free Tier:**
- ✅ 1GB storage
- ✅ 10GB/month network
- ✅ 50K reads/day
- ✅ 20K writes/day
- ✅ 50K MAU (Monthly Active Users)

**For your use case:** Should be completely free for MVP!

---

## Timeline: 5 weeks

- **Week 1:** Firebase setup, schema design, security rules
- **Week 2-3:** Serverless functions (auth, drops, performance)
- **Week 4:** Frontend integration, real-time updates
- **Week 5:** Admin dashboard, testing, deployment

---

## Next Steps

1. ✅ Create Firebase project
2. ✅ Set up Firestore database
3. ✅ Enable Authentication
4. ✅ Create service account
5. ✅ Add secrets to HubSpot
6. ✅ Create first serverless function
7. ✅ Test Firebase connection


