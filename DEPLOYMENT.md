# Deployment Guide - GitHub & Netlify

## GitHub Setup

### Option 1: Using SSH (Recommended)

1. **Check if you have SSH keys:**
```bash
ls -al ~/.ssh
```

2. **If you don't have SSH keys, generate them:**
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

3. **Add your SSH key to GitHub:**
   - Copy your public key: `cat ~/.ssh/id_ed25519.pub`
   - Go to GitHub → Settings → SSH and GPG keys → New SSH key
   - Paste your key and save

4. **Update the remote URL to use SSH:**
```bash
cd "/Users/callumsaunders/Hubspot Theme/social-thunderclap"
git remote set-url origin git@github.com:CalOneUp/ouamplify.git
```

5. **Push to GitHub:**
```bash
git push -u origin main
```

### Option 2: Using Personal Access Token

1. **Create a Personal Access Token:**
   - Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Generate new token (classic)
   - Select scopes: `repo` (full control of private repositories)
   - Copy the token

2. **Push using the token:**
```bash
git push -u origin main
# When prompted for username: enter your GitHub username
# When prompted for password: paste your personal access token
```

### Option 3: Using GitHub CLI

1. **Install GitHub CLI:**
```bash
brew install gh
```

2. **Authenticate:**
```bash
gh auth login
```

3. **Push:**
```bash
git push -u origin main
```

## Netlify Deployment

### Step 1: Connect Repository

1. Go to [Netlify](https://app.netlify.com/)
2. Click "Add new site" → "Import an existing project"
3. Choose "GitHub" and authorize Netlify
4. Select the repository: `CalOneUp/ouamplify`
5. Select branch: `main`

### Step 2: Configure Build Settings

Netlify should auto-detect the settings from `netlify.toml`, but verify:

- **Build command**: (leave empty - no build needed)
- **Publish directory**: `.` (root directory)
- **Base directory**: (leave empty)

### Step 3: Environment Variables (if needed)

If you need to set Firebase config or other secrets:
1. Go to Site settings → Environment variables
2. Add any required variables

### Step 4: Deploy

1. Click "Deploy site"
2. Netlify will build and deploy your site
3. Your site will be available at: `https://[random-name].netlify.app`

### Step 5: Custom Domain (Optional)

1. Go to Site settings → Domain management
2. Click "Add custom domain"
3. Follow the DNS configuration instructions

## Post-Deployment

### Update Firebase Configuration

Make sure your Firebase configuration allows your Netlify domain:
1. Go to Firebase Console → Authentication → Settings → Authorized domains
2. Add your Netlify domain (e.g., `your-site.netlify.app`)

### Test the Deployment

1. Visit your Netlify URL
2. Test all features:
   - User authentication
   - Campaign creation
   - Leaderboards
   - Rewards system

## Troubleshooting

### Push Fails with 403 Error

- Use SSH instead of HTTPS (see Option 1 above)
- Or use a Personal Access Token (see Option 2 above)

### Netlify Build Fails

- Check the build logs in Netlify dashboard
- Verify `netlify.toml` is in the root directory
- Ensure all file paths are correct

### Firebase Errors

- Check Firebase console for errors
- Verify authorized domains include your Netlify URL
- Check browser console for specific error messages

## Quick Commands Reference

```bash
# Navigate to project
cd "/Users/callumsaunders/Hubspot Theme/social-thunderclap"

# Check status
git status

# Add changes
git add .

# Commit
git commit -m "Your commit message"

# Push to GitHub
git push origin main

# Check remote
git remote -v
```

