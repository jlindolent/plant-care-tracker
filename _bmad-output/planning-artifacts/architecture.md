---
stepsCompleted: [1, 2, 3]
inputDocuments: ['_bmad-output/planning-artifacts/prd.md', '_bmad-output/planning-artifacts/ux-design-specification.md']
workflowType: 'architecture'
project_name: 'plant-care-tracker'
user_name: 'Jared'
date: '2026-04-02'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._


## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
The application centers on personalized plant care management with 37 functional requirements across 7 core areas:

1. **Plant Inventory Management (FR1-FR6):** CRUD operations for plant collection with species selection, custom naming, and search/filter capabilities
2. **Care Schedule Management (FR7-FR13):** Species-based default schedules with per-plant customization, pause/resume for travel mode, and upcoming task views
3. **Reminder & Notification System (FR14-FR19):** Local push notifications with plant-specific content, user-configurable timing, automatic rescheduling, and in-app notification history fallback
4. **Care Logging & History (FR20-FR24):** Timestamped care action logging with backdating support and comprehensive history views
5. **Species Database (FR25-FR28):** 50+ common houseplants with default care schedules, local caching for offline access
6. **Offline Mode & Data Sync (FR29-FR34):** Complete offline functionality for core workflows with background sync and last-write-wins conflict resolution
7. **User Onboarding (FR35-FR37):** Guided setup with notification permission requests and skip-friendly flow

**Non-Functional Requirements:**
Critical NFRs that will drive architectural decisions:

- **Performance:** 2-second app launch, 1-second user actions, 500ms species search, 5-second sync for typical user (10-15 plants)
- **Notification Reliability:** 95% delivery within 1-hour window, <100ms scheduling logic
- **Offline-First:** Zero performance degradation for core workflows without network
- **Scalability:** Support 100 plants per user, 10,000 care history entries, 25,000 concurrent users
- **Security:** Data encryption at rest and in transit (TLS 1.2+), 30-day session expiration
- **Privacy:** Minimal data collection, no third-party analytics in MVP

**Scale & Complexity:**

- **Primary domain:** Cross-platform mobile application (iOS 14+, Android 8.0+)
- **Complexity level:** Low-to-Medium
  - Low: Straightforward CRUD operations, simple scheduling logic, no complex business rules
  - Medium: Offline-first architecture, local notification reliability, cross-device sync with conflict resolution
- **Estimated architectural components:** 8-10 major components (UI layer, local storage, notification scheduler, sync engine, species database, care logic, authentication, API gateway)

### Technical Constraints & Dependencies

**Platform Constraints:**
- Cross-platform codebase (React Native or Flutter) targeting iOS and Android with feature parity
- Minimum OS versions: iOS 14+, Android 8.0+ (covers 95%+ of target market)
- Local notification scheduling (no server dependency for reliability)
- Offline-first architecture (core workflows must function without network)

**UX-Driven Constraints:**
- One-tap care logging from notification (minimal interaction friction)
- Haptic feedback for tactile confirmation
- Thumb-friendly touch targets for one-handed use
- Glanceable "Today" view showing only what needs attention

**Data Constraints:**
- Species database requires initial download and local caching
- Last-write-wins conflict resolution for multi-device scenarios (acceptable for single-user MVP)
- Local data persistence using device storage (SQLite or equivalent)

**Timeline Constraints:**
- 3-4 month timeline to pilot-ready MVP (50-100 early adopters)
- Ruthless MVP scope discipline (no photo uploads, no adaptive learning, no social features)

### Cross-Cutting Concerns Identified

**1. Offline-First Architecture**
Every core workflow (add plant, log care, view schedules, receive notifications) must function flawlessly without network connectivity. This affects data persistence, notification scheduling, and sync strategy across all components.

**2. Notification Reliability**
Local push notifications are the primary engagement mechanism. Notification delivery must be reliable across device states (background, killed, low battery) and OS-level restrictions (Do Not Disturb, battery optimization). Requires careful handling of background tasks and notification permissions.

**3. Data Synchronization**
Plant inventory and care history must sync across user's devices with transparent background sync. Conflict resolution strategy (last-write-wins for MVP) must be implemented consistently. Sync triggers include app launch, background refresh, and manual pull-to-refresh.

**4. Cross-Platform Consistency**
Single codebase must deliver consistent experience across iOS and Android while respecting platform conventions (iOS navigation vs Android back button, platform-specific gestures). UI library selection (React Native Paper or Flutter Material) impacts consistency and customization flexibility.

**5. Performance Optimization**
2-second app launch and 1-second user actions require careful optimization of startup sequence, lazy loading, and local data access patterns. Offline mode must have zero performance degradation compared to online mode.

**6. Security & Privacy**
Data encryption at rest and in transit, user authentication for sync, session management, and privacy-first data collection must be implemented consistently across all components.

### Architectural Risk Mitigation (Pre-mortem Analysis)

**Critical Risk Areas Identified:**

1. **Notification Reliability (HIGH RISK)**
   - **Failure scenario:** iOS background task limitations and Android battery optimization break local notifications; users disable notifications within first week; retention drops to 15%
   - **Root causes:** Background task handling not properly designed; no graceful degradation; notification timing too rigid
   - **Mitigation strategies:**
     - Design in-app "Today" view as primary interface, notifications as secondary reminder
     - Document iOS background modes and notification entitlements requirements
     - Specify Android foreground service strategy for critical notification scheduling
     - Implement notification scheduling with user timezone and "quiet hours" logic
     - Add notification delivery monitoring and user-facing notification history
     - Test notification reliability across device states (background, killed, low battery, Do Not Disturb)

2. **Offline Sync Conflicts (HIGH RISK)**
   - **Failure scenario:** Multi-device users experience data loss and sync conflicts; plants disappear; care history inconsistent; trust collapses
   - **Root causes:** Last-write-wins too naive; no conflict detection; sync timing unpredictable; silent data loss
   - **Mitigation strategies:**
     - Implement vector clocks or timestamp-based conflict detection (not just last-write-wins)
     - Add conflict resolution UI for critical data (plant inventory) vs auto-merge for append-only data (care history)
     - Design explicit sync status indicator and manual sync trigger
     - Add optimistic UI updates with rollback capability if sync fails
     - Implement sync queue with retry logic and exponential backoff
     - Log sync conflicts for debugging and user support

3. **Framework Choice Validation (MEDIUM RISK)**
   - **Failure scenario:** Framework chosen without prototyping critical features; 2 months into development, team hits blockers requiring costly rewrites
   - **Root causes:** No technical spike to validate notification reliability and offline storage performance; framework limitations underestimated
   - **Mitigation strategies:**
     - Require technical spike for notification reliability before framework decision
     - Prototype offline storage with realistic data volumes (100 plants, 10,000 care entries)
     - Document framework-specific limitations and workarounds for critical features
     - Evaluate native module requirements and maintenance burden
     - Build framework decision criteria matrix with weighted scoring

4. **Performance Degradation at Scale (MEDIUM RISK)**
   - **Failure scenario:** App launch time degrades to 5-8 seconds as users add plants; 2-second target missed by 3x; users abandon app
   - **Root causes:** No lazy loading strategy; species database not optimized; synchronous operations blocking UI thread
   - **Mitigation strategies:**
     - Design lazy loading strategy: load "Today" view data first, defer full plant list
     - Implement species database pagination and image lazy loading
     - Specify startup performance budget with measurement points
     - Document async/await patterns for all I/O operations
     - Add performance monitoring and alerting for key metrics
     - Design progressive rendering: show skeleton UI immediately, populate with data

5. **Species Database Extensibility (LOW RISK)**
   - **Failure scenario:** Hardcoded 50-plant species database becomes outdated; no update mechanism; users request unsupported plants; database stays frozen
   - **Root causes:** No update mechanism designed; species data hardcoded in app bundle; no versioning or migration strategy
   - **Mitigation strategies:**
     - Design species database as downloadable/updatable content (not hardcoded)
     - Implement database versioning and migration strategy
     - Add species request feature (user can submit missing plants)
     - Build admin API for species database updates without app releases
     - Cache species data locally but check for updates on app launch
     - Design fallback for custom/unknown species (user can create manual schedules)

### First Principles Architecture Foundation

**Core Truth: Plant care system is fundamentally a scheduling engine with memory**
- Architecture centers on deterministic scheduling logic: Plant + Schedule + History → "What needs care now?"
- All other layers (UI, notifications, sync) serve this core scheduling engine
- Scheduling logic must be pure functions, testable in isolation, with no side effects

**Reliability Hierarchy (Graceful Degradation):**
1. **Must work (offline, no notifications):** Local data persistence, schedule calculation, care logging, in-app "Today" view
2. **Should work (degraded experience):** Manual sync trigger, notification history fallback
3. **Nice to have (enhancement):** Push notifications, automatic background sync
4. **Future (post-MVP):** Adaptive learning, photo uploads, social features

**User-Centric Architecture Principles:**
- **One-tap primacy:** Most frequent action (log care from notification) should be simplest architecturally
- **Zero-config defaults:** Species selection auto-configures schedules without user input
- **Invisible infrastructure:** Sync, notifications, and storage happen transparently—users never think about "online vs offline"
- **Speed over richness:** Optimize for 2-second launch and 1-tap actions, not feature complexity

**Minimal Viable Architecture (5 Layers):**
1. **Core Scheduling Engine:** Pure logic calculating what needs care when (deterministic, testable)
2. **Local State Management:** SQLite persistence as single source of truth (encrypted at rest)
3. **UI Layer:** Minimal screens optimized for speed ("Today" view primary, <1 second load)
4. **Notification Layer:** Optional enhancement (local scheduling, graceful degradation if fails)
5. **Sync Layer:** Optional enhancement (REST API, optimistic updates, conflict resolution)

**Key Architectural Insight:**
The app should work perfectly offline with no notifications—that's the baseline. Notifications and sync are enhancements that make the baseline experience better, but they're not requirements for core functionality. This inverts the typical mobile app architecture where backend connectivity is assumed.


## Starter Template Evaluation

### Primary Technology Domain

**Cross-platform mobile application** using React Native with Expo, based on:
- Team experience with React Native and Expo
- Small team requiring managed workflow and simplified builds
- Critical requirement for local notification support
- Offline-first architecture needs

### Technical Preferences Confirmed

**Team & Experience:**
- Existing React Native experience
- Small team already using Expo
- Managed workflow preferred for faster iteration

**Critical Requirements:**
- Local notification capability (must-have)
- Offline-first architecture with SQLite
- Cross-platform (iOS + Android)
- 3-4 month timeline to pilot

### Starter Options Considered

**Option 1: Expo SDK 55 Default Template (RECOMMENDED)**
- Command: `npx create-expo-app@latest --template default@sdk-55`
- Includes: TypeScript, Expo Router v4 for file-based navigation, React Native 0.83
- Best for: Production apps with navigation structure and TypeScript from day 1

**Option 2: Expo Blank Template**
- Command: `npx create-expo-app@latest`
- Includes: Minimal setup, no navigation
- Best for: Maximum control, but requires more setup

**Option 3: Community Templates**
- Various community templates with NativeWind, Zustand, TanStack Query
- Best for: Specific tech stack preferences, but adds complexity

### Selected Starter: Expo SDK 55 Default Template

**Rationale for Selection:**

1. **TypeScript by default** - Type safety critical for offline-first architecture with complex state management
2. **Expo Router v4 included** - File-based navigation matches "Today" view + Plant List + Plant Detail screen structure
3. **React Native 0.83** - Latest stable with New Architecture support and Hermes V1 for performance
4. **Managed workflow** - EAS Build handles iOS/Android builds without Xcode/Android Studio setup
5. **Production-ready structure** - Organized file structure, example code, basic navigation scaffold
6. **Team familiarity** - Aligns with existing Expo experience, minimal learning curve

**Initialization Command:**

```bash
npx create-expo-app@latest plant-care-tracker --template default@sdk-55
```

**Architectural Decisions Provided by Starter:**

**Language & Runtime:**
- TypeScript configured with strict mode
- React Native 0.83 with Hermes JavaScript engine (V1)
- New Architecture enabled by default in SDK 55
- Node.js 18+ required for development

**Navigation Solution:**
- Expo Router v4 (file-based routing)
- Native navigation with Stack and Tabs support
- Deep linking configured out of the box
- Web support included (bonus for future web companion app)

**Build Tooling:**
- Metro bundler with optimized caching
- EAS Build for cloud-based iOS/Android builds
- Fast Refresh for instant development feedback
- TypeScript compilation integrated

**Project Structure:**
```
plant-care-tracker/
├── app/              # Expo Router file-based routes
│   ├── (tabs)/       # Tab navigation screens
│   ├── _layout.tsx   # Root layout
│   └── +not-found.tsx
├── assets/           # Images, fonts, etc.
├── components/       # Reusable React components
├── constants/        # App constants and config
├── hooks/            # Custom React hooks
├── scripts/          # Build and utility scripts
├── app.json          # Expo configuration
├── package.json      # Dependencies
└── tsconfig.json     # TypeScript configuration
```

**Development Experience:**
- Hot reloading with Fast Refresh
- TypeScript IntelliSense and type checking
- Expo Go for rapid testing on physical devices
- EAS Update for over-the-air updates (post-MVP)

**Key Expo SDK Packages Available:**
- `expo-notifications` - Local and push notifications (critical for our use case)
- `expo-sqlite` - SQLite database for offline storage
- `expo-secure-store` - Encrypted storage for sensitive data
- `expo-haptics` - Haptic feedback for care logging confirmation
- `expo-background-task` - Background task scheduling (SDK 53+)
- `expo-task-manager` - Background notification handling

**What's NOT Included (We'll Add):**
- SQLite database setup and schema
- Local notification scheduling logic
- Offline-first state management (we'll evaluate options)
- Species database and care scheduling engine
- Authentication and sync infrastructure

**Note:** Project initialization using this command should be Epic 1, Story 1 in the implementation plan.

### Framework Decision: React Native with Expo (Confirmed)

**Decision Rationale:**
- Team has existing React Native and Expo experience
- Managed workflow reduces complexity for small team
- Expo SDK 55 provides all critical features needed:
  - Local notifications via `expo-notifications`
  - Offline storage via `expo-sqlite`
  - Background tasks via `expo-background-task`
  - Haptic feedback via `expo-haptics`
- EAS Build handles cross-platform builds without native tooling
- 3-4 month timeline achievable with Expo's rapid development workflow

**Trade-offs Accepted:**
- Slightly larger app bundle size vs bare React Native (acceptable for MVP)
- Dependency on Expo SDK updates (mitigated by active maintenance)
- Some native modules may require custom development config (rare for our use case)

**Validation Against Pre-mortem Risks:**
- ✅ Notification reliability: Expo's `expo-notifications` is mature and well-tested
- ✅ Offline storage: `expo-sqlite` provides robust SQLite support
- ✅ Performance: React Native 0.83 with Hermes V1 meets 2-second launch target
- ✅ Framework choice: Team experience eliminates learning curve risk

