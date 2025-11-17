# Gamification Plan - The Amplifier Platform

## Philosophy

**Core Principle:** Reward effort and quality as much as reach.

We want to ensure that:
- ✅ Someone with 100 followers can compete with someone with 10,000
- ✅ Customizing copy is rewarded
- ✅ Creating original content is valued
- ✅ Quality engagement matters more than just clicks
- ✅ Team collaboration is encouraged

---

## Point System Overview

### Base Points (Participation)
- **Participating in a drop:** 100 points
- **Using provided copy:** 0 bonus
- **Customizing copy:** +50 points
- **Creating original copy:** +100 points
- **Adding personal story/anecdote:** +75 points
- **Including call-to-action:** +25 points

### Performance Points (Reach)
- **Per click generated:** 10 points
- **First 10 clicks:** 15 points each (effort bonus)
- **Clicks 11-50:** 10 points each
- **Clicks 51+:** 8 points each (diminishing returns)

### Quality Multipliers
- **Engagement rate >5%:** 1.5x multiplier
- **Engagement rate 3-5%:** 1.2x multiplier
- **Engagement rate <3%:** 1.0x multiplier

### Bonus Points
- **First to participate:** +50 points
- **Participating in all active drops:** +100 points
- **Consistent participation (3+ drops):** +50 points
- **Team collaboration (tagging colleagues):** +25 points per tag
- **Cross-platform sharing (LinkedIn + Twitter):** +50 points
- **Sharing multiple times (spread over time):** +25 points per additional share

---

## Detailed Point Breakdown

### 1. Participation Points

#### Base Participation
```javascript
const PARTICIPATION_POINTS = {
  PARTICIPATE: 100,           // Just clicking "I'm in"
  USE_PROVIDED_COPY: 0,       // Using default copy (no bonus)
  CUSTOMIZE_COPY: 50,         // Editing provided copy
  ORIGINAL_COPY: 100,         // Writing completely original
  ADD_STORY: 75,              // Personal anecdote/story
  ADD_CTA: 25,                // Clear call-to-action
  FIRST_PARTICIPANT: 50,      // First person in drop
  ALL_DROPS: 100,             // Participated in all active drops
  CONSISTENT: 50              // 3+ drops in a row
};
```

#### Copy Quality Scoring
```javascript
const COPY_QUALITY = {
  PROVIDED: 0,               // Used as-is
  MINOR_EDIT: 25,            // Changed a few words
  CUSTOMIZED: 50,            // Significant edits, kept structure
  ORIGINAL: 100,             // Completely original
  ORIGINAL_WITH_STORY: 175,  // Original + personal story
  ORIGINAL_WITH_CTA: 125     // Original + clear CTA
};
```

### 2. Performance Points (Reach)

#### Click-Based Points (Tiered System)
```javascript
const CLICK_POINTS = {
  TIER_1: { range: [1, 10], points: 15 },    // First 10 clicks
  TIER_2: { range: [11, 50], points: 10 },  // Next 40 clicks
  TIER_3: { range: [51, Infinity], points: 8 } // Diminishing returns
};

function calculateClickPoints(clicks) {
  let total = 0;
  
  // Tier 1: First 10 clicks
  const tier1Clicks = Math.min(clicks, 10);
  total += tier1Clicks * 15;
  
  if (clicks > 10) {
    // Tier 2: Clicks 11-50
    const tier2Clicks = Math.min(clicks - 10, 40);
    total += tier2Clicks * 10;
    
    if (clicks > 50) {
      // Tier 3: Clicks 51+
      const tier3Clicks = clicks - 50;
      total += tier3Clicks * 8;
    }
  }
  
  return total;
}

// Examples:
// 5 clicks = 5 * 15 = 75 points
// 25 clicks = (10 * 15) + (15 * 10) = 300 points
// 100 clicks = (10 * 15) + (40 * 10) + (50 * 8) = 950 points
```

**Why tiered?**
- Rewards early engagement (effort bonus)
- Prevents large accounts from dominating
- Makes it competitive for everyone

### 3. Engagement Quality Multipliers

```javascript
const ENGAGEMENT_MULTIPLIERS = {
  HIGH: { rate: 0.05, multiplier: 1.5 },    // >5% engagement
  MEDIUM: { rate: 0.03, multiplier: 1.2 }, // 3-5% engagement
  LOW: { rate: 0, multiplier: 1.0 }         // <3% engagement
};

function calculateEngagementRate(clicks, followers) {
  if (!followers || followers === 0) return 0;
  return clicks / followers;
}

function applyEngagementMultiplier(basePoints, engagementRate) {
  if (engagementRate >= 0.05) return basePoints * 1.5;
  if (engagementRate >= 0.03) return basePoints * 1.2;
  return basePoints;
}
```

**Note:** Engagement rate requires follower count (optional field)

### 4. Bonus Points

```javascript
const BONUS_POINTS = {
  FIRST_PARTICIPANT: 50,
  ALL_ACTIVE_DROPS: 100,
  CONSISTENT_PARTICIPATION: 50,  // 3+ consecutive drops
  TEAM_COLLABORATION: 25,        // Per colleague tagged
  CROSS_PLATFORM: 50,            // LinkedIn + Twitter
  MULTIPLE_SHARES: 25,           // Per additional share (max 3)
  EARLY_BIRD: 30,                // Participated within 24h of drop launch
  WEEKEND_WARRIOR: 40            // Shared on weekend (lower traffic)
};
```

---

## Complete Point Calculation

### Example Scenarios

#### Scenario 1: High Effort, Low Reach
**User:** Sarah (500 followers)
- ✅ Participated: 100 points
- ✅ Created original copy: +100 points
- ✅ Added personal story: +75 points
- ✅ Added CTA: +25 points
- ✅ First participant: +50 points
- ✅ Generated 15 clicks: (10 * 15) + (5 * 10) = 200 points
- ✅ Engagement rate: 15/500 = 3% → 1.2x multiplier

**Total:** (100 + 100 + 75 + 25 + 50 + 200) * 1.2 = **660 points**

#### Scenario 2: Low Effort, High Reach
**User:** John (10,000 followers)
- ✅ Participated: 100 points
- ❌ Used provided copy: 0 bonus
- ✅ Generated 200 clicks: (10 * 15) + (40 * 10) + (150 * 8) = 1,550 points
- ❌ Engagement rate: 200/10000 = 2% → 1.0x multiplier

**Total:** (100 + 1,550) * 1.0 = **1,650 points**

#### Scenario 3: Balanced (Ideal)
**User:** Mike (2,000 followers)
- ✅ Participated: 100 points
- ✅ Customized copy: +50 points
- ✅ Added CTA: +25 points
- ✅ Generated 80 clicks: (10 * 15) + (40 * 10) + (30 * 8) = 790 points
- ✅ Engagement rate: 80/2000 = 4% → 1.2x multiplier
- ✅ Cross-platform share: +50 points

**Total:** (100 + 50 + 25 + 790 + 50) * 1.2 = **1,218 points**

---

## Badges & Achievements

### Participation Badges
- 🎯 **First Responder** - First to participate in a drop
- 🔥 **On Fire** - Participated in 5+ drops
- 💪 **Consistent** - Participated in 3 consecutive drops
- 🌟 **All Star** - Participated in all active drops
- 🎨 **Creative** - Created original copy 5+ times
- 📝 **Storyteller** - Added personal stories 3+ times

### Performance Badges
- 🚀 **Rocket Start** - Generated 10+ clicks in first 24h
- 📈 **Growth Hacker** - 5%+ engagement rate
- 🎯 **Precision** - 10%+ engagement rate
- 👑 **King/Queen of Clicks** - Generated 100+ clicks in a drop
- 💎 **Diamond Engagement** - 15%+ engagement rate

### Quality Badges
- ✍️ **Wordsmith** - Customized copy 10+ times
- 🎭 **Personal Touch** - Added stories 5+ times
- 🔗 **Connector** - Tagged colleagues 10+ times
- 🌐 **Multi-Platform** - Shared on 2+ platforms 5+ times
- ⏰ **Timely** - Shared within 1 hour of drop launch 3+ times

### Team Badges
- 🤝 **Team Player** - Tagged 5+ colleagues
- 🏆 **Team Champion** - Your team won a drop
- 🎪 **Party Starter** - Started a conversation thread
- 💬 **Engager** - Responded to 10+ comments

---

## Leaderboards

### 1. Overall Leaderboard
- Total Hype points (all-time)
- Updated in real-time
- Top 10 displayed

### 2. Drop Leaderboard
- Points for current/active drop
- Resets each drop
- Shows top performers

### 3. Weekly Leaderboard
- Points earned this week
- Resets every Monday
- Weekly prizes/recognition

### 4. Quality Leaderboard
- Based on effort points (not clicks)
- Rewards customization, originality
- Separate from reach-based

### 5. Team Leaderboard
- By department/team
- Aggregated team points
- Team competitions

### 6. Engagement Leaderboard
- Based on engagement rate (not raw clicks)
- Levels the playing field
- Rewards quality over quantity

---

## Implementation

### Database Schema

**Collection: `userPoints`**
```javascript
{
  userEmail: "user@company.com",
  totalHype: 5000,
  participationPoints: 1200,
  performancePoints: 3500,
  bonusPoints: 300,
  currentDropPoints: 450,
  weeklyPoints: 800,
  badges: ["first-responder", "creative", "growth-hacker"],
  stats: {
    totalDrops: 12,
    totalClicks: 350,
    avgEngagementRate: 0.045,
    originalCopies: 8,
    storiesAdded: 5,
    crossPlatformShares: 3
  }
}
```

**Collection: `dropParticipation`**
```javascript
{
  id: "participation-123",
  dropId: "drop-456",
  userEmail: "user@company.com",
  participationDate: Timestamp,
  points: {
    base: 100,
    copyQuality: 100,        // Original copy
    story: 75,               // Added story
    cta: 25,                 // Added CTA
    firstParticipant: 50,
    earlyBird: 30,
    clicks: 200,            // From click performance
    engagementMultiplier: 1.2,
    bonuses: 50,            // Cross-platform, etc.
    total: 660
  },
  copyType: "original",     // "provided", "customized", "original"
  hasStory: true,
  hasCTA: true,
  clicksGenerated: 15,
  engagementRate: 0.03,
  sharedOn: ["linkedin", "twitter"],
  colleaguesTagged: 2
}
```

### Point Calculation Function

```javascript
// src/lib/gamification.js
import { collection, addDoc, doc, updateDoc, increment, getDoc } from 'firebase/firestore';
import { db } from './firebase';

export async function calculateParticipationPoints(participationData) {
  let points = {
    base: 100,
    copyQuality: 0,
    story: 0,
    cta: 0,
    bonuses: 0,
    total: 0
  };
  
  // Copy quality points
  switch (participationData.copyType) {
    case 'provided':
      points.copyQuality = 0;
      break;
    case 'customized':
      points.copyQuality = 50;
      break;
    case 'original':
      points.copyQuality = 100;
      break;
  }
  
  // Story bonus
  if (participationData.hasStory) {
    points.story = 75;
  }
  
  // CTA bonus
  if (participationData.hasCTA) {
    points.cta = 25;
  }
  
  // First participant bonus
  if (participationData.isFirstParticipant) {
    points.bonuses += 50;
  }
  
  // Early bird bonus (within 24h)
  if (participationData.isEarlyBird) {
    points.bonuses += 30;
  }
  
  // Cross-platform bonus
  if (participationData.sharedOn.length > 1) {
    points.bonuses += 50;
  }
  
  // Team collaboration bonus
  points.bonuses += participationData.colleaguesTagged * 25;
  
  // Multiple shares bonus (max 3)
  const additionalShares = Math.min(participationData.shareCount - 1, 2);
  points.bonuses += additionalShares * 25;
  
  points.total = points.base + points.copyQuality + points.story + 
                 points.cta + points.bonuses;
  
  return points;
}

export async function calculateClickPoints(clicks) {
  let total = 0;
  
  // Tier 1: First 10 clicks
  const tier1Clicks = Math.min(clicks, 10);
  total += tier1Clicks * 15;
  
  if (clicks > 10) {
    // Tier 2: Clicks 11-50
    const tier2Clicks = Math.min(clicks - 10, 40);
    total += tier2Clicks * 10;
    
    if (clicks > 50) {
      // Tier 3: Clicks 51+
      const tier3Clicks = clicks - 50;
      total += tier3Clicks * 8;
    }
  }
  
  return total;
}

export async function applyEngagementMultiplier(basePoints, clicks, followers) {
  if (!followers || followers === 0) return basePoints;
  
  const engagementRate = clicks / followers;
  
  if (engagementRate >= 0.05) return basePoints * 1.5;
  if (engagementRate >= 0.03) return basePoints * 1.2;
  return basePoints;
}

export async function recordParticipation(dropId, userEmail, participationData) {
  // Calculate points
  const participationPoints = await calculateParticipationPoints(participationData);
  
  // Create participation record
  const participationRef = await addDoc(collection(db, 'dropParticipation'), {
    dropId,
    userEmail,
    participationDate: new Date(),
    points: participationPoints,
    ...participationData
  });
  
  // Update user's total points
  const userPointsRef = doc(db, 'userPoints', userEmail);
  const userPointsDoc = await getDoc(userPointsRef);
  
  if (userPointsDoc.exists()) {
    await updateDoc(userPointsRef, {
      totalHype: increment(participationPoints.total),
      participationPoints: increment(participationPoints.total),
      currentDropPoints: increment(participationPoints.total)
    });
  } else {
    await setDoc(userPointsRef, {
      userEmail,
      totalHype: participationPoints.total,
      participationPoints: participationPoints.total,
      performancePoints: 0,
      bonusPoints: 0,
      currentDropPoints: participationPoints.total,
      weeklyPoints: participationPoints.total,
      badges: [],
      stats: {
        totalDrops: 1,
        totalClicks: 0,
        avgEngagementRate: 0,
        originalCopies: participationData.copyType === 'original' ? 1 : 0,
        storiesAdded: participationData.hasStory ? 1 : 0,
        crossPlatformShares: participationData.sharedOn.length > 1 ? 1 : 0
      }
    });
  }
  
  return participationRef.id;
}

export async function updateClickPoints(dropId, userEmail, newClickCount) {
  // Get participation record
  const participationQuery = query(
    collection(db, 'dropParticipation'),
    where('dropId', '==', dropId),
    where('userEmail', '==', userEmail)
  );
  
  const snapshot = await getDocs(participationQuery);
  if (snapshot.empty) return;
  
  const participation = snapshot.docs[0];
  const oldClicks = participation.data().clicksGenerated || 0;
  const newClicks = newClickCount;
  
  // Calculate new click points
  const oldClickPoints = await calculateClickPoints(oldClicks);
  const newClickPoints = await calculateClickPoints(newClicks);
  const clickPointsDelta = newClickPoints - oldClickPoints;
  
  // Get user data for engagement calculation
  const userDoc = await getDoc(doc(db, 'users', userEmail));
  const followers = userDoc.data()?.followers || 0;
  
  // Apply engagement multiplier
  const engagementMultiplier = await applyEngagementMultiplier(
    1.0, newClicks, followers
  );
  
  const adjustedClickPoints = clickPointsDelta * engagementMultiplier;
  
  // Update participation record
  await updateDoc(participation.ref, {
    clicksGenerated: newClicks,
    clickPoints: newClickPoints,
    engagementRate: followers > 0 ? newClicks / followers : 0
  });
  
  // Update user points
  const userPointsRef = doc(db, 'userPoints', userEmail);
  await updateDoc(userPointsRef, {
    totalHype: increment(adjustedClickPoints),
    performancePoints: increment(adjustedClickPoints),
    currentDropPoints: increment(adjustedClickPoints)
  });
}
```

---

## UI Components

### Participation Form

```javascript
// src/components/ParticipationForm.jsx
export function ParticipationForm({ drop, onParticipate }) {
  const [copyType, setCopyType] = useState('provided');
  const [hasStory, setHasStory] = useState(false);
  const [hasCTA, setHasCTA] = useState(false);
  const [sharedOn, setSharedOn] = useState(['linkedin']);
  const [colleaguesTagged, setColleaguesTagged] = useState(0);
  
  const estimatedPoints = calculateEstimatedPoints({
    copyType,
    hasStory,
    hasCTA,
    sharedOn,
    colleaguesTagged
  });
  
  return (
    <div className="participation-form">
      <h3>Join the Drop</h3>
      
      <div className="copy-options">
        <label>
          <input
            type="radio"
            value="provided"
            checked={copyType === 'provided'}
            onChange={(e) => setCopyType(e.target.value)}
          />
          Use provided copy (0 bonus points)
        </label>
        
        <label>
          <input
            type="radio"
            value="customized"
            checked={copyType === 'customized'}
            onChange={(e) => setCopyType(e.target.value)}
          />
          Customize copy (+50 points)
        </label>
        
        <label>
          <input
            type="radio"
            value="original"
            checked={copyType === 'original'}
            onChange={(e) => setCopyType(e.target.value)}
          />
          Write original copy (+100 points)
        </label>
      </div>
      
      <div className="bonus-options">
        <label>
          <input
            type="checkbox"
            checked={hasStory}
            onChange={(e) => setHasStory(e.target.checked)}
          />
          Add personal story (+75 points)
        </label>
        
        <label>
          <input
            type="checkbox"
            checked={hasCTA}
            onChange={(e) => setHasCTA(e.target.checked)}
          />
          Include call-to-action (+25 points)
        </label>
      </div>
      
      <div className="platform-selection">
        <label>Share on:</label>
        <label>
          <input
            type="checkbox"
            checked={sharedOn.includes('linkedin')}
            onChange={(e) => togglePlatform('linkedin')}
          />
          LinkedIn
        </label>
        <label>
          <input
            type="checkbox"
            checked={sharedOn.includes('twitter')}
            onChange={(e) => togglePlatform('twitter')}
          />
          Twitter (+50 bonus if both)
        </label>
      </div>
      
      <div className="points-preview">
        <h4>Estimated Points: {estimatedPoints}</h4>
        <p>Base: 100</p>
        <p>Copy: +{getCopyPoints(copyType)}</p>
        {hasStory && <p>Story: +75</p>}
        {hasCTA && <p>CTA: +25</p>}
        {sharedOn.length > 1 && <p>Cross-platform: +50</p>}
        <p className="note">+ Click performance points (10-15 per click)</p>
      </div>
      
      <button onClick={() => handleParticipate()}>
        Join Drop
      </button>
    </div>
  );
}
```

### Points Display

```javascript
// src/components/PointsDisplay.jsx
export function PointsDisplay({ userEmail }) {
  const { points } = useUserPoints(userEmail);
  
  return (
    <div className="points-display">
      <div className="total-hype">
        <h2>{points.totalHype.toLocaleString()}</h2>
        <p>Total Hype Points</p>
      </div>
      
      <div className="breakdown">
        <div>
          <span>Participation:</span>
          <span>{points.participationPoints}</span>
        </div>
        <div>
          <span>Performance:</span>
          <span>{points.performancePoints}</span>
        </div>
        <div>
          <span>Bonuses:</span>
          <span>{points.bonusPoints}</span>
        </div>
      </div>
      
      <div className="badges">
        {points.badges.map(badge => (
          <Badge key={badge} name={badge} />
        ))}
      </div>
    </div>
  );
}
```

---

## Anti-Gaming Measures

### 1. Click Validation
- Rate limiting (max clicks per hour)
- IP tracking
- Bot detection
- Suspicious pattern detection

### 2. Engagement Validation
- Verify follower count (optional, self-reported)
- Cross-reference with platform APIs if available
- Flag unrealistic engagement rates

### 3. Participation Validation
- Verify actual sharing (optional: link to post)
- Prevent duplicate participation
- Time-based restrictions

---

## Summary

✅ **Effort is rewarded** - Customizing copy, adding stories, etc.
✅ **Reach is rewarded** - But with diminishing returns
✅ **Quality matters** - Engagement multipliers favor quality
✅ **Fair competition** - Tiered system helps smaller accounts
✅ **Team collaboration** - Bonuses for tagging colleagues
✅ **Multiple ways to win** - Quality leaderboard separate from reach
✅ **Badges & achievements** - Recognition for various accomplishments
✅ **Real-time updates** - Points update as clicks come in

This system ensures that someone who puts in effort can compete with someone who has a large following!


