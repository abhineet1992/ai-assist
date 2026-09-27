# Project: AI Screen Copilot

> **Stage 1 — Requirements & Tech Stack** (this document)
> Later stages: 2) Architecture & Design · 3) Implementation · 4) Testing · 5) Packaging & Release

---

## 1. Overview
A **desktop application** that lets a user share their screen (live or on-demand) with an AI
assistant so the AI has visual context of what the user is working on. The user asks
questions in a chat window and reads the AI's response in a dedicated response pane. All
conversations are stored locally so the user can revisit past sessions.

The core value: **the AI "sees" the screen, so the user doesn't have to describe their
context** — they just point and ask.

## 2. Problem
Getting help from an AI on live work today requires manually copying code, screenshots,
error messages, or describing the situation in words. This is slow and lossy — the AI
misses context. There is no lightweight, always-available way to give an AI live visual
context of the actual thing being worked on.

## 3. Solution
A background desktop app with a compact overlay/window that:
1. Captures the screen (or a selected window/region) and sends visual context to a
   vision-capable AI model.
2. Lets the user type a question in a chat box.
3. Streams the AI's response into a response pane.
4. Persists the full chat + screenshot history locally.

## 4. Target Users
- Developers debugging live code / errors.
- Designers and analysts who want feedback on what's on screen.
- Anyone who wants "look at my screen and help me" assistance without copy-pasting.

## 5. Functional Requirements

### 5.1 Screen Capture
- **FR-1** A **Share Screen** button toggles capture on/off.
- **FR-2** User can choose capture source: **entire screen**, a **specific window**, or a
  **selected region**.
- **FR-3** Capture mode: **on-demand snapshot** (capture one frame when a question is sent)
  as the MVP default; **periodic frames** (e.g. 1 frame / N seconds) as an enhancement.
  > _Rationale: current LLMs consume images/frames, not live video streams. Snapshots keep
  > token cost and latency controlled._
- **FR-4** A live thumbnail preview shows what is currently being captured.
- **FR-5** Sensitive-content safeguard: a visible indicator whenever the screen is being
  captured, and a one-click **Pause** to stop sending frames.

### 5.2 Chat & AI Interaction
- **FR-6** A **user input** box to type a question/instruction.
- **FR-7** On send, the app packages `{ user text + latest captured frame(s) + recent
  chat context }` and calls the AI.
- **FR-8** An **AI response pane** renders the reply with **streaming** output and markdown
  (code blocks with syntax highlighting).
- **FR-9** Multi-turn conversation: prior turns are included as context (with a configurable
  window to control cost).

### 5.3 History & Persistence
- **FR-10** Store chat history (messages, timestamps, and referenced screenshots) locally.
- **FR-11** Browse, search, and reopen past conversations.
- **FR-12** Delete a conversation or clear all history.

### 5.4 Settings
- **FR-13** Configure AI provider/model and paste an API key (stored securely in the OS
  keychain, never in plaintext).
- **FR-14** Configure capture frequency, image resolution/quality (to manage cost), and
  context window length.
- **FR-15** Global hotkey to summon the app and ask a question without breaking flow.

## 6. Non-Functional Requirements
- **NFR-1 Performance:** end-to-end (send → first token) under ~2s on a typical connection.
- **NFR-2 Cost control:** downscale/compress screenshots before upload; cap frames per
  request; expose a per-session token/cost estimate.
- **NFR-3 Privacy:** capture only when explicitly enabled; frames processed in memory;
  nothing sent to third parties except the chosen AI provider; local-only storage by
  default.
- **NFR-4 Security:** API keys in OS keychain; no telemetry without opt-in.
- **NFR-5 Cross-platform:** macOS first (dev target), Windows second.
- **NFR-6 Footprint:** small install size and low idle CPU/RAM (background app).

## 7. Constraints & Assumptions
- Requires OS screen-recording permission (macOS: Screen Recording entitlement).
- Requires an AI provider account/API key with **vision (image input)** support.
- MVP is single-user, local; no cloud account or multi-device sync.

## 8. Out of Scope (MVP)
- Real-time continuous video streaming to the model.
- Multi-user / collaboration / cloud sync.
- The AI *controlling* the screen (mouse/keyboard automation).
- Mobile clients.

---

## 9. Ideal Technology Stack ⭐

> **Chosen stack: Tauri 2 (Rust core) + React + TypeScript + Vite, UI in Tailwind +
> shadcn/ui, native screen capture in Rust, Claude vision via a thin Rust command,
> and SQLite for local history.**
> A small, fast, secure native desktop app with a modern web UI. The app treats the
> capture as a single image — **whatever is visible goes into context** — so it works
> for any content (code, dashboards, browsers, chat, PDFs), not just one app type.

### 9.1 Core stack (the decisions)

| Layer | Choice | Version / package | Why this one |
|---|---|---|---|
| **App shell** | **Tauri 2** (Rust core + system webview) | `tauri@2`, `@tauri-apps/api@2` | ~3–10 MB bundle vs Electron's ~150 MB, low idle RAM, native capture + keychain via Rust, capability-scoped security (NFR-1/4/6). |
| **UI framework** | **React 18 + TypeScript** | `react@18`, `typescript@5` | Huge ecosystem for chat/streaming UIs; type safety across the IPC boundary. |
| **Bundler / dev** | **Vite** | `vite@5`, `@vitejs/plugin-react` | Instant HMR, first-class Tauri integration. |
| **Styling** | **Tailwind CSS** | `tailwindcss@3` | Fast, consistent, dark/light theming (the side-by-side layout in the mockup is pure utility classes). |
| **Components** | **shadcn/ui** (Radix primitives) | `radix-ui`, `lucide-react` icons | Accessible dialogs, menus, tooltips — used for the source picker, settings modal, history. |
| **Client state** | **Zustand** + **TanStack Query** | `zustand`, `@tanstack/react-query` | Zustand for UI/session state; Query for async calls to Rust commands with caching/retries. |
| **Markdown / code** | **react-markdown** + **Shiki** | `react-markdown`, `remark-gfm`, `shiki` | Rich, streaming-friendly markdown with accurate syntax highlighting. |

### 9.2 Capture, AI & data (the native + intelligence layer)

| Layer | Choice | Package (Rust crate) | Why this one |
|---|---|---|---|
| **Screen capture** | Native OS APIs in Rust | `scap` (ScreenCaptureKit on macOS; `windows-capture` on Windows) | Efficient frame grab of **entire screen / a window / a region** (FR-2), permission-aware, low CPU. |
| **Image pipeline** | Downscale + encode in Rust before upload | `image`, WebP/JPEG encode | Caps resolution & size to control token cost/latency (NFR-2) before the frame ever leaves the machine. |
| **AI model** | **Claude** with image input + streaming | Anthropic **Messages API** (`claude-sonnet-5` default; `claude-haiku-4-5` for quick turns, `claude-opus-5` for hard problems) | Strong vision + reasoning; native token streaming for a responsive response pane (FR-8, NFR-1). |
| **AI call location** | Thin **Rust command** (Tauri `#[command]`) | `reqwest` (streaming), `tokio` | Keeps the API key **out of the webview JS**, adds retry/backoff & rate-limit; streams tokens to the UI over Tauri events (NFR-4). |
| **Local storage** | **SQLite** | `tauri-plugin-sql` (or `rusqlite` + `sqlx`) | Conversations, messages, timestamps & screenshot *paths* in DB; image files on disk (FR-10/11/12). |
| **Secrets** | **OS keychain** | `keyring` crate / macOS Keychain (Stronghold optional) | API key stored in the OS keychain, never plaintext (FR-13, NFR-4). |
| **Global hotkey** | Tauri global shortcut | `tauri-plugin-global-shortcut` | Summon-and-ask without breaking flow (FR-15). |
| **Packaging / update** | Tauri bundler + updater | `.dmg` / `.msi`, code signing, `tauri-plugin-updater` | Signed native installers and auto-update (Stage 5). |

### 9.3 Tooling & quality

| Concern | Choice |
|---|---|
| **Lint / format** | ESLint + Prettier (TS), `rustfmt` + `clippy` (Rust) |
| **Testing** | Vitest + React Testing Library (UI), `cargo test` (Rust), Playwright for end-to-end where feasible |
| **CI** | GitHub Actions — build/test the Rust + web sides, produce signed macOS/Windows artifacts |
| **Package manager** | pnpm (JS) + Cargo (Rust) |

### 9.4 Rationale & alternatives

- **Why not Electron?** Valid all-JS fallback with the biggest capture-library ecosystem
  (`desktopCapturer` + `getDisplayMedia`), but it's heavy for a background app — Tauri
  better fits **NFR-1 (latency)** and **NFR-6 (footprint)**.
- **Provider-pluggable option:** the AI call lives behind one Rust command with a small
  trait/interface, so swapping in another vision model later is a localized change — but
  Claude is the committed default.
- **Key principle:** the frontend never sees the API key and never talks to the network
  directly; all AI/network calls go through Rust.

**Suggested MVP slice (fastest path to value):** Tauri + React, **on-demand snapshot**
capture of the **entire screen**, a single streaming Claude vision call, and SQLite
history — defer periodic frames, window/region capture, global hotkey, and Windows
support to later stages.

---

## 10. Open Questions (to confirm before Stage 2)
- [ ] Primary OS for the MVP — macOS only, or macOS + Windows?
- [ ] AI provider — commit to **Claude**, or keep it provider-pluggable?
- [ ] Capture default — **snapshot per question** vs **periodic frames**?
- [ ] Is a global hotkey / always-on overlay required for the MVP, or a normal window?

## 11. Milestones
- [ ] **Stage 1:** Requirements & tech stack (this doc) — _in review_
- [ ] **Stage 2:** Architecture, data model, and UI wireframes
- [ ] **Stage 3:** MVP implementation (capture → ask → stream → store)
- [ ] **Stage 4:** History browser, settings, cost/privacy controls
- [ ] **Stage 5:** Packaging, signing, and release
