# Custom URL Shortener Architecture

## Overview

Instead of long tracking URLs, users get short, branded links like:
- `amp.ly/abc123` (custom domain)
- `amplifier.com/go/abc123` (subdomain)
- `amp.ly/q3-drop-john` (custom slug option)

---

## Benefits

✅ **Better UX** - Short, shareable links
✅ **Branding** - Professional appearance
✅ **Trust** - Users trust branded links
✅ **Analytics** - All tracking in one place
✅ **Customization** - Can add custom slugs
✅ **Memorable** - Easier to remember/share

---

## Architecture Options

### Option 1: Subdomain (Recommended) ⭐

**Format:** `amp.yourdomain.com/abc123`

**Pros:**
- ✅ Easy to set up (just DNS)
- ✅ Uses your existing domain
- ✅ Professional
- ✅ No additional domain cost

**Setup:**
- Add CNAME: `amp` → Netlify/Vercel
- Or use subdomain routing

### Option 2: Custom Domain

**Format:** `amp.ly/abc123` or `amplifier.link/abc123`

**Pros:**
- ✅ Shortest possible
- ✅ Memorable
- ✅ Branded

**Cons:**
- ❌ Need to buy domain (~$10-20/year)
- ❌ Additional DNS setup

### Option 3: Path on Main Domain

**Format:** `yourdomain.com/go/abc123` or `yourdomain.com/t/abc123`

**Pros:**
- ✅ No DNS changes
- ✅ Uses existing domain
- ✅ Simple

**Cons:**
- ❌ Longer URLs
- ❌ Less branded

---

## Recommended: Subdomain (`amp.yourdomain.com`)

---

## Implementation

### Step 1: Short Code Generation

**Options:**

#### A. Random Alphanumeric (Recommended)
```javascript
// src/lib/shortener.js
export function generateShortCode(length = 6) {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  let code = '';
  for (let i = 0; i < length; i++) {
    code += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return code;
}

// Example: "aB3xY9"
```

#### B. Base62 Encoding (Shorter)
```javascript
// Encode Firestore document ID to base62
export function encodeBase62(id) {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  let num = parseInt(id.replace(/\D/g, ''), 10);
  let encoded = '';
  
  while (num > 0) {
    encoded = chars[num % 62] + encoded;
    num = Math.floor(num / 62);
  }
  
  return encoded || '0';
}
```

#### C. Custom Slugs (User-Friendly)
```javascript
// Allow users to create custom slugs
export function generateSlug(dropName, userName) {
  const dropSlug = dropName
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '')
    .substring(0, 20);
  
  const userSlug = userName
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '')
    .substring(0, 10);
  
  return `${dropSlug}-${userSlug}`;
}

// Example: "q3-platform-drop-john-doe"
```

### Step 2: Store Short Links in Firestore

**Collection: `shortLinks`**

```javascript
{
  id: "abc123", // Short code
  dropId: "drop-456",
  userEmail: "user@company.com",
  destinationUrl: "https://company.com/product",
  shortUrl: "https://amp.yourdomain.com/abc123",
  clicks: 0,
  createdAt: Timestamp,
  expiresAt: Timestamp, // Optional
  customSlug: "q3-drop-john", // Optional
  status: "active", // "active", "paused", "expired"
  metadata: {
    dropName: "Q3 Platform Drop",
    userName: "John Doe"
  }
}
```

### Step 3: Create Short Link Function

```javascript
// src/lib/shortener.js
import { collection, addDoc, query, where, getDocs } from 'firebase/firestore';
import { db } from './firebase';

export async function createShortLink(dropId, userEmail, destinationUrl, options = {}) {
  const { customSlug, expiresInDays } = options;
  
  let shortCode;
  
  // Use custom slug if provided, otherwise generate random
  if (customSlug) {
    // Check if slug is available
    const existing = await getDocs(
      query(collection(db, 'shortLinks'), where('id', '==', customSlug))
    );
    
    if (!existing.empty) {
      throw new Error('Custom slug already taken');
    }
    
    shortCode = customSlug;
  } else {
    // Generate unique random code
    let attempts = 0;
    do {
      shortCode = generateShortCode(6);
      const existing = await getDocs(
        query(collection(db, 'shortLinks'), where('id', '==', shortCode))
      );
      if (existing.empty) break;
      attempts++;
      if (attempts > 10) {
        shortCode = generateShortCode(8); // Try longer code
      }
    } while (attempts < 20);
  }
  
  // Calculate expiration
  const expiresAt = expiresInDays 
    ? new Date(Date.now() + expiresInDays * 24 * 60 * 60 * 1000)
    : null;
  
  // Create short link document
  const shortLinkRef = await addDoc(collection(db, 'shortLinks'), {
    id: shortCode,
    dropId,
    userEmail,
    destinationUrl,
    shortUrl: `https://amp.yourdomain.com/${shortCode}`,
    clicks: 0,
    createdAt: new Date(),
    expiresAt,
    status: 'active',
    metadata: {
      dropName: options.dropName,
      userName: options.userName
    }
  });
  
  return {
    shortCode,
    shortUrl: `https://amp.yourdomain.com/${shortCode}`,
    destinationUrl
  };
}
```

### Step 4: Redirect Endpoint

**Next.js API Route: `/pages/api/go/[code].js`**

```javascript
import { doc, getDoc, updateDoc, increment } from 'firebase/firestore';
import { db } from '../../lib/firebase-admin';

export default async function handler(req, res) {
  const { code } = req.query;
  
  try {
    // Get short link
    const shortLinkQuery = query(
      collection(db, 'shortLinks'),
      where('id', '==', code)
    );
    const snapshot = await getDocs(shortLinkQuery);
    
    if (snapshot.empty) {
      return res.redirect('/404');
    }
    
    const shortLink = snapshot.docs[0];
    const data = shortLink.data();
    
    // Check if expired
    if (data.expiresAt && data.expiresAt.toDate() < new Date()) {
      await updateDoc(shortLink.ref, { status: 'expired' });
      return res.redirect('/expired');
    }
    
    // Check if paused
    if (data.status !== 'active') {
      return res.redirect('/404');
    }
    
    // Record click
    await updateDoc(shortLink.ref, {
      clicks: increment(1),
      lastClickedAt: new Date()
    });
    
    // Update drop performance (same as tracking endpoint)
    const { dropId, userEmail } = data;
    
    // ... (same performance update logic as tracking endpoint)
    
    // Redirect to destination
    res.redirect(302, data.destinationUrl);
    
  } catch (error) {
    console.error('Redirect error:', error);
    res.redirect('/404');
  }
}
```

### Step 5: Custom Domain Setup

#### For Netlify:

**1. Add Custom Domain:**
- Netlify Dashboard → Domain Settings
- Add custom domain: `amp.yourdomain.com`
- Or buy new domain: `amp.ly`

**2. DNS Configuration:**
```
Type: CNAME
Name: amp
Value: your-netlify-site.netlify.app
```

**3. Netlify Redirects: `/netlify.toml`**
```toml
[[redirects]]
  from = "/*"
  to = "/.netlify/functions/redirect/:splat"
  status = 200
  force = true
```

#### For Vercel:

**1. Add Domain:**
- Vercel Dashboard → Project Settings → Domains
- Add `amp.yourdomain.com`

**2. DNS Configuration:**
```
Type: CNAME
Name: amp
Value: cname.vercel-dns.com
```

**3. Rewrite Rule: `/vercel.json`**
```json
{
  "rewrites": [
    {
      "source": "/:code",
      "destination": "/api/go/:code"
    }
  ]
}
```

### Step 6: User Interface

**Component: Short Link Generator**

```javascript
// src/components/ShortLinkGenerator.jsx
import { useState } from 'react';
import { createShortLink } from '../lib/shortener';
import { useAuth } from '../hooks/useAuth';

export function ShortLinkGenerator({ drop, destinationUrl }) {
  const { user } = useAuth();
  const [shortUrl, setShortUrl] = useState(null);
  const [loading, setLoading] = useState(false);
  const [customSlug, setCustomSlug] = useState('');
  
  const handleGenerate = async () => {
    setLoading(true);
    try {
      const result = await createShortLink(
        drop.id,
        user.email,
        destinationUrl,
        {
          customSlug: customSlug || undefined,
          dropName: drop.name,
          userName: user.name
        }
      );
      setShortUrl(result.shortUrl);
    } catch (error) {
      alert(error.message);
    } finally {
      setLoading(false);
    }
  };
  
  const handleCopy = () => {
    navigator.clipboard.writeText(shortUrl);
    alert('Link copied!');
  };
  
  return (
    <div className="short-link-generator">
      <h3>Get Your Short Link</h3>
      
      <div className="input-group">
        <input
          type="text"
          placeholder="Custom slug (optional)"
          value={customSlug}
          onChange={(e) => setCustomSlug(e.target.value)}
        />
        <button onClick={handleGenerate} disabled={loading}>
          {loading ? 'Generating...' : 'Generate Link'}
        </button>
      </div>
      
      {shortUrl && (
        <div className="short-link-result">
          <input type="text" value={shortUrl} readOnly />
          <button onClick={handleCopy}>Copy</button>
          
          <div className="qr-code">
            {/* Optional: Generate QR code */}
            <img src={`https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${shortUrl}`} />
          </div>
        </div>
      )}
    </div>
  );
}
```

---

## Advanced Features

### 1. Link Analytics Dashboard

**Show user their link stats:**

```javascript
// src/components/LinkAnalytics.jsx
import { doc, onSnapshot } from 'firebase/firestore';
import { db } from '../lib/firebase';

export function LinkAnalytics({ shortCode }) {
  const [stats, setStats] = useState(null);
  
  useEffect(() => {
    const unsubscribe = onSnapshot(
      doc(db, 'shortLinks', shortCode),
      (doc) => {
        if (doc.exists()) {
          setStats(doc.data());
        }
      }
    );
    
    return unsubscribe;
  }, [shortCode]);
  
  return (
    <div className="analytics">
      <h3>Link Analytics</h3>
      <p>Clicks: {stats?.clicks || 0}</p>
      <p>Created: {stats?.createdAt?.toDate().toLocaleDateString()}</p>
      {stats?.expiresAt && (
        <p>Expires: {stats.expiresAt.toDate().toLocaleDateString()}</p>
      )}
    </div>
  );
}
```

### 2. Link Management

**Let users manage their links:**

```javascript
// src/components/MyLinks.jsx
import { collection, query, where, onSnapshot } from 'firebase/firestore';
import { db } from '../lib/firebase';
import { useAuth } from '../hooks/useAuth';

export function MyLinks() {
  const { user } = useAuth();
  const [links, setLinks] = useState([]);
  
  useEffect(() => {
    if (!user) return;
    
    const q = query(
      collection(db, 'shortLinks'),
      where('userEmail', '==', user.email)
    );
    
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const linksData = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data()
      }));
      setLinks(linksData);
    });
    
    return unsubscribe;
  }, [user]);
  
  return (
    <div className="my-links">
      <h2>My Links</h2>
      {links.map(link => (
        <div key={link.id} className="link-item">
          <p>{link.shortUrl}</p>
          <p>Clicks: {link.clicks}</p>
          <button onClick={() => copyToClipboard(link.shortUrl)}>
            Copy
          </button>
        </div>
      ))}
    </div>
  );
}
```

### 3. Custom Slugs with Validation

```javascript
export function validateSlug(slug) {
  // Only alphanumeric and hyphens
  if (!/^[a-z0-9-]+$/.test(slug)) {
    return { valid: false, error: 'Only lowercase letters, numbers, and hyphens allowed' };
  }
  
  // Length check
  if (slug.length < 3 || slug.length > 30) {
    return { valid: false, error: 'Slug must be 3-30 characters' };
  }
  
  // Reserved words
  const reserved = ['api', 'admin', 'go', 't', 'link', 'www'];
  if (reserved.includes(slug)) {
    return { valid: false, error: 'This slug is reserved' };
  }
  
  return { valid: true };
}
```

### 4. Link Expiration

```javascript
// Auto-expire links
export async function expireOldLinks() {
  const now = new Date();
  const expiredQuery = query(
    collection(db, 'shortLinks'),
    where('expiresAt', '<=', now),
    where('status', '==', 'active')
  );
  
  const snapshot = await getDocs(expiredQuery);
  const batch = writeBatch(db);
  
  snapshot.docs.forEach(doc => {
    batch.update(doc.ref, { status: 'expired' });
  });
  
  await batch.commit();
}

// Run periodically (via cron job or scheduled function)
```

### 5. QR Code Generation

```javascript
// Generate QR code for sharing
export function generateQRCode(shortUrl) {
  return `https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${encodeURIComponent(shortUrl)}`;
}

// Or use a library
import QRCode from 'qrcode';

export async function generateQRCodeImage(shortUrl) {
  return await QRCode.toDataURL(shortUrl);
}
```

### 6. Link Preview Cards

**When link is shared, show preview:**

```javascript
// Add Open Graph meta tags to redirect page
export function LinkPreview({ shortLink }) {
  return (
    <>
      <meta property="og:title" content={shortLink.metadata.dropName} />
      <meta property="og:description" content="Join the campaign!" />
      <meta property="og:image" content={shortLink.metadata.image} />
      <meta property="og:url" content={shortLink.shortUrl} />
    </>
  );
}
```

---

## Database Schema

### Collection: `shortLinks`

```javascript
{
  id: "abc123", // Short code (document ID)
  dropId: "drop-456",
  userEmail: "user@company.com",
  destinationUrl: "https://company.com/product?utm_source=amplifier",
  shortUrl: "https://amp.yourdomain.com/abc123",
  clicks: 42,
  createdAt: Timestamp,
  expiresAt: Timestamp, // Optional
  lastClickedAt: Timestamp,
  status: "active", // "active", "paused", "expired"
  customSlug: "q3-drop-john", // Optional
  metadata: {
    dropName: "Q3 Platform Drop",
    userName: "John Doe",
    department: "Marketing"
  },
  clickHistory: [ // Optional: detailed history
    {
      timestamp: Timestamp,
      referrer: "https://linkedin.com/...",
      userAgent: "Mozilla/5.0...",
      ip: "192.168.1.1"
    }
  ]
}
```

---

## Example User Flow

1. **User participates in drop**
   - Clicks "Get My Link"
   - Optionally enters custom slug: "q3-john"
   - Gets: `amp.yourdomain.com/q3-john`

2. **User shares link**
   - Posts on LinkedIn: "Excited about our new product! Check it out: amp.yourdomain.com/q3-john"

3. **Someone clicks**
   - Redirects to product page
   - Records click
   - Updates leaderboard in real-time

4. **User sees results**
   - Dashboard shows: "42 clicks from your link"
   - Leaderboard updates automatically

---

## Cost

**Free:**
- Custom subdomain: Free (uses existing domain)
- Custom domain: ~$10-20/year (optional)
- Firebase: Free tier covers this
- Netlify/Vercel: Free tier

**Total: $0-20/year**

---

## Summary

✅ **Custom URL shortener** with branded domain
✅ **Short, memorable links** (6-8 characters)
✅ **Optional custom slugs** for user-friendly URLs
✅ **Real-time analytics** per link
✅ **Link management** dashboard
✅ **QR code generation** for easy sharing
✅ **Expiration dates** for time-limited campaigns
✅ **Full integration** with tracking system

This gives you a professional, branded URL shortener that integrates seamlessly with your tracking and analytics system!


