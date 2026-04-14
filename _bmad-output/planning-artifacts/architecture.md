---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments: ['_bmad-output/planning-artifacts/prd.md', '_bmad-output/planning-artifacts/ux-design-specification.md']
workflowType: 'architecture'
project_name: 'plant-care-tracker'
user_name: 'Jared'
date: '2026-04-02'
lastStep: 8
status: 'complete'
completedAt: '2026-04-14'
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



## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- State management approach (React Context + Custom Hooks)
- Database schema strategy (Raw SQL with manual migrations)
- Backend & sync architecture (Custom Node.js API)
- Authentication method (JWT with Email/Password)
- Notification scheduling strategy (Hybrid: 7 days ahead + daily refresh)
- API design patterns (Resource-based REST)
- Error handling standards (Structured error responses)

**Important Decisions (Shape Architecture):**
- Hosting strategy (Serverless functions)
- Monitoring approach (Cloud provider logs)
- CI/CD pipeline (Manual deployment for MVP)

**Deferred Decisions (Post-MVP):**
- Advanced monitoring and observability (can add Sentry/LogRocket later)
- Automated CI/CD pipeline (can add GitHub Actions after pilot)
- Multi-region deployment (single region sufficient for pilot)
- Advanced caching strategies (implement if performance issues arise)

### Frontend Architecture

**State Management: React Context + Custom Hooks**
- **Decision:** Use React's built-in Context API with custom hooks for state management
- **Rationale:** Keeps dependencies minimal, sufficient for app's complexity, team already familiar with React patterns
- **Implementation approach:**
  - Create context providers for: Plants, Care Schedules, Sync Status, User Auth
  - Custom hooks for common operations: `usePlants()`, `useCareSchedule()`, `useSync()`
  - Local state with `useState` for component-specific UI state
- **Affects:** Component architecture, data flow patterns, testing strategy
- **Trade-offs accepted:** More boilerplate than Zustand/Redux, but simpler mental model and zero dependencies

**Component Architecture: Provided by Expo Router v4**
- **Decision:** File-based routing with Expo Router v4 (from starter template)
- **Structure:**
  ```
  app/
  ├── (tabs)/
  │   ├── index.tsx          # "Today" view (primary screen)
  │   ├── plants.tsx         # Plant list
  │   └── settings.tsx       # Settings
  ├── plant/[id].tsx         # Plant detail screen
  ├── care-log/[id].tsx      # Care logging screen
  └── _layout.tsx            # Root layout with providers
  ```
- **Rationale:** Matches app's navigation structure, deep linking built-in, web support as bonus
- **Affects:** Navigation patterns, screen organization, deep linking implementation

**Performance Optimization Strategy:**
- **Lazy loading:** Load "Today" view data first, defer full plant list until needed
- **Progressive rendering:** Show skeleton UI immediately, populate with data asynchronously
- **Image optimization:** Lazy load species images, use appropriate sizes for device
- **Startup budget:** 2-second app launch target with measurement points at key stages
- **Affects:** Initial load sequence, bundle size, user experience

### Data Architecture

**Local Database: expo-sqlite (SQLite)**
- **Decision:** Use `expo-sqlite` for local data persistence (from starter template decision)
- **Version:** expo-sqlite ^14.0.0 (Expo SDK 55)
- **Rationale:** Offline-first requirement, mature library, encrypted at rest via `expo-secure-store` for sensitive data
- **Schema approach:** Raw SQL with manual migrations
- **Migration strategy:**
  - Version-tracked migrations in `src/database/migrations/`
  - Run migrations on app startup, check version table
  - No ORM overhead, full control over queries
- **Affects:** Data persistence, offline functionality, sync conflict resolution
- **Trade-offs accepted:** More manual work than ORM, but better performance and control

**Server Database: Managed PostgreSQL**
- **Decision:** Use managed PostgreSQL service (AWS RDS, Supabase, or PlanetScale)
- **Rationale:** Serverless hosting requires managed database, PostgreSQL is reliable and well-supported
- **Recommended options:**
  - AWS RDS PostgreSQL (if using AWS Lambda)
  - Supabase PostgreSQL (includes auth, good for MVP)
  - PlanetScale (MySQL-compatible, excellent developer experience)
- **Affects:** Sync API implementation, data modeling, connection pooling strategy
- **Note:** Mobile app uses SQLite locally, server uses PostgreSQL for sync—standard serverless pattern

**Data Validation Strategy:**
- **Client-side:** TypeScript types + runtime validation before SQLite writes
- **Server-side:** Request validation middleware with structured error responses
- **Sync validation:** Timestamp-based conflict detection, last-write-wins with user notification
- **Affects:** Error handling, data integrity, sync reliability

### Authentication & Security

**Authentication Method: JWT with Email/Password**
- **Decision:** JWT token-based authentication with email/password credentials
- **Rationale:** Full control, no external dependencies, matches custom API strategy
- **Implementation approach:**
  - User registration with email verification
  - Password hashing with bcrypt (cost factor 12)
  - JWT tokens with 30-day expiration (refresh token pattern)
  - Store JWT in `expo-secure-store` on device
- **Affects:** User registration flow, login screen, API authentication middleware, session management
- **Trade-offs accepted:** Need to implement password reset flow, no OAuth convenience (can add later)

**Data Encryption:**
- **At rest:** SQLite database encrypted via `expo-secure-store` for sensitive data (JWT tokens, user credentials)
- **In transit:** TLS 1.2+ for all API communication (enforced by serverless platform)
- **Affects:** Security implementation, compliance requirements

**API Security:**
- **Authentication:** JWT bearer tokens in Authorization header
- **Rate limiting:** Basic rate limiting at API gateway level (serverless platform feature)
- **Input validation:** Validate all inputs, sanitize before database operations
- **Affects:** API middleware, error handling, security testing

### API & Communication Patterns

**API Design: Resource-Based REST**
- **Decision:** Standard REST endpoints with HTTP methods
- **Version:** Express.js v5.2 (latest stable as of 2026)
- **Endpoint structure:**
  ```
  GET    /api/plants              # List user's plants
  POST   /api/plants              # Create new plant
  GET    /api/plants/:id          # Get plant details
  PUT    /api/plants/:id          # Update plant
  DELETE /api/plants/:id          # Delete plant
  
  GET    /api/care-logs           # List care logs
  POST   /api/care-logs           # Log care action
  
  GET    /api/species             # List species database
  GET    /api/species/:id         # Get species details
  
  POST   /api/auth/register       # User registration
  POST   /api/auth/login          # User login
  POST   /api/auth/refresh        # Refresh JWT token
  ```
- **Rationale:** Simple, predictable, widely understood, easy to test
- **Affects:** Client-side API layer, backend routing, API documentation

**Error Handling: Structured Error Responses**
- **Decision:** Consistent error format with error codes, messages, and field references
- **Response format:**
  ```json
  {
    "error": {
      "code": "PLANT_NOT_FOUND",
      "message": "Plant with ID 123 not found",
      "field": "plantId",
      "timestamp": "2026-04-14T10:30:00Z"
    }
  }
  ```
- **HTTP status codes:** Standard codes (200, 201, 400, 401, 404, 409, 500)
- **Error code categories:**
  - `VALIDATION_ERROR` - Input validation failures
  - `AUTH_ERROR` - Authentication/authorization failures
  - `NOT_FOUND` - Resource not found
  - `CONFLICT` - Sync conflicts, duplicate resources
  - `SERVER_ERROR` - Internal server errors
- **Rationale:** Enables specific client-side error handling, better UX, helps with sync conflict resolution
- **Affects:** API implementation, client error handling, sync conflict UI

**Sync Strategy:**
- **Approach:** Optimistic updates with background sync
- **Conflict resolution:** Timestamp-based detection with last-write-wins, user notification for conflicts
- **Sync triggers:** App launch, background refresh (daily), manual pull-to-refresh
- **Sync queue:** Retry logic with exponential backoff for failed syncs
- **Affects:** Offline functionality, data consistency, user experience

### Backend & Sync Architecture

**Backend Framework: Custom Node.js API with Express.js**
- **Decision:** Build custom REST API using Express.js v5.2
- **Version:** Express.js v5.2.1 (latest stable, 2026)
- **Rationale:** Full control over sync logic, simple REST endpoints, no vendor lock-in, team can customize as needed
- **Key features:**
  - JWT authentication middleware
  - Structured error handling middleware
  - Request validation middleware
  - Database connection pooling
  - Sync conflict detection logic
- **Affects:** All backend implementation, sync engine, API endpoints, authentication flow
- **Trade-offs accepted:** More implementation work than BaaS, but complete control and flexibility

### Notification Architecture

**Notification Scheduling: Hybrid Strategy**
- **Decision:** Schedule 7 days ahead + daily background refresh
- **Implementation approach:**
  - On app launch: Schedule next 7 days of notifications (stays under iOS 64-notification limit)
  - Daily background task: Refresh notification schedule, adjust for logged care actions
  - On care logged: Immediately reschedule affected plant's notifications
  - Notification content: Plant name, care action, custom message
- **Background task:** Use `expo-background-task` for daily refresh (requires background permissions)
- **Graceful degradation:** In-app "Today" view as primary interface, notifications as secondary reminder
- **Rationale:** Maximum reliability (works when app killed) + dynamic adjustment (respects user actions)
- **Affects:** Background task implementation, notification scheduler, battery optimization handling, iOS/Android permission flows
- **Trade-offs accepted:** More complex than single strategy, but provides best user experience

**Notification Delivery Monitoring:**
- **Track delivery:** Log scheduled vs delivered notifications
- **Fallback:** In-app notification history for missed notifications
- **User controls:** Quiet hours, notification timing preferences, per-plant notification toggle
- **Affects:** Notification reliability, user settings, debugging workflow

### Infrastructure & Deployment

**Hosting: Serverless Functions**
- **Decision:** Deploy API as serverless functions (AWS Lambda or Vercel Functions)
- **Rationale:** Pay-per-request pricing minimizes costs for pilot phase, auto-scaling handles growth, zero server management
- **Recommended platforms:**
  - **AWS Lambda** with API Gateway (if using AWS RDS for database)
  - **Vercel Functions** (simpler deployment, good developer experience)
- **Cold start optimization:**
  - Keep functions warm with periodic health checks (if needed)
  - Minimize dependencies to reduce cold start time
  - Use connection pooling for database (RDS Proxy or Supabase connection pooler)
- **Affects:** API architecture (stateless functions), database strategy (managed PostgreSQL required), deployment process
- **Trade-offs accepted:** Cold start latency (typically 100-500ms), stateless architecture required

**CI/CD: Manual Deployment for MVP**
- **Decision:** Manual deployment scripts for pilot phase
- **Deployment process:**
  - Run tests locally before deploy
  - Deploy via CLI commands (`vercel deploy` or `aws lambda update-function-code`)
  - Manual database migrations
  - Environment variables managed via platform dashboard
- **Rationale:** Simplest approach for MVP, can add GitHub Actions automation after pilot validates product
- **Affects:** Deployment workflow, testing process, release cadence
- **Future enhancement:** Add GitHub Actions for automated testing and deployment post-pilot

**Environment Configuration:**
- **Environments:** Development (local), Staging (optional), Production
- **Configuration management:** Environment variables via platform (Vercel/AWS)
- **Secrets:** Database credentials, JWT secret, API keys stored in platform secret manager
- **Affects:** Configuration management, security, deployment process

**Monitoring & Logging: Cloud Provider Logs**
- **Decision:** Use console.log with cloud provider logging (AWS CloudWatch or Vercel logs)
- **Rationale:** Zero cost, built-in, sufficient for pilot phase with 50-100 users
- **Logging strategy:**
  - Structured console.log with JSON format for easier parsing
  - Log levels: ERROR, WARN, INFO, DEBUG
  - Include request ID, user ID, timestamp in all logs
  - Monitor error rates and API latency via platform dashboards
- **Affects:** Error handling implementation, debugging workflow, incident response
- **Future enhancement:** Add Sentry for error tracking if error rates become difficult to manage

### Decision Impact Analysis

**Implementation Sequence:**

1. **Foundation (Epic 1):**
   - Initialize Expo project with SDK 55 template
   - Set up SQLite database with schema and migrations
   - Implement React Context providers for state management
   - Create basic navigation structure with Expo Router

2. **Authentication (Epic 2):**
   - Build JWT authentication API (Express.js serverless functions)
   - Set up managed PostgreSQL database
   - Implement registration and login flows
   - Secure token storage with expo-secure-store

3. **Core Features (Epic 3-5):**
   - Plant inventory CRUD with local SQLite
   - Species database integration
   - Care scheduling engine (pure functions)
   - Care logging with history

4. **Notifications (Epic 6):**
   - Implement hybrid notification scheduling
   - Set up daily background refresh task
   - Build notification permission flows
   - Add in-app notification history fallback

5. **Sync (Epic 7):**
   - Build sync API endpoints (REST)
   - Implement optimistic updates with rollback
   - Add conflict detection and resolution
   - Create sync status UI

6. **Deployment (Epic 8):**
   - Deploy serverless functions (Vercel or AWS Lambda)
   - Configure managed PostgreSQL database
   - Set up environment variables and secrets
   - Manual deployment scripts

**Cross-Component Dependencies:**

- **State Management ↔ Sync:** React Context providers must handle optimistic updates and rollback for sync failures
- **SQLite ↔ Sync API:** Local database schema must match server schema for seamless sync
- **Notifications ↔ Background Tasks:** Daily refresh requires background execution permissions and proper task scheduling
- **Authentication ↔ All API Calls:** JWT tokens required for all authenticated endpoints, must handle token refresh
- **Serverless Functions ↔ Database:** Connection pooling critical for serverless (use RDS Proxy or Supabase pooler)
- **Error Handling ↔ Sync:** Structured error responses enable proper conflict resolution UI
- **Offline-First ↔ All Features:** Every feature must work offline first, sync as enhancement

**Technology Stack Summary:**

**Frontend:**
- React Native 0.83 with Expo SDK 55
- TypeScript (strict mode)
- Expo Router v4 (file-based navigation)
- React Context + Custom Hooks (state management)
- expo-sqlite (local database)
- expo-notifications (local push notifications)
- expo-background-task (daily refresh)
- expo-secure-store (encrypted storage)
- expo-haptics (tactile feedback)

**Backend:**
- Node.js 18+ with Express.js v5.2
- Serverless functions (AWS Lambda or Vercel Functions)
- Managed PostgreSQL (AWS RDS, Supabase, or PlanetScale)
- JWT authentication with bcrypt password hashing
- Structured error responses with error codes

**Infrastructure:**
- Serverless hosting (AWS Lambda or Vercel)
- Managed PostgreSQL database
- Cloud provider logging (CloudWatch or Vercel logs)
- Manual deployment for MVP
- TLS 1.2+ for all API communication


## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical Conflict Points Identified:** 9 areas where AI agents could make different implementation choices

These patterns ensure all AI agents working on this project write compatible, consistent code that works together seamlessly.

### Naming Patterns

**Database Naming Conventions:**
- **Tables:** camelCase, plural for collections: `plants`, `careLogs`, `species`, `users`
- **Columns:** camelCase: `plantId`, `userId`, `createdAt`, `updatedAt`, `careType`
- **Primary keys:** `id` (integer, auto-increment)
- **Foreign keys:** `{resource}Id` format: `plantId`, `userId`, `speciesId`
- **Timestamps:** `createdAt`, `updatedAt` (ISO 8601 strings)
- **Boolean columns:** `is{Condition}` format: `isActive`, `isPaused`, `isDeleted`

**Examples:**
```sql
CREATE TABLE plants (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  userId INTEGER NOT NULL,
  speciesId INTEGER,
  name TEXT NOT NULL,
  nickname TEXT,
  createdAt TEXT NOT NULL,
  updatedAt TEXT NOT NULL
);

CREATE TABLE careLogs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  plantId INTEGER NOT NULL,
  careType TEXT NOT NULL,
  loggedAt TEXT NOT NULL,
  notes TEXT
);
```

**API Naming Conventions:**
- **Endpoints:** Plural resource names: `/api/plants`, `/api/care-logs`, `/api/species`
- **Route parameters:** `:id` format: `/api/plants/:id`, `/api/care-logs/:id`
- **Query parameters:** camelCase: `?userId=123&includeDeleted=false`
- **HTTP methods:** Standard REST verbs (GET, POST, PUT, DELETE)
- **Multi-word resources:** kebab-case in URL: `/api/care-logs`, camelCase in code: `careLogs`

**Examples:**
```
GET    /api/plants              # List user's plants
POST   /api/plants              # Create new plant
GET    /api/plants/:id          # Get plant details
PUT    /api/plants/:id          # Update plant
DELETE /api/plants/:id          # Delete plant

GET    /api/care-logs           # List care logs
POST   /api/care-logs           # Log care action
GET    /api/care-logs/:id       # Get care log details

GET    /api/species             # List species
GET    /api/species/:id         # Get species details
```

**Code Naming Conventions:**
- **Variables:** camelCase: `plantId`, `userId`, `createdAt`, `careSchedule`
- **Functions:** camelCase verbs: `getUserPlants()`, `logCareAction()`, `syncData()`
- **Components:** PascalCase in code: `PlantCard`, `TodayView`, `CareLogItem`
- **Component files:** kebab-case: `plant-card.tsx`, `today-view.tsx`, `care-log-item.tsx`
- **Utility files:** kebab-case: `date-utils.ts`, `sync-engine.ts`, `notification-scheduler.ts`
- **Constants:** UPPER_SNAKE_CASE: `MAX_PLANTS`, `DEFAULT_CARE_INTERVAL`, `API_BASE_URL`
- **Types/Interfaces:** PascalCase: `Plant`, `CareLog`, `Species`, `SyncStatus`
- **Enums:** PascalCase with UPPER_CASE values: `CareType.WATER`, `SyncStatus.SYNCING`

**Examples:**
```typescript
// Variables and functions
const plantId = 123;
const userPlants = await getUserPlants(userId);
const careSchedule = calculateCareSchedule(plant);

// Constants
const MAX_PLANTS_PER_USER = 100;
const API_BASE_URL = 'https://api.example.com';

// Types
interface Plant {
  id: number;
  userId: number;
  name: string;
  createdAt: string;
}

// Enums
enum CareType {
  WATER = 'WATER',
  FERTILIZE = 'FERTILIZE',
  PRUNE = 'PRUNE'
}

// Component (in plant-card.tsx file)
export default function PlantCard({ plant }: { plant: Plant }) {
  return <View>...</View>;
}
```

### Structure Patterns

**Project Organization:**
- **Source code:** `src/` directory for all application code
- **Tests:** `tests/` directory mirroring source structure
  - Source: `src/components/plant-card.tsx`
  - Test: `tests/components/plant-card.test.tsx`
- **Components:** Organized by feature in `src/components/{feature}/`
  - `src/components/plants/` - Plant-related components
  - `src/components/care/` - Care logging components
  - `src/components/common/` - Shared components
- **Screens:** Expo Router file-based routing in `app/` directory
- **Business logic:** `src/services/` for API calls, `src/lib/` for utilities
- **Database:** `src/database/` for schema, migrations, queries
- **State management:** `src/contexts/` for React Context providers
- **Types:** `src/types/` for shared TypeScript types

**File Structure Patterns:**
```
plant-care-tracker/
├── app/                          # Expo Router screens
│   ├── (tabs)/
│   │   ├── index.tsx            # Today view
│   │   ├── plants.tsx           # Plant list
│   │   └── settings.tsx         # Settings
│   ├── plant/[id].tsx           # Plant detail
│   └── _layout.tsx              # Root layout
├── src/
│   ├── components/              # React components
│   │   ├── plants/
│   │   │   ├── plant-card.tsx
│   │   │   └── plant-list.tsx
│   │   ├── care/
│   │   │   ├── care-log-item.tsx
│   │   │   └── care-button.tsx
│   │   └── common/
│   │       ├── button.tsx
│   │       └── loading-spinner.tsx
│   ├── contexts/                # React Context providers
│   │   ├── plants-context.tsx
│   │   ├── auth-context.tsx
│   │   └── sync-context.tsx
│   ├── database/                # Database layer
│   │   ├── schema.ts
│   │   ├── migrations/
│   │   │   ├── 001-initial.ts
│   │   │   └── 002-add-species.ts
│   │   └── queries/
│   │       ├── plants.ts
│   │       └── care-logs.ts
│   ├── services/                # API services
│   │   ├── api-client.ts
│   │   ├── plants-service.ts
│   │   └── sync-service.ts
│   ├── lib/                     # Utilities
│   │   ├── date-utils.ts
│   │   ├── notification-scheduler.ts
│   │   └── sync-engine.ts
│   ├── types/                   # TypeScript types
│   │   ├── plant.ts
│   │   ├── care-log.ts
│   │   └── api.ts
│   └── hooks/                   # Custom React hooks
│       ├── use-plants.ts
│       └── use-sync.ts
├── tests/                       # Test files (mirrors src/)
│   ├── components/
│   │   └── plants/
│   │       └── plant-card.test.tsx
│   ├── services/
│   │   └── plants-service.test.tsx
│   └── lib/
│       └── date-utils.test.tsx
├── assets/                      # Static assets
│   ├── images/
│   └── fonts/
└── constants/                   # App constants
    └── config.ts
```

**Configuration Files:**
- **Root level:** `package.json`, `tsconfig.json`, `app.json`, `.env`
- **Environment files:** `.env.local`, `.env.production` (not committed)
- **Database migrations:** Numbered sequentially: `001-initial.ts`, `002-add-species.ts`

### Format Patterns

**API Response Formats:**

**Success Response:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Monstera",
    "userId": 123,
    "createdAt": "2026-04-14T10:30:00Z"
  }
}
```

**Success Response (List):**
```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Monstera" },
    { "id": 2, "name": "Pothos" }
  ]
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "PLANT_NOT_FOUND",
    "message": "Plant with ID 123 not found",
    "field": "plantId",
    "timestamp": "2026-04-14T10:30:00Z"
  }
}
```

**Data Exchange Formats:**
- **JSON field naming:** camelCase: `plantId`, `userId`, `createdAt`
- **Date/time format:** ISO 8601 strings with UTC timezone: `"2026-04-14T10:30:00Z"`
- **Boolean values:** `true`/`false` (not `1`/`0` or `"true"`/`"false"`)
- **Null handling:** Use `null` for missing optional values, omit fields for undefined
- **Arrays:** Always return arrays for collections, even if empty: `[]`
- **Numbers:** Use numbers for IDs and counts, not strings: `123` not `"123"`

**Examples:**
```json
{
  "success": true,
  "data": {
    "id": 1,
    "userId": 123,
    "name": "Monstera",
    "nickname": null,
    "isPaused": false,
    "careSchedule": {
      "waterIntervalDays": 7,
      "fertilizeIntervalDays": 30
    },
    "lastWatered": "2026-04-10T08:00:00Z",
    "createdAt": "2026-04-01T10:30:00Z",
    "updatedAt": "2026-04-14T10:30:00Z"
  }
}
```

### Communication Patterns

**State Management Patterns:**
- **State updates:** Immutable updates using spread operator
  ```typescript
  // Array updates
  setPlants([...plants, newPlant])
  setPlants(plants.filter(p => p.id !== deletedId))
  setPlants(plants.map(p => p.id === updatedId ? updatedPlant : p))
  
  // Object updates
  setState({ ...state, loading: false })
  setPlant({ ...plant, name: newName })
  ```
- **Context providers:** One provider per domain: `PlantsProvider`, `AuthProvider`, `SyncProvider`
- **Custom hooks:** Expose context via hooks: `usePlants()`, `useAuth()`, `useSync()`
- **Hook naming:** `use{Domain}` format: `usePlants()`, `useCareSchedule()`, `useNotifications()`

**Sync Event Patterns:**
- **Event naming:** camelCase with domain prefix: `sync.started`, `sync.completed`, `sync.failed`
- **Event payload:** Consistent structure with `type` and `data` fields
  ```typescript
  {
    type: 'sync.completed',
    data: {
      syncedAt: '2026-04-14T10:30:00Z',
      itemsSynced: 5,
      conflicts: []
    }
  }
  ```

### Process Patterns

**Error Handling Patterns:**
- **Async operations:** Wrap in try-catch blocks
  ```typescript
  try {
    const plants = await syncPlants();
    setPlants(plants);
  } catch (error) {
    handleSyncError(error);
    showErrorToast('Failed to sync plants');
  }
  ```
- **React Error Boundaries:** Wrap major sections (tabs, screens) to catch render errors
- **Error logging:** Log errors with context (user ID, operation, timestamp)
- **User-facing errors:** Show friendly messages, hide technical details
- **API errors:** Parse structured error responses and show appropriate messages

**Loading State Patterns:**
- **Boolean flags:** Use descriptive boolean flags per operation
  ```typescript
  const [isLoading, setIsLoading] = useState(false);
  const [isSyncing, setIsSyncing] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  ```
- **Naming convention:** `is` + present participle: `isLoading`, `isSyncing`, `isSaving`
- **Loading UI:** Show loading indicators for operations > 300ms
- **Optimistic updates:** Update UI immediately, rollback on error
  ```typescript
  // Optimistic update
  setPlants([...plants, newPlant]);
  try {
    await createPlant(newPlant);
  } catch (error) {
    // Rollback on error
    setPlants(plants.filter(p => p.id !== newPlant.id));
    showErrorToast('Failed to create plant');
  }
  ```

**Async Operation Patterns:**
- **Always use async/await:** Never use `.then()` chains
- **Error handling:** Always wrap async operations in try-catch
- **Loading states:** Set loading flag before operation, clear after completion or error
- **Cancellation:** Use AbortController for cancellable requests
  ```typescript
  const abortController = new AbortController();
  try {
    setIsLoading(true);
    const data = await fetchData({ signal: abortController.signal });
    setData(data);
  } catch (error) {
    if (error.name !== 'AbortError') {
      handleError(error);
    }
  } finally {
    setIsLoading(false);
  }
  ```

### Enforcement Guidelines

**All AI Agents MUST:**

1. **Follow naming conventions exactly:**
   - Database: camelCase tables and columns
   - API: Plural resource names, camelCase parameters
   - Files: kebab-case for all files
   - Code: camelCase variables/functions, PascalCase types/components
   - Constants: UPPER_SNAKE_CASE

2. **Use consistent file structure:**
   - Source code in `src/` directory
   - Tests in `tests/` directory mirroring source structure
   - Components organized by feature
   - Database code in `src/database/`

3. **Follow API response format:**
   - Always wrap responses with `{ success: true/false, data/error: ... }`
   - Use structured error responses with code, message, field
   - Use ISO 8601 date strings
   - Use camelCase for JSON fields

4. **Implement consistent patterns:**
   - Immutable state updates with spread operator
   - Try-catch for all async operations
   - Boolean loading flags with `is{Action}` naming
   - React Error Boundaries for component errors

5. **Maintain code quality:**
   - TypeScript strict mode enabled
   - No `any` types without explicit justification
   - Async/await instead of promise chains
   - Descriptive variable and function names

**Pattern Enforcement:**
- **Code reviews:** Check for pattern violations before merging
- **Linting:** Configure ESLint rules to enforce naming conventions
- **Type checking:** TypeScript strict mode catches many pattern violations
- **Testing:** Tests should follow same patterns as source code
- **Documentation:** Update this document when patterns evolve

### Pattern Examples

**Good Examples:**

```typescript
// ✅ Good: Follows all naming conventions
interface Plant {
  id: number;
  userId: number;
  name: string;
  createdAt: string;
}

async function getUserPlants(userId: number): Promise<Plant[]> {
  try {
    const response = await fetch(`/api/plants?userId=${userId}`);
    const json = await response.json();
    
    if (!json.success) {
      throw new Error(json.error.message);
    }
    
    return json.data;
  } catch (error) {
    console.error('Failed to fetch plants:', error);
    throw error;
  }
}

// ✅ Good: Immutable state update
function addPlant(newPlant: Plant) {
  setPlants([...plants, newPlant]);
}

// ✅ Good: Loading state pattern
const [isLoading, setIsLoading] = useState(false);

async function loadPlants() {
  try {
    setIsLoading(true);
    const plants = await getUserPlants(userId);
    setPlants(plants);
  } catch (error) {
    handleError(error);
  } finally {
    setIsLoading(false);
  }
}
```

**Anti-Patterns (Avoid These):**

```typescript
// ❌ Bad: snake_case in TypeScript
interface plant_data {
  plant_id: number;
  user_id: number;
}

// ❌ Bad: Promise chains instead of async/await
function getPlants() {
  fetch('/api/plants')
    .then(res => res.json())
    .then(data => setPlants(data))
    .catch(err => console.error(err));
}

// ❌ Bad: Direct mutation
function addPlant(newPlant: Plant) {
  plants.push(newPlant); // Mutates array directly
}

// ❌ Bad: Generic loading flag
const [loading, setLoading] = useState(false); // Not descriptive

// ❌ Bad: No error handling
async function loadPlants() {
  const plants = await getUserPlants(userId); // No try-catch
  setPlants(plants);
}

// ❌ Bad: Inconsistent API response
return { plants: [...] }; // Should be { success: true, data: [...] }

// ❌ Bad: Wrong date format
const createdAt = Date.now(); // Should be ISO 8601 string
```

### Pattern Validation Checklist

Before implementing any feature, AI agents should verify:

- [ ] All names follow camelCase/kebab-case/PascalCase conventions
- [ ] Files are in correct directories (`src/`, `tests/`)
- [ ] API responses use `{ success, data/error }` format
- [ ] Dates are ISO 8601 strings
- [ ] State updates are immutable
- [ ] Async operations have try-catch blocks
- [ ] Loading states use `is{Action}` naming
- [ ] TypeScript types are properly defined
- [ ] No `any` types without justification
- [ ] Tests mirror source structure in `tests/` directory


## Project Structure & Boundaries

### Complete Project Directory Structure

**Mobile Application (React Native + Expo):**

```
plant-care-tracker/
├── README.md
├── package.json
├── tsconfig.json
├── app.json                          # Expo configuration
├── eas.json                          # EAS Build configuration
├── .env.local                        # Local environment variables
├── .env.example                      # Environment template
├── .gitignore
├── .eslintrc.js
├── babel.config.js
├── metro.config.js
│
├── app/                              # Expo Router screens
│   ├── _layout.tsx                   # Root layout with providers
│   ├── +not-found.tsx                # 404 screen
│   │
│   ├── (tabs)/                       # Tab navigation
│   │   ├── _layout.tsx               # Tab layout
│   │   ├── index.tsx                 # Today view (primary screen)
│   │   ├── plants.tsx                # Plant list screen
│   │   └── settings.tsx              # Settings screen
│   │
│   ├── onboarding/                   # Onboarding flow
│   │   ├── _layout.tsx               # Onboarding layout
│   │   ├── welcome.tsx               # Welcome screen
│   │   ├── permissions.tsx           # Notification permissions
│   │   └── complete.tsx              # Onboarding complete
│   │
│   ├── auth/                         # Authentication screens
│   │   ├── login.tsx                 # Login screen
│   │   ├── register.tsx              # Registration screen
│   │   └── forgot-password.tsx       # Password reset
│   │
│   ├── plant/                        # Plant-related screens
│   │   ├── [id].tsx                  # Plant detail screen
│   │   ├── add.tsx                   # Add new plant
│   │   └── edit/[id].tsx             # Edit plant
│   │
│   └── care/                         # Care-related screens
│       ├── log/[plantId].tsx         # Log care action
│       └── history/[plantId].tsx     # Care history
│
├── src/
│   ├── components/                   # React components
│   │   ├── plants/
│   │   │   ├── plant-card.tsx        # Plant card component
│   │   │   ├── plant-list.tsx        # Plant list component
│   │   │   ├── plant-form.tsx        # Plant form component
│   │   │   └── species-selector.tsx  # Species selection component
│   │   │
│   │   ├── care/
│   │   │   ├── care-button.tsx       # Quick care action button
│   │   │   ├── care-log-item.tsx     # Care log list item
│   │   │   ├── care-schedule.tsx     # Care schedule display
│   │   │   └── today-task-card.tsx   # Today's task card
│   │   │
│   │   ├── notifications/
│   │   │   ├── notification-settings.tsx  # Notification preferences
│   │   │   └── notification-history.tsx   # Notification history
│   │   │
│   │   ├── onboarding/
│   │   │   ├── onboarding-step.tsx   # Onboarding step component
│   │   │   └── permission-prompt.tsx # Permission request component
│   │   │
│   │   └── common/
│   │       ├── button.tsx            # Reusable button
│   │       ├── input.tsx             # Reusable input
│   │       ├── loading-spinner.tsx   # Loading indicator
│   │       ├── error-boundary.tsx    # Error boundary component
│   │       └── empty-state.tsx       # Empty state component
│   │
│   ├── contexts/                     # React Context providers
│   │   ├── plants-context.tsx        # Plants state management
│   │   ├── auth-context.tsx          # Authentication state
│   │   ├── sync-context.tsx          # Sync state management
│   │   └── notifications-context.tsx # Notification state
│   │
│   ├── hooks/                        # Custom React hooks
│   │   ├── use-plants.ts             # Plants hook
│   │   ├── use-auth.ts               # Authentication hook
│   │   ├── use-sync.ts               # Sync hook
│   │   ├── use-notifications.ts      # Notifications hook
│   │   └── use-care-schedule.ts      # Care schedule hook
│   │
│   ├── database/                     # SQLite database layer
│   │   ├── index.ts                  # Database initialization
│   │   ├── schema.ts                 # Database schema definitions
│   │   │
│   │   ├── migrations/               # Database migrations
│   │   │   ├── 001-initial-schema.ts
│   │   │   ├── 002-add-species.ts
│   │   │   └── 003-add-sync-metadata.ts
│   │   │
│   │   ├── queries/                  # Database queries
│   │   │   ├── plants.ts             # Plant CRUD operations
│   │   │   ├── care-logs.ts          # Care log operations
│   │   │   ├── care-schedules.ts     # Care schedule operations
│   │   │   ├── species.ts            # Species queries
│   │   │   └── users.ts              # User data operations
│   │   │
│   │   └── seeds/                    # Seed data
│   │       └── species.json          # Species database seed
│   │
│   ├── services/                     # API services
│   │   ├── api-client.ts             # Base API client with auth
│   │   ├── plants-service.ts         # Plants API calls
│   │   ├── care-logs-service.ts      # Care logs API calls
│   │   ├── species-service.ts        # Species API calls
│   │   ├── auth-service.ts           # Authentication API calls
│   │   └── sync-service.ts           # Sync API calls
│   │
│   ├── lib/                          # Utilities and business logic
│   │   ├── date-utils.ts             # Date formatting and calculations
│   │   ├── care-scheduler.ts         # Care scheduling logic
│   │   ├── notification-scheduler.ts # Notification scheduling
│   │   ├── sync-engine.ts            # Sync conflict resolution
│   │   ├── background-tasks.ts       # Background task registration
│   │   ├── storage.ts                # Secure storage wrapper
│   │   └── validation.ts             # Input validation utilities
│   │
│   └── types/                        # TypeScript type definitions
│       ├── plant.ts                  # Plant types
│       ├── care-log.ts               # Care log types
│       ├── care-schedule.ts          # Care schedule types
│       ├── species.ts                # Species types
│       ├── user.ts                   # User types
│       ├── sync.ts                   # Sync types
│       ├── notification.ts           # Notification types
│       └── api.ts                    # API response types
│
├── tests/                            # Test files (mirrors src/)
│   ├── components/
│   │   ├── plants/
│   │   │   └── plant-card.test.tsx
│   │   └── care/
│   │       └── care-button.test.tsx
│   │
│   ├── services/
│   │   ├── plants-service.test.ts
│   │   └── sync-service.test.ts
│   │
│   ├── lib/
│   │   ├── care-scheduler.test.ts
│   │   ├── sync-engine.test.ts
│   │   └── date-utils.test.ts
│   │
│   └── database/
│       └── queries/
│           └── plants.test.ts
│
├── assets/                           # Static assets
│   ├── images/
│   │   ├── species/                  # Species images
│   │   └── icons/                    # App icons
│   └── fonts/                        # Custom fonts
│
└── constants/                        # App constants
    ├── config.ts                     # App configuration
    ├── colors.ts                     # Color palette
    └── care-types.ts                 # Care type definitions
```

**Backend API (Express.js Serverless):**

```
plant-care-api/
├── README.md
├── package.json
├── tsconfig.json
├── .env                              # Environment variables
├── .env.example                      # Environment template
├── .gitignore
├── .eslintrc.js
│
├── src/
│   ├── index.ts                      # API entry point
│   ├── app.ts                        # Express app configuration
│   │
│   ├── routes/                       # API routes
│   │   ├── index.ts                  # Route registration
│   │   ├── auth.ts                   # Auth routes
│   │   ├── plants.ts                 # Plant routes
│   │   ├── care-logs.ts              # Care log routes
│   │   └── species.ts                # Species routes
│   │
│   ├── controllers/                  # Request handlers
│   │   ├── auth-controller.ts        # Auth logic
│   │   ├── plants-controller.ts      # Plant CRUD logic
│   │   ├── care-logs-controller.ts   # Care log logic
│   │   └── species-controller.ts     # Species logic
│   │
│   ├── services/                     # Business logic
│   │   ├── auth-service.ts           # Authentication logic
│   │   ├── plants-service.ts         # Plant business logic
│   │   ├── sync-service.ts           # Sync conflict resolution
│   │   └── email-service.ts          # Email notifications
│   │
│   ├── repositories/                 # Data access layer
│   │   ├── plants-repository.ts      # Plant database operations
│   │   ├── care-logs-repository.ts   # Care log database operations
│   │   ├── species-repository.ts     # Species database operations
│   │   └── users-repository.ts       # User database operations
│   │
│   ├── middleware/                   # Express middleware
│   │   ├── auth-middleware.ts        # JWT authentication
│   │   ├── error-handler.ts          # Error handling middleware
│   │   ├── validation.ts             # Request validation
│   │   └── rate-limiter.ts           # Rate limiting
│   │
│   ├── database/                     # Database configuration
│   │   ├── connection.ts             # Database connection pool
│   │   ├── schema.sql                # PostgreSQL schema
│   │   └── migrations/               # Database migrations
│   │       ├── 001-initial-schema.sql
│   │       ├── 002-add-species.sql
│   │       └── 003-add-sync-metadata.sql
│   │
│   ├── lib/                          # Utilities
│   │   ├── jwt.ts                    # JWT token utilities
│   │   ├── bcrypt.ts                 # Password hashing
│   │   ├── validation.ts             # Input validation
│   │   └── logger.ts                 # Logging utilities
│   │
│   └── types/                        # TypeScript types
│       ├── plant.ts                  # Plant types
│       ├── care-log.ts               # Care log types
│       ├── user.ts                   # User types
│       ├── api.ts                    # API types
│       └── express.d.ts              # Express type extensions
│
├── tests/                            # Test files
│   ├── integration/
│   │   ├── auth.test.ts
│   │   ├── plants.test.ts
│   │   └── sync.test.ts
│   │
│   └── unit/
│       ├── services/
│       │   └── sync-service.test.ts
│       └── lib/
│           └── jwt.test.ts
│
└── scripts/                          # Deployment scripts
    ├── deploy.sh                     # Deployment script
    └── migrate.sh                    # Migration script
```

### Architectural Boundaries

**API Boundaries:**

**Mobile App → Backend API:**
- **Authentication boundary:** All API requests require JWT token in `Authorization: Bearer {token}` header
- **Endpoints:** RESTful resources at `/api/*` (plants, care-logs, species, auth)
- **Data format:** JSON with `{ success, data/error }` wrapper
- **Error handling:** Structured error responses with codes
- **Rate limiting:** Applied at API gateway level

**Mobile App → SQLite Database:**
- **Direct access:** App reads/writes to local SQLite database
- **No network required:** All core operations work offline
- **Sync boundary:** Sync engine mediates between local SQLite and remote PostgreSQL

**Backend API → PostgreSQL Database:**
- **Connection pooling:** Use RDS Proxy or Supabase pooler for serverless
- **Data access layer:** Repositories abstract database operations
- **Transaction boundary:** Controllers manage transaction scope

**Component Boundaries:**

**React Context Providers:**
- **PlantsProvider:** Manages plant inventory state, exposes `usePlants()` hook
- **AuthProvider:** Manages authentication state, exposes `useAuth()` hook
- **SyncProvider:** Manages sync state and queue, exposes `useSync()` hook
- **NotificationsProvider:** Manages notification state, exposes `useNotifications()` hook

**Component Communication:**
- **Parent → Child:** Props for data and callbacks
- **Child → Parent:** Callbacks for events
- **Sibling → Sibling:** Via shared Context provider
- **Cross-feature:** Via Context providers or navigation params

**Service Boundaries:**

**API Services (Mobile):**
- **api-client.ts:** Base HTTP client with auth, error handling, retry logic
- **Feature services:** Wrap api-client for specific resources (plants-service, care-logs-service)
- **Boundary:** Services handle network errors, return typed responses

**Backend Services:**
- **Controllers:** Handle HTTP requests, validate input, call services
- **Services:** Business logic, orchestrate repositories
- **Repositories:** Data access, SQL queries, transaction management
- **Boundary:** Services don't know about HTTP, repositories don't know about business logic

**Data Boundaries:**

**Mobile App Data Flow:**
```
UI Components
    ↓ (via hooks)
Context Providers
    ↓ (read/write)
SQLite Database (local)
    ↓ (sync via)
Sync Engine
    ↓ (HTTP)
Backend API
    ↓ (SQL)
PostgreSQL Database (server)
```

**Sync Boundary:**
- **Optimistic updates:** UI updates immediately, sync in background
- **Conflict detection:** Timestamp-based, last-write-wins with user notification
- **Sync queue:** Failed syncs queued with retry logic
- **Rollback:** Failed syncs rollback local changes

**Database Schema Boundaries:**
- **Mobile (SQLite):** Subset of server schema + sync metadata (lastSyncedAt, isDirty)
- **Server (PostgreSQL):** Full schema with user data, audit logs
- **Shared tables:** plants, careLogs, species (synced)
- **Local-only tables:** notificationHistory, syncQueue
- **Server-only tables:** users, sessions, auditLogs

### Requirements to Structure Mapping

**Plant Inventory Management (FR1-FR6):**
- **Screens:** `app/(tabs)/plants.tsx`, `app/plant/[id].tsx`, `app/plant/add.tsx`, `app/plant/edit/[id].tsx`
- **Components:** `src/components/plants/plant-card.tsx`, `plant-list.tsx`, `plant-form.tsx`, `species-selector.tsx`
- **Context:** `src/contexts/plants-context.tsx`
- **Hook:** `src/hooks/use-plants.ts`
- **Database:** `src/database/queries/plants.ts`
- **API Service:** `src/services/plants-service.ts`
- **Backend:** `src/routes/plants.ts`, `src/controllers/plants-controller.ts`, `src/repositories/plants-repository.ts`

**Care Schedule Management (FR7-FR13):**
- **Components:** `src/components/care/care-schedule.tsx`, `today-task-card.tsx`
- **Business Logic:** `src/lib/care-scheduler.ts` (pure functions for schedule calculation)
- **Database:** `src/database/queries/care-schedules.ts`
- **Hook:** `src/hooks/use-care-schedule.ts`

**Reminder & Notification System (FR14-FR19):**
- **Notification Logic:** `src/lib/notification-scheduler.ts` (schedule 7 days ahead + daily refresh)
- **Background Tasks:** `src/lib/background-tasks.ts` (register daily refresh task)
- **Components:** `src/components/notifications/notification-settings.tsx`, `notification-history.tsx`
- **Context:** `src/contexts/notifications-context.tsx`
- **Hook:** `src/hooks/use-notifications.ts`

**Care Logging & History (FR20-FR24):**
- **Screens:** `app/care/log/[plantId].tsx`, `app/care/history/[plantId].tsx`
- **Components:** `src/components/care/care-button.tsx`, `care-log-item.tsx`
- **Database:** `src/database/queries/care-logs.ts`
- **API Service:** `src/services/care-logs-service.ts`
- **Backend:** `src/routes/care-logs.ts`, `src/controllers/care-logs-controller.ts`, `src/repositories/care-logs-repository.ts`

**Species Database (FR25-FR28):**
- **Components:** `src/components/plants/species-selector.tsx`
- **Database:** `src/database/queries/species.ts`
- **Seed Data:** `src/database/seeds/species.json` (50+ species with default schedules)
- **API Service:** `src/services/species-service.ts`
- **Backend:** `src/routes/species.ts`, `src/controllers/species-controller.ts`, `src/repositories/species-repository.ts`

**Offline Mode & Data Sync (FR29-FR34):**
- **Sync Engine:** `src/lib/sync-engine.ts` (conflict resolution, queue management)
- **Context:** `src/contexts/sync-context.tsx`
- **Hook:** `src/hooks/use-sync.ts`
- **API Service:** `src/services/sync-service.ts`
- **Backend:** `src/services/sync-service.ts` (conflict detection logic)
- **Database:** Sync metadata in all tables (lastSyncedAt, isDirty columns)

**User Onboarding (FR35-FR37):**
- **Screens:** `app/onboarding/welcome.tsx`, `permissions.tsx`, `complete.tsx`
- **Components:** `src/components/onboarding/onboarding-step.tsx`, `permission-prompt.tsx`

**Cross-Cutting Concerns:**

**Authentication (NFR Security):**
- **Screens:** `app/auth/login.tsx`, `register.tsx`, `forgot-password.tsx`
- **Context:** `src/contexts/auth-context.tsx`
- **Hook:** `src/hooks/use-auth.ts`
- **API Service:** `src/services/auth-service.ts`
- **Storage:** `src/lib/storage.ts` (JWT token in expo-secure-store)
- **Backend:** `src/routes/auth.ts`, `src/controllers/auth-controller.ts`, `src/middleware/auth-middleware.ts`

**Error Handling:**
- **Component:** `src/components/common/error-boundary.tsx` (React Error Boundary)
- **Middleware:** `src/middleware/error-handler.ts` (backend error handling)
- **Utilities:** Try-catch blocks in all async operations

**Performance Optimization:**
- **Lazy Loading:** Load "Today" view first in `app/(tabs)/index.tsx`
- **Image Optimization:** Lazy load species images in `src/components/plants/species-selector.tsx`
- **Database Indexing:** Indexes defined in `src/database/schema.ts`

### Integration Points

**Internal Communication:**

**Mobile App Internal:**
- **UI → Context:** Components call hooks (`usePlants()`, `useAuth()`, `useSync()`)
- **Context → Database:** Context providers call database queries
- **Context → API Services:** Context providers call API services for sync
- **Services → API Client:** All API services use shared `api-client.ts` with auth

**Backend API Internal:**
- **Routes → Controllers:** Routes delegate to controllers
- **Controllers → Services:** Controllers call services for business logic
- **Services → Repositories:** Services call repositories for data access
- **Repositories → Database:** Repositories execute SQL queries

**External Integrations:**

**Mobile App → Backend API:**
- **Protocol:** HTTPS with TLS 1.2+
- **Authentication:** JWT bearer tokens
- **Base URL:** Environment variable `API_BASE_URL`
- **Retry Logic:** Exponential backoff in `api-client.ts`

**Backend API → PostgreSQL:**
- **Connection:** Connection pooling via RDS Proxy or Supabase pooler
- **Credentials:** Environment variables (`DATABASE_URL`)
- **Migrations:** Manual SQL migrations in `src/database/migrations/`

**Mobile App → Expo Services:**
- **Notifications:** `expo-notifications` for local push notifications
- **Storage:** `expo-secure-store` for encrypted JWT storage
- **Background Tasks:** `expo-background-task` for daily notification refresh
- **Haptics:** `expo-haptics` for tactile feedback

**Data Flow:**

**Plant Creation Flow:**
```
1. User fills form in app/plant/add.tsx
2. Form submits to PlantsProvider via usePlants() hook
3. PlantsProvider:
   a. Validates input
   b. Writes to local SQLite (src/database/queries/plants.ts)
   c. Updates UI optimistically
   d. Queues sync operation
4. SyncProvider (background):
   a. Calls plants-service.ts to POST /api/plants
   b. Backend creates plant in PostgreSQL
   c. Returns created plant with server ID
   d. Updates local SQLite with server ID
   e. Marks as synced
5. If sync fails:
   a. Keeps local plant with isDirty flag
   b. Retries with exponential backoff
   c. Shows sync status in UI
```

**Notification Scheduling Flow:**
```
1. User adds plant or logs care
2. care-scheduler.ts calculates next care dates
3. notification-scheduler.ts:
   a. Cancels old notifications for plant
   b. Schedules next 7 days of notifications
   c. Stores notification IDs in SQLite
4. Daily background task (background-tasks.ts):
   a. Runs at 6 AM daily
   b. Recalculates schedules for all plants
   c. Reschedules notifications for next 7 days
5. User taps notification:
   a. Opens app to care/log/[plantId].tsx
   b. Pre-fills care action from notification
   c. One-tap to log care
```

**Sync Conflict Resolution Flow:**
```
1. User edits plant on Device A (offline)
2. User edits same plant on Device B (offline)
3. Device A comes online, syncs first:
   a. Sends plant update to backend
   b. Backend updates PostgreSQL
   c. Returns success
4. Device B comes online, syncs second:
   a. Sends plant update to backend
   b. Backend detects conflict (updatedAt mismatch)
   c. Returns 409 Conflict with server version
5. sync-engine.ts on Device B:
   a. Compares local vs server version
   b. Applies last-write-wins (server wins)
   c. Shows notification to user about conflict
   d. Updates local SQLite with server version
```

### File Organization Patterns

**Configuration Files:**
- **Root level:** `package.json`, `tsconfig.json`, `app.json`, `eas.json`
- **Environment:** `.env.local` (local), `.env.example` (template, committed)
- **Build:** `babel.config.js`, `metro.config.js` (React Native bundler)
- **Linting:** `.eslintrc.js`, `.prettierrc`

**Source Organization:**
- **Screens:** `app/` directory (Expo Router file-based routing)
- **Components:** `src/components/` organized by feature (plants, care, notifications, common)
- **Business Logic:** `src/lib/` for pure functions and utilities
- **Data Access:** `src/database/` for SQLite, `src/services/` for API calls
- **State Management:** `src/contexts/` for React Context providers
- **Types:** `src/types/` for shared TypeScript types

**Test Organization:**
- **Mirror source:** `tests/` directory mirrors `src/` structure
- **Test files:** `{filename}.test.ts` or `{filename}.test.tsx`
- **Test utilities:** `tests/__mocks__/` for mocks and fixtures
- **Integration tests:** `tests/integration/` for API integration tests

**Asset Organization:**
- **Images:** `assets/images/` with subdirectories (species, icons)
- **Fonts:** `assets/fonts/` for custom fonts
- **Species images:** `assets/images/species/` (downloaded on first launch)

### Development Workflow Integration

**Development Server Structure:**
- **Mobile:** `npx expo start` runs Metro bundler, serves from `app/` and `src/`
- **Backend:** `npm run dev` runs Express server with hot reload
- **Database:** Local SQLite file in app data directory, PostgreSQL via Docker or cloud

**Build Process Structure:**
- **Mobile:** EAS Build reads `eas.json`, builds from `app/` and `src/`, outputs to `dist/`
- **Backend:** TypeScript compilation outputs to `dist/`, serverless deployment packages `dist/`
- **Assets:** Expo optimizes images and fonts during build

**Deployment Structure:**
- **Mobile:** EAS Build → App Store / Google Play
- **Backend:** Serverless functions (Vercel or AWS Lambda) deployed from `dist/`
- **Database:** Managed PostgreSQL (AWS RDS, Supabase, or PlanetScale)
- **Migrations:** Manual SQL migrations run via `scripts/migrate.sh`


## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
All architectural decisions are fully compatible and work together seamlessly:

- **Frontend Stack:** React Native 0.83 + Expo SDK 55 + TypeScript strict mode are fully compatible and represent the latest stable versions
- **Backend Stack:** Express.js v5.2 + Node.js 18+ are compatible and production-ready
- **Database Strategy:** SQLite (mobile) + PostgreSQL (server) is a proven pattern for offline-first mobile apps with server sync
- **Hosting Strategy:** Serverless functions (AWS Lambda/Vercel) + managed PostgreSQL is a standard, well-supported pattern with connection pooling solutions (RDS Proxy, Supabase pooler)
- **Authentication:** JWT tokens + Express middleware + expo-secure-store is a standard, secure pattern
- **API Design:** REST with structured responses + React Context + optimistic updates is a coherent, proven pattern

**No conflicting decisions identified.** All technology choices support each other and enable the chosen architectural patterns.

**Pattern Consistency:**
Implementation patterns fully support architectural decisions:

- **Naming Conventions:** camelCase for code/database aligns with JavaScript/TypeScript ecosystem, kebab-case for files works across all platforms
- **State Management:** Immutable updates with spread operator matches React best practices and Context API requirements
- **Error Handling:** Try-catch for async operations aligns with async/await pattern and React Error Boundaries
- **API Format:** `{ success, data/error }` wrapper provides consistent structure for both success and error cases
- **Date Format:** ISO 8601 strings work seamlessly with JavaScript `Date` objects and JSON serialization
- **Loading States:** Boolean flags (`isLoading`, `isSyncing`) provide clear, explicit state management

**All patterns are internally consistent and mutually reinforcing.**

**Structure Alignment:**
Project structure fully supports architectural decisions and patterns:

- **Expo Router:** File-based routing in `app/` directory matches chosen navigation solution
- **Feature Organization:** Components organized by feature (plants, care, notifications) supports modularity and clear boundaries
- **Test Structure:** Separate `tests/` directory mirroring source structure aligns with chosen test location pattern
- **Database Layer:** Dedicated `src/database/` with migrations, queries, and seeds supports schema evolution and data access patterns
- **Service Layer:** `src/services/` for API calls supports separation of concerns and testability
- **Context Providers:** `src/contexts/` for state management aligns with React Context + hooks pattern

**Structure enables all architectural decisions without friction.**

### Requirements Coverage Validation ✅

**Functional Requirements Coverage (37 FRs across 7 categories):**

1. **Plant Inventory Management (FR1-FR6):** ✅ Fully Covered
   - CRUD operations: `src/components/plants/`, `src/database/queries/plants.ts`, `/api/plants` endpoints
   - Species selection: `src/components/plants/species-selector.tsx`, `src/database/seeds/species.json`
   - Search/filter: `src/hooks/use-plants.ts` with filtering logic
   - Screens: `app/(tabs)/plants.tsx`, `app/plant/[id].tsx`, `app/plant/add.tsx`

2. **Care Schedule Management (FR7-FR13):** ✅ Fully Covered
   - Default schedules: `src/database/seeds/species.json` with care intervals
   - Per-plant customization: `src/database/queries/care-schedules.ts`
   - Pause/resume: `isPaused` flag in plants table
   - Schedule calculation: `src/lib/care-scheduler.ts` (pure functions)
   - Upcoming tasks: `app/(tabs)/index.tsx` (Today view)

3. **Reminder & Notification System (FR14-FR19):** ✅ Fully Covered
   - Local push notifications: `expo-notifications` package
   - Hybrid scheduling: `src/lib/notification-scheduler.ts` (7 days ahead + daily refresh)
   - Background refresh: `src/lib/background-tasks.ts` with `expo-background-task`
   - User configuration: `src/components/notifications/notification-settings.tsx`
   - Notification history: `src/components/notifications/notification-history.tsx`

4. **Care Logging & History (FR20-FR24):** ✅ Fully Covered
   - Timestamped logging: `src/database/queries/care-logs.ts`
   - Backdating support: Date picker in `app/care/log/[plantId].tsx`
   - History views: `app/care/history/[plantId].tsx`
   - Quick logging: `src/components/care/care-button.tsx` (one-tap from notification)

5. **Species Database (FR25-FR28):** ✅ Fully Covered
   - 50+ species: `src/database/seeds/species.json`
   - Default schedules: Included in species seed data
   - Local caching: SQLite storage in `src/database/queries/species.ts`
   - Offline access: Species data seeded on first launch

6. **Offline Mode & Data Sync (FR29-FR34):** ✅ Fully Covered
   - Complete offline functionality: SQLite-first architecture
   - Background sync: `src/lib/sync-engine.ts` with queue and retry logic
   - Conflict resolution: Timestamp-based detection, last-write-wins with user notification
   - Sync triggers: App launch, daily background refresh, manual pull-to-refresh
   - Optimistic updates: Context providers update UI immediately, rollback on error

7. **User Onboarding (FR35-FR37):** ✅ Fully Covered
   - Guided setup: `app/onboarding/` screens (welcome, permissions, complete)
   - Notification permissions: `app/onboarding/permissions.tsx` with permission prompt
   - Skip-friendly flow: Navigation allows skipping non-critical steps

**All 37 functional requirements are architecturally supported with specific components, services, and data structures.**

**Non-Functional Requirements Coverage:**

- **Performance (2s launch, 1s actions, 500ms search, 5s sync):** ✅ Addressed
  - Lazy loading strategy: Load "Today" view first, defer full plant list
  - Progressive rendering: Skeleton UI immediately, populate asynchronously
  - Database indexing: Indexes on frequently queried columns (userId, plantId, createdAt)
  - Async operations: All I/O operations use async/await, non-blocking

- **Notification Reliability (95% delivery within 1-hour window):** ✅ Addressed
  - Hybrid scheduling: 7 days ahead (works when app killed) + daily refresh (dynamic adjustment)
  - Graceful degradation: In-app "Today" view as primary interface
  - Delivery monitoring: Track scheduled vs delivered notifications
  - Fallback: Notification history for missed notifications

- **Offline-First (zero performance degradation):** ✅ Addressed
  - SQLite-first architecture: All core operations work offline
  - Sync as enhancement: Sync happens in background, doesn't block UI
  - Optimistic updates: UI updates immediately, sync later
  - No network checks: App never checks network before operations

- **Scalability (100 plants, 10K logs, 25K users):** ✅ Addressed
  - Database indexing: Efficient queries for large datasets
  - Pagination: Species list and care history paginated
  - Connection pooling: RDS Proxy or Supabase pooler for serverless
  - Serverless auto-scaling: Handles traffic spikes automatically

- **Security (encryption, TLS 1.2+, 30-day sessions):** ✅ Addressed
  - Data at rest: `expo-secure-store` for JWT tokens and sensitive data
  - Data in transit: TLS 1.2+ enforced by serverless platform
  - Authentication: JWT tokens with 30-day expiration, refresh token pattern
  - Password hashing: bcrypt with cost factor 12
  - API security: JWT bearer tokens, input validation, rate limiting

- **Privacy (minimal data collection, no third-party analytics):** ✅ Addressed
  - Local-first architecture: Data stays on device unless user syncs
  - No analytics: No third-party analytics in MVP
  - Minimal server data: Only synced plant and care log data
  - User control: User can delete account and all data

**All non-functional requirements are architecturally addressed with specific strategies and implementations.**

### Implementation Readiness Validation ✅

**Decision Completeness:**
All critical architectural decisions are documented with specific details:

- ✅ **Technology versions verified:** Express.js v5.2, Expo SDK 55, React Native 0.83, Node.js 18+
- ✅ **State management decided:** React Context + Custom Hooks with immutable updates
- ✅ **Database strategy decided:** SQLite (mobile) with camelCase naming, PostgreSQL (server) with managed hosting
- ✅ **Backend framework decided:** Custom Node.js API with Express.js v5.2
- ✅ **Authentication decided:** JWT with email/password, bcrypt hashing, 30-day expiration
- ✅ **Notification strategy decided:** Hybrid (7 days ahead + daily refresh) with `expo-notifications`
- ✅ **API design decided:** Resource-based REST with plural endpoints, structured error responses
- ✅ **Hosting decided:** Serverless functions (AWS Lambda or Vercel) with managed PostgreSQL
- ✅ **Monitoring decided:** Cloud provider logs (CloudWatch or Vercel logs) for MVP
- ✅ **CI/CD decided:** Manual deployment for MVP, automation post-pilot

**No critical decisions are missing or ambiguous.**

**Structure Completeness:**
Project structure is fully defined and implementation-ready:

- ✅ **Complete directory trees:** Both mobile app and backend API have complete file/directory structures
- ✅ **All files explicitly defined:** Every component, service, utility, and configuration file is specified
- ✅ **Integration points mapped:** Data flow diagrams for plant creation, notification scheduling, sync conflicts
- ✅ **Component boundaries clear:** React Context providers, service layers, repository patterns all defined
- ✅ **Requirements mapped to structure:** Each FR category mapped to specific files and directories

**AI agents have a complete blueprint for where every piece of code should live.**

**Pattern Completeness:**
Implementation patterns address all potential conflict points:

- ✅ **Naming conventions comprehensive:** Database (camelCase), API (plural resources), files (kebab-case), code (camelCase), constants (UPPER_SNAKE_CASE)
- ✅ **Structure patterns defined:** Test location (tests/ directory), component organization (by feature), file organization (src/ structure)
- ✅ **Format patterns specified:** API responses ({ success, data/error }), dates (ISO 8601), JSON fields (camelCase)
- ✅ **Communication patterns documented:** State updates (immutable), Context providers (per domain), hooks (use{Domain})
- ✅ **Process patterns established:** Error handling (try-catch + Error Boundaries), loading states (boolean flags), async operations (async/await)
- ✅ **Examples provided:** Good examples and anti-patterns for all major patterns
- ✅ **Validation checklist included:** AI agents can verify pattern compliance before implementation

**All potential AI agent conflict points are addressed with clear, enforceable patterns.**

### Gap Analysis Results

**Critical Gaps:** None identified ✅

**Important Gaps:** None identified ✅

**Minor Enhancement Opportunities (Optional, not blocking):**

1. **Database Index Specifications:**
   - Current: Indexes mentioned in general terms
   - Enhancement: Could specify exact index definitions (e.g., `CREATE INDEX idx_plants_userId ON plants(userId)`)
   - Impact: Low - indexes can be added during implementation based on query patterns
   - Recommendation: Define during Epic 1 (Foundation) when schema is implemented

2. **Expo Package Versions:**
   - Current: Expo SDK 55 specified, individual packages implied
   - Enhancement: Could specify exact versions (e.g., `expo-notifications@~0.28.0`, `expo-sqlite@~14.0.0`)
   - Impact: Low - Expo SDK 55 pins compatible versions automatically
   - Recommendation: Lock versions in package.json during project initialization

3. **API Rate Limiting Specifics:**
   - Current: "Basic rate limiting at API gateway level" mentioned
   - Enhancement: Could specify exact limits (e.g., 100 requests/minute per user)
   - Impact: Low - can be tuned based on actual usage patterns during pilot
   - Recommendation: Start with platform defaults, adjust based on monitoring

4. **PostgreSQL Version:**
   - Current: "Managed PostgreSQL" specified
   - Enhancement: Could recommend specific version (e.g., PostgreSQL 15 or 16)
   - Impact: Low - managed services default to stable versions
   - Recommendation: Use managed service default (typically latest stable)

**None of these gaps block implementation. All can be decided during implementation based on actual needs.**

### Validation Issues Addressed

**No blocking issues identified during validation.** ✅

The architecture is coherent, complete, and ready for implementation. All decisions work together, all requirements are covered, and AI agents have comprehensive guidance for consistent implementation.

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed (scale, complexity, constraints)
- [x] Functional requirements mapped to architectural components (37 FRs across 7 categories)
- [x] Non-functional requirements addressed (performance, security, scalability, privacy)
- [x] Technical constraints identified (offline-first, notification reliability, cross-platform)
- [x] Cross-cutting concerns mapped (authentication, error handling, sync, notifications)
- [x] Pre-mortem analysis completed (5 risk areas with mitigation strategies)
- [x] First principles foundation established (scheduling engine with memory)

**✅ Architectural Decisions**
- [x] Framework selected and validated (React Native + Expo SDK 55)
- [x] Starter template chosen (Expo SDK 55 Default Template with TypeScript)
- [x] State management decided (React Context + Custom Hooks)
- [x] Database strategy defined (SQLite mobile, PostgreSQL server, camelCase naming)
- [x] Backend framework selected (Express.js v5.2 with custom API)
- [x] Authentication method chosen (JWT with email/password)
- [x] Notification strategy defined (Hybrid: 7 days ahead + daily refresh)
- [x] API design patterns established (Resource-based REST, structured errors)
- [x] Hosting strategy decided (Serverless functions with managed PostgreSQL)
- [x] Monitoring approach selected (Cloud provider logs for MVP)
- [x] CI/CD pipeline defined (Manual deployment for MVP)
- [x] All technology versions verified (Express 5.2, Expo SDK 55, Node 18+, React Native 0.83)

**✅ Implementation Patterns**
- [x] Naming conventions established (camelCase, kebab-case, PascalCase, UPPER_SNAKE_CASE)
- [x] Structure patterns defined (tests/ directory, feature-based components, src/ organization)
- [x] Format patterns specified (API responses, dates, JSON fields, booleans)
- [x] Communication patterns documented (immutable updates, Context providers, hooks)
- [x] Process patterns established (error handling, loading states, async operations)
- [x] Concrete examples provided (good patterns and anti-patterns)
- [x] Validation checklist created (pattern compliance verification)
- [x] Enforcement guidelines documented (code reviews, linting, type checking)

**✅ Project Structure**
- [x] Complete directory structure defined (mobile app and backend API)
- [x] All files and directories explicitly specified
- [x] Component boundaries established (React Context, service layers, repositories)
- [x] Integration points mapped (data flow diagrams for key operations)
- [x] Requirements to structure mapping complete (each FR mapped to specific files)
- [x] Cross-cutting concerns located (auth, error handling, performance)
- [x] Development workflow integrated (dev server, build process, deployment)

**✅ Architecture Validation**
- [x] Coherence validated (all decisions compatible and mutually reinforcing)
- [x] Requirements coverage verified (all 37 FRs and NFRs architecturally supported)
- [x] Implementation readiness confirmed (decisions, patterns, structure all complete)
- [x] Gap analysis completed (no critical gaps, minor enhancements identified)
- [x] Validation issues addressed (no blocking issues found)

### Architecture Readiness Assessment

**Overall Status:** ✅ **READY FOR IMPLEMENTATION**

**Confidence Level:** **HIGH**

The architecture is comprehensive, coherent, and implementation-ready. All critical decisions are documented with specific versions, all requirements are architecturally supported, and AI agents have complete guidance for consistent implementation.

**Key Strengths:**

1. **Offline-First Foundation:** SQLite-first architecture with sync as enhancement ensures core functionality works without network, addressing the most critical requirement

2. **Proven Technology Stack:** React Native + Expo SDK 55 + Express.js v5.2 are mature, well-supported technologies with strong ecosystems and team familiarity

3. **Comprehensive Pattern Documentation:** 9 conflict areas addressed with concrete examples, anti-patterns, and validation checklists ensure AI agents write consistent code

4. **Complete Project Structure:** Every file and directory explicitly defined with requirements mapped to specific locations provides clear implementation blueprint

5. **Coherent Decision Set:** All architectural decisions are compatible and mutually reinforcing, with no conflicts or contradictions

6. **Risk Mitigation:** Pre-mortem analysis identified 5 critical risk areas with specific mitigation strategies built into the architecture

7. **Graceful Degradation:** Notification reliability hierarchy (must work offline → should work with degraded experience → nice to have enhancements) ensures robust user experience

8. **Clear Boundaries:** Component boundaries, service boundaries, and data boundaries are well-defined with explicit communication patterns

**Areas for Future Enhancement (Post-MVP):**

1. **Advanced Monitoring:** Add Sentry for error tracking and LogRocket for session replay when error rates justify the cost

2. **Automated CI/CD:** Implement GitHub Actions for automated testing and deployment after pilot validates product-market fit

3. **Performance Optimization:** Add caching strategies (Redis, CDN) if performance monitoring reveals bottlenecks

4. **Multi-Region Deployment:** Expand to multiple regions if user base grows beyond single region capacity

5. **Advanced Sync:** Implement operational transformation or CRDTs for more sophisticated conflict resolution if last-write-wins proves insufficient

6. **Species Database Updates:** Build admin API for species database updates without app releases

7. **OAuth Integration:** Add Google/Apple Sign-In for faster onboarding if user feedback indicates demand

**These enhancements are not needed for MVP and can be prioritized based on actual usage patterns and user feedback during pilot.**

### Implementation Handoff

**AI Agent Guidelines:**

1. **Follow architectural decisions exactly as documented:**
   - Use specified technology versions (Express 5.2, Expo SDK 55, Node 18+)
   - Implement patterns as defined (camelCase naming, kebab-case files, immutable updates)
   - Respect project structure (src/ organization, tests/ mirroring, feature-based components)

2. **Use implementation patterns consistently across all components:**
   - Apply naming conventions to all code (variables, functions, files, database tables)
   - Follow API response format ({ success, data/error }) for all endpoints
   - Use try-catch for all async operations
   - Implement immutable state updates with spread operator
   - Use boolean loading flags with is{Action} naming

3. **Respect project structure and boundaries:**
   - Place files in correct directories (components by feature, tests mirroring source)
   - Respect component boundaries (Context providers, service layers, repositories)
   - Follow integration patterns (data flow diagrams for plant creation, notifications, sync)

4. **Refer to this document for all architectural questions:**
   - Check naming conventions before creating files or variables
   - Verify API response format before implementing endpoints
   - Review data flow diagrams before implementing cross-component features
   - Consult pattern examples when unsure about implementation approach

5. **Validate pattern compliance before committing code:**
   - Use validation checklist in "Implementation Patterns & Consistency Rules" section
   - Verify naming conventions match documented patterns
   - Ensure API responses use correct format
   - Confirm state updates are immutable
   - Check error handling uses try-catch blocks

**First Implementation Priority:**

**Epic 1, Story 1: Project Initialization**

```bash
# Initialize Expo project with SDK 55 default template
npx create-expo-app@latest plant-care-tracker --template default@sdk-55

# Navigate to project directory
cd plant-care-tracker

# Install additional dependencies
npm install expo-sqlite expo-notifications expo-secure-store expo-haptics expo-background-task

# Initialize git repository
git init
git add .
git commit -m "Initial commit: Expo SDK 55 project with TypeScript"
```

**Next Steps After Initialization:**
1. Set up SQLite database schema and migrations (Epic 1, Story 2)
2. Implement React Context providers for state management (Epic 1, Story 3)
3. Create basic navigation structure with Expo Router (Epic 1, Story 4)
4. Build authentication API and JWT middleware (Epic 2, Story 1)

**Architecture document is complete and ready to guide implementation.** ✅
