# TGRide - Secure Carpooling App for Togo

A production-ready mobile carpooling application for Togo built with React Native, Expo, and Supabase. Features enterprise-grade robustness including offline-first sync, real-time messaging, and security-first architecture.

**[📚 Full Documentation](./INTEGRATION_GUIDE.md)** | **[🛡️ Robustness](./ROBUSTNESS_CHECKLIST.md)** | **[🔧 Integration](./UI_INTEGRATION_STEPS.md)**

## How can I edit this code?

There are several ways of editing your native mobile application.

### Development with Local Editor

You can work locally using your preferred code editor and push changes to GitHub.

Requirements:
- Node.js v18+ ([install via nvm](https://github.com/nvm-sh/nvm))
- Bun ([install](https://bun.sh/docs/installation))

Follow these steps:

```bash
# Step 1: Clone the repository using the project's Git URL.
git clone <YOUR_GIT_URL>

# Step 2: Navigate to the project directory.
cd <YOUR_PROJECT_NAME>

# Step 3: Install the necessary dependencies.
bun i

# Step 4: Start the instant web preview of your Rork app in your browser, with auto-reloading of your changes
bun run start-web

# Step 5: Start iOS preview
# Option A (recommended):
bun run start  # then press "i" in the terminal to open iOS Simulator
# Option B (if supported by your environment):
bun run start -- --ios
```

### **Edit a file directly in GitHub**

- Navigate to the desired file(s).
- Click the "Edit" button (pencil icon) at the top right of the file view.
- Make your changes and commit the changes.

## 🛠️ Tech Stack

- **Mobile Framework**: React Native 0.81
- **App Shell**: Expo 54
- **Routing**: Expo Router 6
- **State**: Zustand + Tanstack Query
- **Backend**: Supabase (PostgreSQL)
- **Real-time**: Supabase Realtime (WebSocket)
- **Maps**: React Native Maps + Mapbox
- **Notifications**: Expo Notifications + Sentry
- **Language**: TypeScript 5.9
- **Build**: EAS Build
- **Package Manager**: Bun

## 📁 Project Structure

```
app/                       # Expo Router screens
utils/                     # 9 robustness utilities
hooks/                     # Custom React hooks
supabase/                  # Edge Functions + Schema
contexts/                  # React Context
components/                # UI components
constants/                 # App configuration
lib/                       # Library setup
types/                     # TypeScript types
tests/                     # Test suites
```

## 🚀 Getting Started

### Requirements
- Node.js v18+ ([install via nvm](https://github.com/nvm-sh/nvm))
- Bun ([install](https://bun.sh/docs/installation))
- Supabase Account (free tier)
- Expo Account (free tier)

### Installation

```bash
git clone https://github.com/FreshTouchEvent/ride.git
cd ride
bun i
cp .env.example .env
```

Edit .env with your Supabase credentials.

### Development

```bash
# Start dev server
bun start

# iOS Simulator (press 'i')
bun start -- --ios

# Android Emulator
bun start -- --android

# Web preview
bun start-web
```

### Testing

```bash
bun test                  # All tests
bun run test:rls         # RLS security
bun run test:booking     # Booking edge cases
bun lint                 # Code quality
```

## 📦 Deployment

### Mobile Apps

```bash
# Install EAS CLI
bun i -g @expo/eas-cli

# iOS → App Store
eas build --platform ios
eas submit --platform ios

# Android → Google Play
eas build --platform android
eas submit --platform android
```

### Web

```bash
# Build for web
eas build --platform web

# Deploy to Vercel/Netlify
eas hosting:deploy
```

## 📚 Documentation

- [ROBUSTNESS_CHECKLIST.md](./ROBUSTNESS_CHECKLIST.md) - Feature inventory & monitoring setup
- [INTEGRATION_GUIDE.md](./INTEGRATION_GUIDE.md) - Code patterns for each feature
- [UI_INTEGRATION_STEPS.md](./UI_INTEGRATION_STEPS.md) - Step-by-step integration
- [PATTERN_TEMPLATES.md](./PATTERN_TEMPLATES.md) - Reusable patterns for new screens
- [CLAUDE.md](./CLAUDE.md) - Development command reference

## 🔧 Configuration

### Environment Variables

```env
EXPO_PUBLIC_SUPABASE_URL=your_url
EXPO_PUBLIC_SUPABASE_ANON_KEY=your_key
EXPO_PUBLIC_MAPBOX_TOKEN=your_token
SENTRY_DSN=your_dsn
EXPO_PUBLIC_DEBUG_MODE=false
```

### App Configuration
- Deep linking: tgride://
- iOS bundle: com.tgride.covoiturage
- Android package: com.tgride.covoiturage
- Permissions: Location (background), Camera, Notifications

## 🐛 Troubleshooting

| Issue | Solution |
|-------|----------|
| App not loading | Same WiFi + `bun start --tunnel` |
| Build failing | `bunx expo start --clear` |
| TypeScript errors | `bun tsc --noEmit` |
| RLS blocking | Check user auth + ID match |
| Realtime down | Enable realtime in Supabase |
| Sync queue stuck | Check network + AsyncStorage |

See [ROBUSTNESS_CHECKLIST.md](./ROBUSTNESS_CHECKLIST.md) for detailed monitoring.

## 📊 Monitoring

- **Error Tracking**: Sentry integration
- **Session Monitoring**: user_sessions table
- **Error Logs**: error_logs table with severity levels
- **Rate Limiting**: Per-user tracking
- **Offline Sync**: AsyncStorage queue

## 📝 License

Proprietary - All rights reserved

## 🤝 Contributing

Internal development. Push to main after code review.

---

**Built with ❤️ for secure, reliable carpooling in Togo**
