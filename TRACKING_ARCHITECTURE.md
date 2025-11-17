# Click Tracking Architecture

## Overview

When users share posts with their unique links, we need to:
1. Track clicks on those links
2. Attribute clicks back to the user who shared
3. Update performance metrics in real-time
4. Update leaderboards automatically

---

## Tracking Flow

```
User shares post on LinkedIn
    ↓
Someone clicks unique link
    ↓
Tracking endpoint records click
    ↓
Updates Firestore (dropPerformance collection)
    ↓
Firebase real-time listener updates UI
    ↓
Leaderboard updates automatically
```

---

## Solution Architecture

### Option 1: Server-Side Redirect (Recommended) ⭐

**How it works:**
1. User gets unique link: `https://yourdomain.com/track/{dropId}/{userId}`
2. Tracking endpoint records click
3. Redirects to actual destination URL

**Pros:**
- ✅ Accurate tracking (server-side)
- ✅ Works even if JavaScript disabled
- ✅ Can't be blocked by ad blockers
- ✅ Simple implementation

**Cons:**
- ❌ Requires backend endpoint
- ❌ Slight redirect delay

### Option 2: Client-Side Tracking Pixel

**How it works:**
1. User gets link: `https://actual-url.com?tracking={token}`
2. Landing page includes tracking pixel
3. Pixel fires and records click

**Pros:**
- ✅ No redirect needed
- ✅ User goes directly to destination

**Cons:**
- ❌ Can be blocked by ad blockers
- ❌ Requires JavaScript
- ❌ Less reliable

### Option 3: Hybrid Approach (Best) ⭐⭐

**How it works:**
1. Server-side redirect for tracking
2. Client-side pixel for additional analytics
3. Both update Firestore

**Pros:**
- ✅ Most accurate
- ✅ Redundant tracking
- ✅ Best of both worlds

---

## Recommended: Server-Side Redirect

### Link Structure

**Format:**
```
https://amplifier.yourdomain.com/track/{dropId}/{userEmail}/{encodedDestination}
```

**Example:**
```
https://amplifier.yourdomain.com/track/drop-123/john@company.com/https%3A%2F%2Fwww.company.com%2Fnew-product
```

**Or shorter with token:**
```
https://amplifier.yourdomain.com/track/{trackingToken}
```

Where `trackingToken` is a short, unique identifier stored in Firestore.

---

## Implementation

### Step 1: Generate Tracking Links

**When user participates in a drop:**

```javascript
// src/lib/tracking.js
import { collection, addDoc, doc, getDoc } from 'firebase/firestore';
import { db } from './firebase';

export async function generateTrackingLink(dropId, userEmail, destinationUrl) {
  // Create tracking record
  const trackingRef = await addDoc(collection(db, 'trackingLinks'), {
    dropId,
    userEmail,
    destinationUrl,
    clicks: 0,
    createdAt: new Date(),
    status: 'active'
  });

  // Generate short token (or use document ID)
  const token = trackingRef.id;
  
  // Build tracking URL
  const trackingUrl = `${window.location.origin}/track/${token}`;
  
  return {
    trackingUrl,
    token,
    destinationUrl
  };
}
```

### Step 2: Create Tracking Endpoint

**Next.js API Route: `/pages/api/track/[token].js`**

```javascript
import { doc, getDoc, updateDoc, increment, collection, query, where, getDocs } from 'firebase/firestore';
import { db } from '../../lib/firebase-admin'; // Server-side Firebase Admin SDK

export default async function handler(req, res) {
  const { token } = req.query;
  
  try {
    // Get tracking link record
    const trackingDoc = await getDoc(doc(db, 'trackingLinks', token));
    
    if (!trackingDoc.exists()) {
      return res.redirect('/404');
    }
    
    const trackingData = trackingDoc.data();
    const { dropId, userEmail, destinationUrl } = trackingData;
    
    // Record the click
    await updateDoc(doc(db, 'trackingLinks', token), {
      clicks: increment(1),
      lastClickedAt: new Date()
    });
    
    // Update drop performance
    const performanceQuery = query(
      collection(db, 'dropPerformance'),
      where('dropId', '==', dropId),
      where('userEmail', '==', userEmail)
    );
    
    const performanceSnapshot = await getDocs(performanceQuery);
    
    if (performanceSnapshot.empty) {
      // Create new performance record
      await addDoc(collection(db, 'dropPerformance'), {
        dropId,
        userEmail,
        clicksGenerated: 1,
        participationDate: new Date(),
        uniqueLink: trackingUrl
      });
    } else {
      // Update existing record
      const perfDoc = performanceSnapshot.docs[0];
      await updateDoc(perfDoc.ref, {
        clicksGenerated: increment(1)
      });
    }
    
    // Update user's total hype
    const userRef = doc(db, 'users', userEmail);
    const userDoc = await getDoc(userRef);
    
    if (userDoc.exists()) {
      const currentHype = userDoc.data().totalHype || 0;
      const clickMultiplier = 10; // From drop settings
      await updateDoc(userRef, {
        totalHype: currentHype + clickMultiplier
      });
    }
    
    // Update drop analytics (aggregated)
    const analyticsRef = doc(db, 'dropAnalytics', dropId);
    const analyticsDoc = await getDoc(analyticsRef);
    
    if (analyticsDoc.exists()) {
      await updateDoc(analyticsRef, {
        totalClicks: increment(1),
        updatedAt: new Date()
      });
    }
    
    // Redirect to destination
    res.redirect(302, destinationUrl);
    
  } catch (error) {
    console.error('Tracking error:', error);
    // Still redirect even if tracking fails
    res.redirect(302, destinationUrl || '/');
  }
}
```

### Step 3: Alternative - Netlify/Vercel Serverless Function

**If using Netlify: `/netlify/functions/track.js`**

```javascript
const { initializeApp, cert } = require('firebase-admin/app');
const { getFirestore, FieldValue } = require('firebase-admin/firestore');

// Initialize Firebase Admin (one-time)
if (!global.firebaseAdmin) {
  const serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT);
  
  initializeApp({
    credential: cert(serviceAccount)
  });
  
  global.firebaseAdmin = true;
}

const db = getFirestore();

exports.handler = async (event, context) => {
  const { token } = event.pathParameters || {};
  
  try {
    const trackingRef = db.collection('trackingLinks').doc(token);
    const trackingDoc = await trackingRef.get();
    
    if (!trackingDoc.exists) {
      return {
        statusCode: 404,
        headers: { Location: '/' },
        body: ''
      };
    }
    
    const data = trackingDoc.data();
    const { dropId, userEmail, destinationUrl } = data;
    
    // Record click (batch update for performance)
    const batch = db.batch();
    
    // Update tracking link
    batch.update(trackingRef, {
      clicks: FieldValue.increment(1),
      lastClickedAt: new Date()
    });
    
    // Update or create performance record
    const perfQuery = db.collection('dropPerformance')
      .where('dropId', '==', dropId)
      .where('userEmail', '==', userEmail)
      .limit(1);
    
    const perfSnapshot = await perfQuery.get();
    
    if (perfSnapshot.empty) {
      const newPerfRef = db.collection('dropPerformance').doc();
      batch.set(newPerfRef, {
        dropId,
        userEmail,
        clicksGenerated: 1,
        participationDate: new Date()
      });
    } else {
      batch.update(perfSnapshot.docs[0].ref, {
        clicksGenerated: FieldValue.increment(1)
      });
    }
    
    // Update user total hype
    const userRef = db.collection('users').doc(userEmail);
    const userDoc = await userRef.get();
    
    if (userDoc.exists) {
      const clickMultiplier = 10;
      batch.update(userRef, {
        totalHype: FieldValue.increment(clickMultiplier)
      });
    }
    
    // Update drop analytics
    const analyticsRef = db.collection('dropAnalytics').doc(dropId);
    const analyticsDoc = await analyticsRef.get();
    
    if (analyticsDoc.exists) {
      batch.update(analyticsRef, {
        totalClicks: FieldValue.increment(1),
        updatedAt: new Date()
      });
    } else {
      batch.set(analyticsRef, {
        dropId,
        totalClicks: 1,
        totalParticipants: 0,
        createdAt: new Date(),
        updatedAt: new Date()
      });
    }
    
    // Execute batch
    await batch.commit();
    
    // Redirect
    return {
      statusCode: 302,
      headers: {
        Location: destinationUrl
      },
      body: ''
    };
    
  } catch (error) {
    console.error('Tracking error:', error);
    return {
      statusCode: 302,
      headers: {
        Location: destinationUrl || '/'
      },
      body: ''
    };
  }
};
```

### Step 4: Real-Time UI Updates

**React Hook for Real-Time Performance:**

```javascript
// src/hooks/usePerformance.js
import { useState, useEffect } from 'react';
import { doc, onSnapshot } from 'firebase/firestore';
import { db } from '../lib/firebase';

export function usePerformance(dropId, userEmail) {
  const [performance, setPerformance] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!dropId || !userEmail) return;

    // Query performance record
    const q = query(
      collection(db, 'dropPerformance'),
      where('dropId', '==', dropId),
      where('userEmail', '==', userEmail)
    );

    const unsubscribe = onSnapshot(q, (snapshot) => {
      if (!snapshot.empty) {
        const perfData = snapshot.docs[0].data();
        setPerformance({
          id: snapshot.docs[0].id,
          ...perfData
        });
      } else {
        setPerformance(null);
      }
      setLoading(false);
    });

    return unsubscribe;
  }, [dropId, userEmail]);

  return { performance, loading };
}
```

**Component Usage:**

```javascript
// src/components/DropCard.jsx
import { usePerformance } from '../hooks/usePerformance';
import { useAuth } from '../hooks/useAuth';

export function DropCard({ drop }) {
  const { user } = useAuth();
  const { performance } = usePerformance(drop.id, user?.email);

  return (
    <div>
      <h3>{drop.name}</h3>
      
      {performance && (
        <div className="performance-stats">
          <p>Clicks: {performance.clicksGenerated}</p>
          <p>Hype Points: {performance.clicksGenerated * 10}</p>
        </div>
      )}
      
      <button onClick={() => generateLink(drop.id)}>
        Get My Tracking Link
      </button>
    </div>
  );
}
```

---

## Advanced Features

### 1. Click Attribution with Metadata

Track additional data:
- Referrer (where the click came from)
- User agent (device/browser)
- Timestamp
- Geographic location (optional)

```javascript
// Enhanced tracking record
await updateDoc(trackingRef, {
  clicks: increment(1),
  clickHistory: arrayUnion({
    timestamp: new Date(),
    referrer: req.headers.referer,
    userAgent: req.headers['user-agent'],
    ip: req.headers['x-forwarded-for']
  })
});
```

### 2. Fraud Prevention

- Rate limiting (max clicks per hour)
- IP tracking
- Bot detection
- Click validation

```javascript
// Check for suspicious activity
const recentClicks = await getDocs(
  query(
    collection(db, 'trackingLinks'),
    where('token', '==', token),
    where('lastClickedAt', '>', new Date(Date.now() - 60000)) // Last minute
  )
);

if (recentClicks.size > 10) {
  // Possible bot, flag for review
  await updateDoc(trackingRef, {
    flagged: true,
    flagReason: 'High click rate'
  });
}
```

### 3. Analytics Dashboard

Track:
- Click-through rate per user
- Best performing drops
- Time of day analysis
- Device breakdown

### 4. UTM Parameter Preservation

Preserve original UTM parameters:

```javascript
// When generating link
const utmParams = new URLSearchParams({
  utm_campaign: drop.name,
  utm_source: 'amplifier',
  utm_content: userEmail,
  utm_medium: 'social'
});

const trackingUrl = `${origin}/track/${token}?${utmParams.toString()}`;

// When redirecting, preserve UTM params
const destination = new URL(destinationUrl);
utmParams.forEach((value, key) => {
  destination.searchParams.set(key, value);
});

res.redirect(302, destination.toString());
```

---

## Database Schema Updates

### Collection: `trackingLinks`

```javascript
{
  id: "token-123",
  dropId: "drop-456",
  userEmail: "user@company.com",
  destinationUrl: "https://company.com/product",
  clicks: 42,
  createdAt: Timestamp,
  lastClickedAt: Timestamp,
  status: "active", // "active", "paused", "expired"
  clickHistory: [
    {
      timestamp: Timestamp,
      referrer: "https://linkedin.com/...",
      userAgent: "Mozilla/5.0...",
      ip: "192.168.1.1"
    }
  ]
}
```

### Collection: `dropPerformance` (Updated)

```javascript
{
  dropId: "drop-456",
  userEmail: "user@company.com",
  clicksGenerated: 42,
  participationDate: Timestamp,
  uniqueLink: "https://amplifier.com/track/token-123",
  lastClickAt: Timestamp,
  clickRate: 0.05 // clicks / impressions (if tracking impressions)
}
```

---

## Testing Strategy

### 1. Manual Testing
- Generate tracking link
- Click link
- Verify Firestore updates
- Check UI updates in real-time

### 2. Automated Testing
- Unit tests for link generation
- Integration tests for tracking endpoint
- E2E tests for full flow

### 3. Load Testing
- Test with high click volume
- Verify batch updates work
- Check Firebase rate limits

---

## Monitoring & Alerts

### Track:
- Click volume per drop
- Unusual patterns (potential fraud)
- Failed redirects
- Firebase errors

### Alerts:
- High click rate (possible bot)
- Failed tracking (database errors)
- Unusual patterns

---

## Summary

**Recommended Approach:**
1. ✅ Server-side redirect endpoint (`/track/{token}`)
2. ✅ Batch updates to Firestore for performance
3. ✅ Real-time UI updates via Firestore listeners
4. ✅ Short, shareable tracking tokens
5. ✅ Preserve UTM parameters
6. ✅ Fraud prevention measures

**Benefits:**
- Accurate tracking (server-side)
- Real-time updates
- Scalable (Firebase handles load)
- Simple for users (just share the link)
- Can't be blocked by ad blockers

This architecture ensures every click is tracked accurately and updates appear in real-time across all users viewing the platform.


