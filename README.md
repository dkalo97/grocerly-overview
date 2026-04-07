# Grocerly

Grocerly is an AI-powered recipe and grocery management app for iOS. Users build a pantry, get smart recipe suggestions based on what they have, and generate shareable grocery lists — all assisted by a Claude-powered chat interface. It targets home cooks who want to reduce food waste and simplify meal planning.

> **Note:** This repository is private during the active App Store launch window to protect unreleased product details. Source access is available to hiring managers and technical reviewers on request.

---

## Why is this repo private?

This codebase is private during the active App Store review and launch window. Making it public before approval would expose product details, prompt engineering, and configuration that are part of the competitive product surface. I plan to open-source portions of the architecture (hooks patterns, Capacitor + Firebase integration) after launch.

---

## Technical Architecture

### Frontend — React 19 PWA + Capacitor iOS

The app is built as a PWA with [Capacitor](https://capacitorjs.com/) wrapping it into a native iOS binary. There is no React Native or Expo in the stack — the entire UI is standard React running in a WKWebView with native bridge calls for OAuth, in-app purchases, and file access.

- **React 19 + Vite** — JSX throughout, no TypeScript
- **Inline styles only** — no CSS framework, no Tailwind, no CSS modules; all styles are plain JS objects
- **22 custom hooks** — state and logic are decomposed into domain hooks (`usePantryState`, `useRecipeState`, `useChatState`, `useFriendsState`, etc.) and handler hooks (`usePantryHandlers`, `useChatSend`, `useRecipeImport`, `useRecipeSharing`, etc.). `App.jsx` is the orchestrating root; it composes hooks and passes props down to view components.

### Backend — Vercel Serverless + Claude Streaming

All AI calls are proxied through a Vercel serverless function (`api/chat.js`) that holds the Anthropic API key server-side and streams responses back to the client using the Anthropic Node SDK. The client never touches the API key. Streaming is handled with `ReadableStream` / `TextDecoder` on the frontend consuming chunked SSE output.

- **Model**: Claude Sonnet 4 (latest)
- **Prompts**: system prompt injected server-side per-request with pantry/recipe context
- **Secondary endpoint**: `api/fetch.js` for server-side HTML fetching of recipe URLs (bypasses CORS)

### Auth — Firebase

Authentication is handled by Firebase with three providers:

| Provider | Path |
|---|---|
| Google OAuth | Capacitor Google Auth plugin → Firebase credential |
| Apple OAuth | Capacitor Sign In with Apple plugin → Firebase credential |
| Email/password | Firebase native, with custom password validation |

`AuthContext.jsx` wraps the Firebase SDK and exposes auth state app-wide. Native OAuth flows use Capacitor plugins that exchange tokens with Firebase rather than the web popup flow.

### Persistence — localStorage + Firestore Sync

The primary data store is `localStorage` for fast, offline-first reads. Firestore acts as a cloud backup layer:

- On login, `useFirebaseSync` restores pantry, recipes, preferences, and grocery list from Firestore
- Changes debounce-sync to Firestore with a 2-second delay
- Chat history, usage counters, and UI state are `localStorage`-only (never synced)
- Raw Firestore operations live in `src/utils/firestore.js` with dual-path logic for native (Capacitor) vs. web environments

### Monetization — RevenueCat Paywall

Subscriptions are managed through [RevenueCat](https://www.revenuecat.com/) with the Capacitor Purchases plugin. The paywall (`PaywallModal.jsx`) gates AI chat usage above a free-tier message limit. Entitlement checks run client-side against the RevenueCat SDK; the free tier allows a fixed number of AI messages before prompting upgrade.

### Social — Friends & Recipe Sharing

Users can add friends by username, view their shared recipe collections, and send recipes directly. The friends system is backed by Firestore with a bidirectional relationship model. `useFriendsState`, `useFriends`, and `useRecipeSharing` handle state, Firestore ops, and the sharing flow respectively. `AddFriendModal` and `ShareRecipeModal` are the UI entry points.

---

## Project Structure

```
src/
  App.jsx                  # Root orchestrator — composes all hooks and views
  hooks/                   # 22 custom hooks (state + handlers, by domain)
  components/              # View components: PantryView, RecipeView, ChatView,
  │                        #   GroceryListView, AccountView, Login, and modals/
  context/AuthContext.jsx  # Firebase auth state
  utils/
    helpers.js             # AI API calls, ingredient normalization
    firestore.js           # Native + web Firestore operations
    categorization.js      # Ingredient → category mapping
  data/constants.js        # Categories, templates, UI tokens
api/
  chat.js                  # Vercel serverless: streams Claude responses
  fetch.js                 # Vercel serverless: server-side URL fetch proxy
ios/                       # Capacitor iOS project (Xcode)
```

---

## App Store

- **Live web app/PWA**: [grocerly-six.vercel.app](https://grocerly-six.vercel.app)
- **iOS App Store listing**: coming soon

---

I'm happy to do a live walkthrough or grant temporary read access. Reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.
