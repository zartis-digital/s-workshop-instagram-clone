# Technical Summary

This is the **frontend** of an Instagram clone: a TanStack Start (file-based
routing) + React 19 + Tailwind v4 SPA that talks to a separately hosted Hono
API over REST + WebSocket. The backend service is not part of this repository.

The app is functional end-to-end for most core flows, but a number of
mutations are currently unimplemented no-op stubs or missing wiring — these
are called out explicitly below so the current runtime state of each feature
is clear.

## Stack

- **Routing**: TanStack Start / TanStack Router, file-based (`src/routes`), with route masks so modal routes (e.g. `/p/$postId/modal`) display a clean standalone URL.
- **Server state**: TanStack Query (`useQuery`/`useInfiniteQuery`/`useMutation`), cursor-based pagination, optimistic updates with rollback for several mutations.
- **Styling**: Tailwind v4 + shadcn/ui-style primitives (`src/components/ui`).
- **Forms**: TanStack Form + Zod.
- **Uploads**: `@better-upload/client`, posting directly to the hosted API's `/upload` endpoint (routes used: `posts`, `stories`, `messages`; no route is wired for avatars — see Uploads section below).
- **Realtime**: two native WebSocket connections (see below), no socket library.
- **Env config**: `src/env.ts` (`@t3-oss/env-core`) — `VITE_API_URL`, `VITE_WS_URL`, `VITE_STORAGE_PUBLIC_URL`.

## Pages / Routes

| Route | Purpose | Auth-gated |
|---|---|---|
| `/sign-in`, `/sign-up` | Email/password auth forms | redirects away if already logged in |
| `/` (`_feed`) | Home feed, stories bar, right-rail suggestions | ✅ |
| `/explore` + `/explore/p/$postId/modal` | Explore grid (infinite scroll), post modal overlay | ✅ |
| `/reels` | Reels viewer — **uses local mock data** (`src/lib/mock-data.ts`), not the API; like/follow/bookmark buttons only toggle local component `useState` and are not persisted anywhere | ✅ |
| `/p/$postId` + `/$username/p/$postId.modal` / `/_feed/p/$postId.modal` | Standalone post page / feed-overlaid modal (masked routes) | ✅ |
| `/$username` | Profile page — header, highlights, post grid | ✅ |
| `/stories/$username` (+ `.modal` variants) | Story viewer | ✅ |
| `/direct`, `/direct/inbox`, `/direct/$conversationId` | Messaging: inbox list layout, empty state, conversation thread | ✅ |
| `/settings/edit-profile` | Edit display name, bio, avatar | ✅ |
| `/settings/account-privacy` | Public/private account toggle | ✅ |
| `/settings/notifications` | Per-category notification toggles | ✅ |
| `/settings/saved` | Saved posts grid | ✅ |

Auth gating happens in `beforeLoad` on each route (or its parent layout), redirecting to `/sign-in` when `context.user` is falsy. `context.user` is populated once in the root route via `GET /auth/me`.

Two index routes are pure redirects: `/direct/` → `/direct/inbox`, and `/settings/` → `/settings/edit-profile`.

## Backend API Endpoints Consumed

All paths are relative to `VITE_API_URL`. Swagger docs for the hosted API are available at `/swagger` on that origin.

### Auth
| Method | Path | Notes |
|---|---|---|
| GET | `/auth/me` | Loaded once in root route; 401 → treated as logged out |
| POST | `/auth/login` | `{ email, password }` |
| POST | `/auth/register` | `{ email, password, username, displayName }` |
| POST | `/auth/logout` | |

### Posts / Feed
| Method | Path | Notes |
|---|---|---|
| GET | `/posts?limit&cursor` | Home feed, cursor-paginated |
| GET | `/posts/explore?limit&cursor` | Explore grid |
| GET | `/posts/saved` | Saved posts list |
| GET | `/posts/:id` | Post detail (incl. comments) |
| POST | `/posts/:id/save` | Toggle save, optimistic |
| DELETE | `/posts/:id` | Delete own post |
| POST | `/posts` | Create post `{ imageUrls, caption }` — **not implemented, dialog cannot submit** |
| POST/DELETE | `/posts/:postId/comments/:commentId/like` | **not implemented** |
| POST | `/posts/:postId/comments` | **not implemented** |
| — | like a post | **broken** — `useLikeMutation` is currently a no-op (does not call the API) |

### Stories
| Method | Path | Notes |
|---|---|---|
| GET | `/stories/feed` | Other users' stories |
| GET | `/stories/me` | Own stories |
| GET | `/stories/:id` | Also used to record a view (side-effecting GET) |
| POST | `/stories` | Create story `{ segments: [{ mediaUrl, mediaType }] }` |
| POST/DELETE | `/stories/:id/like` | |
| POST | `/stories/:id/reply` | `{ content }` |
| DELETE | `/stories/:id` | |

### Users / Profile / Social graph
| Method | Path | Notes |
|---|---|---|
| GET | `/users/:username` | Profile |
| GET | `/users/:username/posts?limit&cursor` | User's post grid |
| GET | `/users/suggested` | "Suggested for you" |
| GET | `/users/search?q=` | Live search |
| GET/POST/DELETE | `/users/search/recent[/:id]` | Recent search history |
| GET | `/users/me/follow-requests` | Pending incoming requests |
| POST | `/users/me/follow-requests/:requesterId/accept` \| `/reject` | **not implemented** |
| POST/DELETE | `/users/:userId/follow` | Follow/unfollow — **not implemented**, called from 3 call sites: profile page, suggestions panel, and the notifications sheet's follow-back button |
| PATCH | `/users/me` | Update profile `{ displayName?, bio?, avatarUrl?, isPrivate? }` — **not implemented** |
| GET/PATCH | `/users/me/notification-settings` | Per-category toggles — implemented |

### Messaging
| Method | Path | Notes |
|---|---|---|
| GET | `/messages` | Conversation list (polled every 30s) |
| GET | `/messages/unread-count` | Polled every 30s |
| GET | `/messages/:conversationId/messages?limit&cursor` | Message history |
| POST | `/messages` | Start new conversation `{ recipientId, content, messageType?, sharedPostId? }` |
| POST | `/messages/:conversationId/messages` | Send message (optimistic, with temp-ID reconciliation) |
| PUT | `/messages/:conversationId/messages/:messageId` | Edit message (optimistic) |
| DELETE | `/messages/:conversationId/messages/:messageId` | Soft-delete (optimistic) |
| POST | `/messages/:conversationId/read` | Mark read (optimistic) |
| GET | `/users/:username` (raw `fetch`, not via `lib/api`) | Used only by the floating messages widget's quick "new conversation" button (`messages-widget.tsx`) to resolve a typed username before starting a chat — a separate code path from the proper `/direct` inbox's "New message" dialog, which uses `useSearchUsersQuery` (`/users/search?q=`) instead |

### Notifications
| Method | Path | Notes |
|---|---|---|
| GET | `/notifications?limit&cursor` | Cursor-paginated, polled every 30s while panel open |
| GET | `/notifications/unread-count` | Polled every 30s |
| POST | `/notifications/read` | Mark all read |

### Uploads
| Method | Path | Notes |
|---|---|---|
| POST | `/upload` (route: `posts`) | Post image upload, via `@better-upload/client` (`create-post-dialog.tsx`) |
| POST | `/upload` (route: `stories`) | Story media upload (`create-story-dialog.tsx`) |
| POST | `/upload` (route: `messages`) | In-chat image sharing (`direct/$conversationId.tsx`) |
| Avatar upload | — | **Not implemented** — edit-profile page has no upload route wired up (stub) |
| — | `VITE_STORAGE_PUBLIC_URL` | Public origin serving uploaded files (MinIO in the hosted env); uploaded object keys are combined with this to build the final image URL |

`src/components/upload.tsx` (`ImageUploader`, route: `posts`) is a standalone example component that is **not imported or rendered anywhere** — dead/reference code, not part of any real flow.

### WebSocket
| Endpoint | Scope | Purpose |
|---|---|---|
| `wss://.../ws/users/me` | One per authenticated session, mounted at root layout | Pushes `conversation_updated` events to refresh inbox/unread badge globally |
| `wss://.../ws/conversations/:id` | One per open conversation | Pushes `new_message`, `typing`, `message_edited`, `message_deleted`, `read_receipt`; both sockets auto-reconnect with exponential backoff (max 30s) |

## Feature Status Summary

**Fully implemented**: sign-in/sign-up, feed + explore browsing, post detail view, save/unsave post, delete post, stories (create/view/like/reply/delete), direct messaging (send/edit/delete/mark-read) with live WebSocket updates and typing indicators, notifications feed + unread badges + per-category settings, user search + recent searches, saved posts view.

**Not yet implemented / broken (buttons and forms exist in the UI but don't call the API)**:
- Liking a post (`useLikeMutation` is a no-op)
- Creating a new post (dialog collects the image/caption but never submits)
- Commenting on a post, liking a comment
- Following / unfollowing a user (profile page and suggestions panel)
- Accepting / rejecting a follow request
- Updating profile (display name, bio, avatar) and the account-privacy toggle
- Avatar upload on the edit-profile page

**Using placeholder data instead of the live API**: `/reels` renders static mock data from `src/lib/mock-data.ts` rather than fetching real reels from the backend.

## Screenshots

Reference screenshots of every functional page are stored in [`screenshots/`](screenshots) at the project root, captured against a real signed-up account on the hosted API:

| File | Page |
|---|---|
| `01-sign-in.png` | `/sign-in` |
| `02-sign-up.png` | `/sign-up` |
| `03-home-feed.png` | `/` home feed |
| `04-explore.png` | `/explore` |
| `05-reels.png` | `/reels` (mock data) |
| `06-profile.png` | `/$username` own profile |
| `07-post-detail.png` | `/p/$postId` standalone post |
| `08-story-viewer.png` | `/stories/$username` |
| `09-direct-inbox-empty.png` | `/direct/inbox` empty state |
| `10-new-message-dialog.png` | New message dialog |
| `11-direct-conversation.png` | `/direct/$conversationId` |
| `12-settings-edit-profile.png` | `/settings/edit-profile` |
| `13-settings-account-privacy.png` | `/settings/account-privacy` |
| `14-settings-notifications.png` | `/settings/notifications` |
| `15-settings-saved.png` | `/settings/saved` |

Pages with broken/unimplemented mutations (e.g. create post, follow/unfollow) are still shown as they render, since the surrounding page is functional even though the specific action doesn't persist.
