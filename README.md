# AmpUp Platform - Social Thunderclap Gamification System

A comprehensive gamification platform for social media campaigns with user authentication, campaign management, leaderboards, achievements, prize wheel, battle pass, admin dashboard, and real-time analytics.

## Features

- 🎮 **Gamification System**: Points, levels, achievements, and leaderboards
- 🎁 **Rewards**: Prize wheel, battle pass, and reward redemption
- 📊 **Analytics**: Real-time campaign tracking and user statistics
- 👥 **User Management**: Authentication, profiles, and team management
- 🎯 **Campaign Management**: Create and manage social media campaigns
- 🏆 **Leaderboards**: Competitive rankings and department-based views
- 🔔 **Notifications**: Achievement unlocks and activity feed

## Tech Stack

- **Frontend**: HTML, CSS, JavaScript (Vanilla)
- **Backend**: Firebase (Firestore, Authentication)
- **Styling**: Tailwind CSS (CDN)
- **Charts**: Chart.js
- **Hosting**: Netlify

## Getting Started

### Local Development

1. Clone the repository:
```bash
git clone https://github.com/CalOneUp/ouamplify.git
cd ouamplify
```

2. Open `templates/ampup platform.html` in your browser or use a local server:
```bash
# Using Python
python -m http.server 8000

# Using Node.js
npx serve .
```

3. Access the platform at `http://localhost:8000/templates/ampup platform.html`

### Firebase Setup

1. Create a Firebase project at [Firebase Console](https://console.firebase.google.com/)
2. Enable Firestore Database
3. Enable Authentication (Anonymous and Email/Password)
4. Update the Firebase configuration in `templates/ampup platform.html`:
```javascript
const firebaseConfig = {
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_AUTH_DOMAIN",
    projectId: "YOUR_PROJECT_ID",
    // ... other config
};
```

## Deployment

### Netlify

1. Push your code to GitHub
2. Connect your repository to Netlify
3. Netlify will automatically deploy from the `main` branch
4. The site will be available at your Netlify URL

### Configuration

- **Build command**: (none needed for static site)
- **Publish directory**: `.` (root)
- **Redirects**: Configured in `netlify.toml`

## Project Structure

```
social-thunderclap/
├── templates/
│   └── ampup platform.html    # Main application file
├── modules/
│   └── Amplifier Platform.module/
├── README.md
├── netlify.toml               # Netlify configuration
└── .gitignore
```

## Development Guidelines

See [DEVELOPMENT_GUIDELINES.md](./DEVELOPMENT_GUIDELINES.md) for coding standards and best practices.

## License

[Add your license here]

## Contributing

[Add contribution guidelines here]
