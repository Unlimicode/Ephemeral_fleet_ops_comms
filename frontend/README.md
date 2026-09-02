# SwiftLink — Frontend

React + Vite PWA for the SwiftLink privacy-preserving fleet operations platform.
Serves all three actor interfaces: fleet manager dashboard, driver PWA, and
client trip-booking PWA.

See the [root README](../README.md) for the full project overview, architecture,
and setup instructions.

## Quick start

```bash
npm install
npm run dev      # Vite dev server
npm run lint     # ESLint
npm run build    # production build to dist/
```

`VITE_API_URL` and `VITE_WS_URL` (see `.env.example`) point the Axios client and
Socket.IO connection at the backend.

## Layout

```
src/
  api/          Axios instance + Bearer token interceptor
  context/      AuthContext — token, role, user (sessionStorage-backed)
  hooks/        useChat, usePushNotifications, useOnlineStatus, useWindowWidth, ...
  components/   Shared UI; components/layout/ holds the manager and driver shells
  pages/        Route components, grouped by actor: manager/, driver/, booking/
  styles/       tokens.css (design tokens) + animations.css
  utils/        ripple, compliancePdf
```
