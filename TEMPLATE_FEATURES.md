# Amplifier Platform Template - Features Overview

## What's New in the Template

The standalone template (`templates/Amplifier Platform.html`) includes all the planned features:

### ✅ Gamification System

1. **Participation Form with Point Options:**
   - Use provided copy (0 bonus)
   - Customize copy (+50 points)
   - Write original copy (+100 points)
   - Add personal story (+75 points)
   - Add call-to-action (+25 points)

2. **Tiered Click Points:**
   - First 10 clicks: 15 points each
   - Clicks 11-50: 10 points each
   - Clicks 51+: 8 points each

3. **Real-time Points Preview:**
   - Shows estimated points as user selects options
   - Updates dynamically

### ✅ URL Shortener Integration

- Short link generation: `amp.yourdomain.com/abc123`
- Displayed prominently after participation
- Copy button for easy sharing
- Integrated with tracking system

### ✅ Enhanced User Stats

- Total Hype points display
- Current drop breakdown
- Participation points vs. click points
- Badges display

### ✅ Improved Leaderboard

- Shows badges next to names
- Real-time updates
- Highlights current user
- Top 3 get special styling

### ✅ Email-Based Login

- Changed from name to email input
- Domain extraction for future verification
- Better user identification

## File Structure

```
social-thunderclap/
├── templates/
│   └── Amplifier Platform.html    # Standalone template (NEW)
├── modules/
│   └── Amplifier Platform.module/
│       └── module.html             # HubSpot module (existing)
└── ...
```

## How to Use

### Option 1: Standalone Template (Recommended for Preview)

1. Open `templates/Amplifier Platform.html` in a browser
2. No server needed - works as static HTML
3. Perfect for previewing and testing

### Option 2: HubSpot Module

1. Upload module to HubSpot (already done)
2. Add to any page template
3. Configure via module fields

## Key Features in Template

### 1. Gamification Participation Form

When user clicks "Join Drop", they see:
- Copy style options (with point values)
- Story checkbox (+75 points)
- CTA checkbox (+25 points)
- Real-time points preview
- Estimated total points

### 2. Short Link Display

After participation:
- Short link: `amp.yourdomain.com/abc123`
- Copy button
- Instructions for sharing

### 3. Points Breakdown

User stats show:
- Total Hype points
- Current drop participation points
- Click-generated points
- Combined total

### 4. Badges System

- Badges displayed next to user names
- Shows achievements
- Visual recognition

## Next Steps for Full Implementation

1. **Connect to Firebase:**
   - Replace mock data with Firestore
   - Add real authentication
   - Implement real-time updates

2. **Add Tracking Endpoint:**
   - Create `/api/track/[code]` endpoint
   - Record clicks
   - Update points in real-time

3. **Implement URL Shortener:**
   - Generate actual short codes
   - Store in Firestore
   - Create redirect endpoint

4. **Add Admin Dashboard:**
   - Create drops interface
   - Manage users
   - View analytics

## Testing the Template

1. Open `templates/Amplifier Platform.html` in browser
2. Enter email (any email works for demo)
3. Click "Join Drop"
4. Select participation options
5. See points preview update
6. Click "Join Drop & Get My Link"
7. See short link appear
8. Check leaderboard updates

All features are functional with mock data - ready for Firebase integration!


