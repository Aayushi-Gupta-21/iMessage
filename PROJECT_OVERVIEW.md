# iMessage — Project Overview

A full-stack, real-time 1:1 chat application styled after Apple's iMessage. React/Vite SPA on the frontend, Node/Express + Socket.io on the backend, MongoDB for storage, Clerk for authentication, and ImageKit for media hosting.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite, Tailwind CSS 4, Hero UI, Zustand, React Router 7, Socket.io client |
| Backend | Node.js (ESM), Express 5, Socket.io, Mongoose/MongoDB |
| Auth | Clerk (`@clerk/express` server-side, `@clerk/react` client-side) |
| Media | ImageKit (image/video upload + optimization), Multer (in-memory upload handling) |
| Deployment | Single Docker image (Render), MongoDB Atlas |

## Repository layout

```
imessage/
├── backend/
│   └── src/
│       ├── controllers/    # request handlers (auth, messages)
│       ├── routes/         # Express routers
│       ├── middleware/     # protectRoute (Clerk auth), upload (multer)
│       ├── models/         # Mongoose schemas: User, Message
│       ├── lib/            # db connection, socket.io server, cron job, ImageKit client
│       ├── webhooks/       # Clerk webhook handler (keeps Mongo users in sync)
│       ├── seeds/          # local dev user seeding script
│       └── index.js        # app entrypoint — wires routes, static SPA serving
├── frontend/
│   └── src/
│       ├── pages/          # AuthPage, ChatPage
│       ├── components/     # auth/ and chat/ UI components
│       ├── store/          # Zustand: useAuthStore, useChatStore
│       ├── context/        # ThemeContext, WallpaperContext
│       ├── hooks/          # useSelectedConversation, useKeyboardSound, useMediaQuery, ...
│       ├── data/           # wallpapers.js, herouiThemePresets.js
│       └── lib/            # axios instance, misc utils
└── Dockerfile               # multi-stage build → single production image
```

## Core features

- **Real-time 1:1 messaging** via Socket.io, with live online-presence indicators
- **Text, image, and video messages** — media uploaded through the backend to ImageKit
- **Conversation list vs. full user directory** — sidebar tabs for "Chats" (people you've messaged) and "Users" (everyone), both searchable
- **Light/dark mode** plus 11 Hero UI color theme presets
- **13 desktop/abstract wallpapers** applied as the app's background frame
- **Optional keyboard sound effects** (random keystroke sound on typing/sending)
- **Responsive layout** — single-pane on mobile (sidebar or conversation, not both), two-pane on large screens
- **Authentication** fully delegated to Clerk (sign in/up UI, session management)

## Authentication flow

Clerk is the identity provider; the app's own MongoDB `User` collection is a **read-only mirror** kept in sync via webhook — the app never creates or edits users directly.

1. User signs in through Clerk's UI (`ClerkProvider` wraps the app in `main.jsx`).
2. Clerk sends `user.created` / `user.updated` / `user.deleted` events to `backend/src/webhooks/clerk.webhook.js`, which verifies the signature (`verifyWebhook`, requires the **raw** request body) and upserts/deletes the matching `User` document by `clerkId`.
3. On the client, `App.jsx` watches Clerk's `useAuth()` (`isSignedIn`/`isLoaded`) to route between `/auth` and `/`.
4. Once signed in, `useAuthStore.checkAuth()` calls `GET /api/auth/check`, which is protected by `protectRoute` middleware. That middleware reads the Clerk session (`getAuth(req)`) and looks up the local `User` by `clerkId`, attaching it as `req.user`.
5. If the webhook hasn't synced the user yet, `protectRoute` responds 404 ("User profile is not synced yet") — a real timing edge case right after signup.
6. Once the local profile loads, `checkAuth()` opens the Socket.io connection (`connectSocket`).

## Real-time messaging flow

- `backend/src/lib/socket.js` creates one shared Express `app`, `http.Server`, and Socket.io `Server`, and exports all three. `index.js` imports `{ app, server }` from there instead of creating its own — routes and socket handling share one process/port.
- Online presence is tracked **in-memory only** (`userSocketMap: { userId → socketId }`), rebroadcast as `getOnlineUsers` to everyone on every connect/disconnect. This does not persist and does not scale beyond a single Node instance (no Redis adapter).
- Sending a message (`POST /api/messages/send/:id`) writes to MongoDB first, then — if the receiver currently has a socket connected — emits `newMessage` directly to them. If they're offline, they simply pick the message up on their next `GET /api/messages/:id` fetch; there's no offline notification queue.
- On the client, `useChatStore.subscribeToMessages(userId)` is scoped to whichever conversation is currently open and is re-subscribed (via `ChatPage`'s `useEffect`) whenever the active conversation changes.

## Media uploads

`ChatComposer` lets the user attach a single image or video per message. The file goes through:

1. `multer` (memory storage, 25MB limit, image/video mimetypes only — `upload.middleware.js`)
2. `lib/imagekit.js#uploadChatMedia` uploads the buffer to ImageKit under `/chat` and returns a public URL
3. The URL is saved on the `Message` document as `image` or `video` depending on mimetype

If `IMAGEKIT_PRIVATE_KEY` isn't configured, media sends fail with a 500 rather than degrading silently.

## Data models

**User** (`backend/src/models/user.model.js`) — mirrors a Clerk identity: `clerkId` (unique), `email`, `fullName`, `profilePic`.

**Message** (`backend/src/models/message.model.js`) — `senderId`/`receiverId` (refs to `User`), optional `text`, optional `image`, optional `video`, timestamps.

The "conversation list" isn't a stored entity — it's computed on read via a Mongo aggregation (`getConversationsForSidebar`) that groups a user's messages by chat partner and returns the most-recently-active partners first.

## Frontend state management

Two Zustand stores, no React Context for server data:

- **`useAuthStore`** — the authenticated user, the online-users list, and the socket instance itself (`connectSocket`/`disconnectSocket` live here).
- **`useChatStore`** — users, conversations, messages, selected conversation, search query, sidebar tab, composer text, sound preference. Uses `zustand/middleware persist`, but `partialize`s down to just `isSoundEnabled` (localStorage key `imessage-storage`) — everything else resets on reload.

`useSelectedConversation` (a hook, not a store) adapts the raw `users`/`messages` data into the view-model the chat UI renders (`activeConversation.peer`, `activeConversation.messages` with `role: "me" | "them"`), and also exposes `isLargeScreen` for responsive layout decisions.

Theming (`ThemeContext`) and wallpaper (`WallpaperContext`) are separate, each backed by its own `localStorage` key and applied directly to the DOM (class/attribute toggling) in a `useLayoutEffect`, independent of the Zustand stores.

## Production deployment

A single multi-stage `Dockerfile`:

1. **Stage 1** builds the Vite SPA (`frontend/dist`)
2. **Stage 2** "builds" the backend (ESM, so this is just `cp -R src dist`)
3. **Stage 3** runtime image: installs backend prod deps only, copies the backend `dist/` and the frontend build into `public/`

At runtime, one Express process on one port serves `/api/*` routes and the built SPA (with a catch-all route that serves `index.html` for client-side routing). This is also why the frontend talks to the backend via same-origin `/api` and same-origin sockets in production, but hardcodes `http://localhost:3000` in development.

A cron job (`lib/cron.js`) self-pings `${FRONTEND_URL}/health` every 14 minutes in production to prevent the free-tier host from idling.

## Environment variables

**Backend:** `PORT`, `NODE_ENV`, `MONGO_URI`, `CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `CLERK_WEBHOOK_SIGNING_SECRET`, `IMAGEKIT_PRIVATE_KEY`, `FRONTEND_URL`

**Frontend:** `VITE_CLERK_PUBLISHABLE_KEY`

## Known rough edges

- No test suite exists anywhere in the repo.
- Presence tracking and message delivery both assume a single backend process (no pub/sub adapter for horizontal scaling).
- New-signup race: a user can be authenticated by Clerk slightly before the webhook has created their Mongo `User`, causing a transient 404 from `protectRoute`.
- No offline push/notification system — offline users only see new messages when they next open the conversation.
