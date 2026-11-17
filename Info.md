# Info.md


## Repo layout

The actual project lives in `imessage/` (this is the git repo root, not the workspace root). It's a monorepo with two independently-deployed apps:

- `imessage/backend` — Node.js/Express ESM API + Socket.io server
- `imessage/frontend` — React 19 + Vite SPA

There is no test suite in this repo.

## Commands

Run these from `imessage/backend` or `imessage/frontend` respectively (no root-level package.json/workspace tooling).

Backend:
```bash
npm run dev       # nodemon src/index.js — requires a .env (see Environment variables below)
npm run build     # rm -rf dist && cp -R src dist (no transpilation; ESM copied as-is)
npm start         # node src/index.js
npm run db:seed   # node src/seeds/user.seed.js — upserts ~20 fake users for local testing
```

Frontend:
```bash
npm run dev       # vite dev server, expects backend on localhost:3000
npm run build     # vite build
npm run lint      # eslint .
npm run preview
```

There's no single-test-runner command since there are no tests; verifying a change means running both dev servers and exercising the flow in the browser (see the `run` skill).

## Environment variables

Backend (`imessage/backend/.env`):
`PORT`, `NODE_ENV`, `MONGO_URI`, `CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `CLERK_WEBHOOK_SIGNING_SECRET`, `IMAGEKIT_PRIVATE_KEY`, `FRONTEND_URL`

Frontend (`imessage/frontend/.env`):
`VITE_CLERK_PUBLISHABLE_KEY`

In dev, the frontend hardcodes API/socket calls to `http://localhost:3000` (see `frontend/src/lib/axios.js` and `frontend/src/store/useAuthStore.js`); in production it uses relative `/api` and same-origin sockets because the backend serves the built SPA itself.

## Architecture

### Deployment shape
Production is a single container: the multi-stage `Dockerfile` builds the Vite SPA and copies its output into the backend's `public/` dir, then `backend/src/index.js` serves both the `/api/*` routes and the static SPA (with a catch-all `sendFile` fallback to `index.html`) from one Express process on one port. In dev, frontend and backend run as separate servers (Vite on 5173, API on `PORT`).

### Auth: Clerk owns identity, Mongo mirrors it
Clerk is the source of truth for authentication; the app never writes users directly. Instead `backend/src/webhooks/clerk.webhook.js` receives `user.created`/`user.updated`/`user.deleted` events (verified via `verifyWebhook`, requires the **raw** body — this route is mounted with `express.raw()` *before* `express.json()` in `index.js`) and upserts/deletes a local `User` document keyed by `clerkId`.

`backend/src/middleware/auth.middleware.js` (`protectRoute`) reads the Clerk session via `getAuth(req)`, then looks up the matching local `User` by `clerkId` and attaches it as `req.user`. If the webhook hasn't synced the user yet, this middleware 404s with "User profile is not synced yet" — a real race condition to be aware of when working on auth-adjacent code.

Frontend mirrors this two-step model: `@clerk/react`'s `useAuth()` gates routing in `App.jsx`, while `useAuthStore.checkAuth()` separately calls `/api/auth/check` to load the local Mongo profile and only then opens the socket connection.

### Realtime layer
`backend/src/lib/socket.js` creates one shared Express `app` + `http.Server` + `socket.io` `Server` and exports all three; `index.js` imports `{ app, server }` from here rather than creating its own, so route registration in `index.js` and socket handling in `socket.js` share the same underlying server. Online presence is tracked in-memory (`userSocketMap: { userId: socketId }`, not persisted), broadcast as `getOnlineUsers` on every connect/disconnect. There's no multi-instance/adapter support — presence and delivery both assume a single Node process.

Message send (`message.controller.js#sendMessage`) writes to Mongo first, then emits `newMessage` directly to the receiver's socket if they're online (`getReceiverSocketId`). If the receiver is offline, the message is only picked up on their next `getMessages` fetch — there's no offline queue/notification.

### Frontend state (Zustand)
Two stores, no React Context for server state:
- `useAuthStore` — auth user, online users list, and the socket instance itself. `connectSocket`/`disconnectSocket` live here.
- `useChatStore` — users, conversations, messages, selected conversation, composer text, sidebar tab. Uses `zustand/middleware persist` but `partialize`s down to just `isSoundEnabled` (localStorage key `imessage-storage`) — everything else is intentionally not persisted across reloads.

`subscribeToMessages(userId)` is scoped to whichever conversation is currently open (filters incoming socket messages by `senderId === userId`) and is re-subscribed via a `useEffect` in `ChatPage.jsx` keyed on `activeConversationId`.

Theming (`ThemeContext`) and wallpaper (`WallpaperContext`) are separate React Contexts, each backed by their own `localStorage` key and applied to the DOM imperatively via `useLayoutEffect` (class/attribute toggling), independent of the Zustand stores.

### Media uploads
Single pipeline for images and videos: `multer` memory storage (`upload.middleware.js`, 25MB limit, image/video mimetypes only) → `lib/imagekit.js#uploadChatMedia` uploads the buffer to ImageKit and returns a URL, which is stored on the `Message` doc as either `image` or `video` depending on mimetype. If `IMAGEKIT_PRIVATE_KEY` isn't set, media sends fail with a 500 rather than silently no-op'ing.

### Cron
`lib/cron.js` self-pings `${FRONTEND_URL}/health` every 14 minutes to prevent the free-tier host from idling out; it's only started in `index.js` when `NODE_ENV === "production"`.
