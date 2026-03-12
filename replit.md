# MemberSync - AI-Powered Member Engagement Platform

## Overview
MemberSync is an AI-powered member engagement platform designed to enhance member retention and satisfaction for Mainstreet REALTORS®. It provides tools for staff to manage and engage their member base, featuring a three-tier member classification system, comprehensive engagement scoring, and compliance tracking. A key capability is the advanced SmartSends email system, which utilizes AI for audience building and content generation. The platform aims to improve engagement for over 19,000 members, significantly boosting member loyalty and providing a competitive advantage through intelligent, data-driven strategies.

## User Preferences
- Bi-directional navigation between members and offices (clickable links)
- Data tables support pagination: 50, 100, 250, or "View All"
- Mobile-responsive with bottom navigation on small screens

## System Architecture
The application uses a React + TypeScript + Vite frontend and an Express.js + TypeScript backend, styled with Tailwind CSS v4. Authentication is managed via JWT tokens with `httpOnly` cookies and bcryptjs for password hashing, supporting `admin`, `staff`, and `readonly` roles. State management is handled by TanStack Query v5, and Wouter is used for routing. UI/UX leverages Shadcn UI components with custom theming, Inter and Overpass fonts, and dark mode support.

**Core Architectural Features:**
-   **Member Classification & Engagement Scoring**: Members are categorized into three tiers with an engagement score (0-10). Scoring logic is centralized for consistency. Member profiles display real-time data for compliance (Illinois CE, Code of Ethics, license expiration), production statistics, and a detailed engagement score breakdown.
-   **AI Services & Shared Schema**: Four AI services (Query, Audience Builder, Chat, Email Generator) share a common database schema for data awareness and tier management. AI content generation adheres to strict tone guardrails (`MAINSTREET_BRIEFING`). An "Ask Anything" chatbox uses Anthropic's Claude. The AI Chat system includes a RAG memory layer: `ai_business_context` (Membio-authored, read-only) and `ai_memory` (staff-managed entries) are stored in `tenant_config` and injected into every AI query's system prompt. A terminology rule ensures the AI uses client-specific language from the business context. The settings page at `/chats/settings` provides CRUD for memory entries (500 char per entry, 5K soft limit, 8K hard cap). Memory context fetch is failure-safe — AI continues without context if the fetch fails.
-   **Canonical Data Consistency**: All member and office counts are derived from `member_metrics_cache` for consistent reporting.
-   **SmartSends System**:
    -   **Homepage**: A command center displaying activity, performance metrics, top campaigns, and quick actions.
    -   **Create Page**: Entry point for creating emails, sequences, automations, and triggers.
    -   **Reporting & Analytics**: Unified campaign reporting with trend charts and filterable lists, merging email campaigns, drafts, and triggered campaigns. Campaigns are typed (Email, Trigger, Automation, Sequence). Campaign details provide comprehensive statistics and member activity logs.
    -   **Campaign Architect**: A conversational AI for planning multi-email campaign sequences, proposing structured plans with timing, tone, A/B tests, and event anchoring. It supports conditional branching (registration skip/alternate, prior attendance) and ensures plan persistence. It includes save confirmation indicators for audience and event selections.
    -   **Settings**: Includes an Email Governor for send-rate limiting, performance metrics, send queue, email archive, and staff-facing unsubscribe management.
-   **Triggered Campaigns**: Automated email campaigns triggered by member activities with configurable send windows and throttling.
-   **Email System**: High-performance platform with background queue processing, campaign versioning, image library, opt-out tracking, and merge field previewing. It supports four SendGrid unsubscribe groups (Marketing, Recognition, Compliance, Transactional) and a category-aware unsubscribe process. The SendGrid webhook handler (`POST /api/webhooks/sendgrid/events`) processes unsubscribe events with category awareness: `group_unsubscribe` events use `asm_group_id` mapped against `tenant_config` group IDs to set only the matching category opt-out (`marketing_opt_out` or `recognition_opt_out`); compliance/transactional group unsubscribes are logged but no suppression is applied; plain `unsubscribe` events (no group context) set only global `email_opt_out`; `group_resubscribe` events clear the matching category opt-out. Email suppression is universal (members AND non-members) using `email_address` as the **primary key** of `member_email_preferences`. The `member_key` column is nullable (with a partial index `idx_member_email_prefs_member_key WHERE member_key IS NOT NULL`), allowing suppression of external contacts without member records. Unsubscribe/preference center links use `?e=[email]&t=[HMAC]` tokens. `canSendToRecipient()` checks by email address (with conditional `member_key` clause only when truthy); `canSendToMember()` resolves email via `resolveEmailForMember()` before delegating, incrementing a suppression health failure counter on resolution failures. All write paths use UPSERT keyed on `email_address` (`ON CONFLICT (email_address) DO UPDATE`). The bulk opt-out check in `queueEmailCampaign` uses Drizzle `inArray` (no raw SQL string interpolation). Startup migration resolves placeholder `email_address` values (rows without `@`) and sets `migration_status` to `resolved` or `unresolved`. Staff unsubscribe management supports search by name, email, or Member ID. A `getSuppressionHealthStatus()` function tracks email resolution failures (24h rolling window, degraded at 10+ failures) and is exposed on `GET /api/admin/usage` as `suppression_health`. A member preference center allows recipients to manage their opt-out settings. Email health dashboard provides insights into performance.
-   **Audience Management**: Supports manual, CSV upload, filter-based, and AI-generated SQL audiences.
-   **Caching Infrastructure**: `activity_cache` and `member_metrics_cache` pre-compute data daily for faster loading.
-   **Compliance Dashboard**: Tracks Illinois CE, Code of Ethics, and license expiration.
-   **Analytics & Production Profiles**: Provides office and agent production analytics with YoY comparison and leaderboards.
-   **Member Recognition**: Metrics-driven recognition with integration to SmartSends, including a review queue for recognition emails.
-   **Duplicate Detection & Archive-Aware AI**: AI compares draft emails against recent campaigns to prevent duplication and considers past email summaries to avoid repetition during content generation.
-   **Image Library**: Manages optimized, categorized images with alt text, supporting detailed management, filtering, tag-based search, and drag-and-drop upload.
-   **AI Usage Logging & Cost Controls**: All AI services log token usage and estimated costs, enforcing daily rate limits per feature with configurable limits via `tenant_config`.

## External Dependencies
-   **SendGrid**: For email sending, event webhooks, and sandbox mode.
-   **Anthropic Claude**: Powers AI Assistant and SmartSends AI features.
-   **Replit App Storage (Object Storage)**: For persistent storage of uploaded images and rendered email HTML.
-   **PostgreSQL**: The primary database.
-   **Google Fonts**: For Inter and Overpass fonts.