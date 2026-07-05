# Maintor Caching Strategy & Performance Analysis

This document provides a summary of Maintor's current caching configuration, explains the pros and cons of potential caching techniques to improve page load speed, and outlines a recommended roadmap for implementation.

---

## 1. Current Caching Architecture

### Static Frontend Assets
* **Static Assets (JS, CSS, images, webp, woff, etc.)**: Configured in firebase.json with aggressive HTTP cache-control headers (`public, max-age=31536000, immutable`). These are heavily cached by browsers/CDNs.
* **HTML Entrypoint (`index.html`)**: Configured with `Cache-Control: no-cache` in firebase.json. This ensures users always get code updates instantly when a new build is deployed.

### Client-Side Navigation
* The application is a Vue-based Single Page Application (SPA) using Quasar. Pages are not fetched from the server.
* **No Keep-Alive**: Views loaded in `<router-view />` in MainLayout.vue are not cached. Switching routes completely destroys the current page and recreates the target page.

### API & Database Queries
* **Settings & Config**: Cached globally in memory (settings.js) to avoid re-fetching account settings during a session.
* **Business Data (Tickets, Assets, Parts)**: Loaded fresh on every page mount. The Google Cloud Function API queries MongoDB on every page view without any caching layer or HTTP Cache headers.

---

## 2. Comparison of Caching Solutions

Implementing caching at different layers has distinct trade-offs. The table below outlines the **pros** and **cons** of each approach.

| Cache Layer | Description | Pros | Cons (Consequences & Risks) |
| :--- | :--- | :--- | :--- |
| **Option 1: Frontend Data Store** | Store fetched lists (e.g. tickets, assets) in a global shared context/store so navigating back reads from memory. | * Near-instantaneous page transitions.<br>* Drastically reduces backend and database load. | * **Stale Data:** Users won't see updates made by other technicians unless they manually refresh or we build invalidation/polling logic. |
| **Option 2: API Cache Headers** | Send `Cache-Control` response headers from the Google Cloud Function to browser cache. | * Very simple to implement.<br>* Offloads traffic from the DB. | * **Hard to Invalidate:** Browser HTTP caches are hard to force-clear from JavaScript. If a user updates a ticket and returns to the list, the browser may still serve the stale cache. |
| **Option 3: Keep-Alive Views** | Wrap `<router-view />` in Vue `<keep-alive>` to keep DOM nodes and component states in memory. | * Instant tab switching.<br>* Retains user scroll position and input focus. | * **Memory Leaks:** Storing many large pages/components in memory can bloat browser RAM.<br>* **No Updates:** `onMounted` hooks will not run when returning to a page, freezing the data indefinitely. |

---

## 3. Why Not Cache All Three?

If we blindly cache all three, it creates a **cache coordination problem** that results in stale data and synchronization bugs:

1. **Conflict in Invalidation**: If the browser caches the API response (Option 2) and Vue keeps the page alive (Option 3), a user who edits a ticket and goes back will see the *old, unedited* list. Even reloading the page will query the browser's HTTP cache and return the old data.
2. **Complexity Overhead**: Coordinating cache invalidation across client memory, Vue DOM components, and HTTP browser caches requires complex code, increasing the risk of race conditions and stale UI states.

---

## 4. Recommended Balanced Approach

To achieve the best speed improvements without causing stale data issues, we recommend the **Stale-While-Revalidate** pattern:

### Implementation Checklist
1. **Frontend Store Caching (Yes)**: Use the shared context/provider to cache data.
2. **Background Revalidation (Yes)**: When a component mounts, render the cached version *immediately* so the user sees a load time of **0ms**, then fetch fresh data from the API in the background. Once the request resolves, update the UI and the cache.
3. **Keep-Alive (No)**: Do not freeze component DOMs to ensure page transition hooks run correctly.
4. **API Caching (No)**: Keep API endpoints dynamic (`no-cache`) so background fetches are always guaranteed to fetch the latest state from MongoDB.
