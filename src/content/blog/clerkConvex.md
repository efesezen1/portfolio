---
title: "Clerk + Convex + Expo"
description: "Authentication Implementation Guide"
pubDate: "Feb 13 2026"
heroImage: "/clerkConvex.png"
---

# Authentication Implementation Guide

> This document describes how auth is implemented in the Competitive Sudoku project (React Native + Expo + Convex). Use it as a blueprint to replicate the same auth system in another project.

---

## Table of Contents

1. [Tech Stack & Packages](#1-tech-stack--packages)
2. [Environment Variables](#2-environment-variables)
3. [Backend (Convex)](#3-backend-convex)
   - [Auth Config](#31-auth-config)
   - [User Schema](#32-user-schema)
   - [Auth Helper](#33-auth-helper-getuser)
   - [Webhook Handler](#34-clerk-webhook-handler)
   - [User Mutations (Internal)](#35-user-mutations-internal--webhook-triggered)
   - [User Mutations (Public)](#36-user-mutations-public)
   - [Onboarding Mutations](#37-onboarding-mutations)
   - [Auth Enforcement Patterns](#38-auth-enforcement-patterns-in-mutationsqueries)
4. [Webhook Setup (Step-by-Step)](#4-webhook-setup-step-by-step)
   - [Why Webhooks Are Needed](#41-why-webhooks-are-needed)
   - [Step 1: Install Dependencies](#42-step-1-install-dependencies)
   - [Step 2: Define User Schema](#43-step-2-define-user-schema)
   - [Step 3: Create Internal Mutations](#44-step-3-create-internal-mutations)
   - [Step 4: Create HTTP Endpoint](#45-step-4-create-http-endpoint)
   - [Step 5: Deploy Convex](#46-step-5-deploy-convex)
   - [Step 6: Configure Clerk Dashboard](#47-step-6-configure-clerk-dashboard)
   - [Step 7: Set Environment Variable](#48-step-7-set-environment-variable)
   - [Step 8: Verify It Works](#49-step-8-verify-it-works)
   - [Gotchas & Troubleshooting](#410-gotchas--troubleshooting)
5. [Frontend (React Native / Expo)](#5-frontend-react-native--expo)
   - [Auth Provider Setup](#51-auth-provider-setup)
   - [Root Layout Integration](#52-root-layout-integration)
   - [Route Protection](#53-route-protection)
   - [Sign-In / Sign-Up Form](#54-sign-in--sign-up-form)
   - [Email Verification Form](#55-email-verification-form)
   - [Nickname / Onboarding Form](#56-nickname--onboarding-form)
   - [Apple Sign-In](#57-apple-sign-in-ios-only)
   - [Auth Modal (Multi-Step Flow)](#58-auth-modal-multi-step-flow)
   - [Sign-Out & Account Deletion](#59-sign-out--account-deletion)
   - [User Online/Offline Status](#510-user-onlineoffline-status-hook)
6. [Auth Flow Diagrams](#6-auth-flow-diagrams)
7. [Clerk Hook Reference](#7-clerk-hook-reference)
8. [Security Summary](#8-security-summary)

---

## 1. Tech Stack & Packages

| Package                     | Version  | Purpose                                                            |
| --------------------------- | -------- | ------------------------------------------------------------------ |
| `@clerk/clerk-expo`         | ^2.19.20 | Clerk SDK for React Native/Expo (client-side auth)                 |
| `@clerk/backend`            | ^2.29.5  | Clerk backend utilities (webhook type definitions)                 |
| `convex`                    | ^1.30.0  | Backend framework (provides `ConvexProviderWithClerk`, `ctx.auth`) |
| `svix`                      | ^1.84.1  | Webhook signature verification (validates Clerk webhooks)          |
| `expo-secure-store`         | ~15.0.8  | Encrypted token storage on device                                  |
| `expo-apple-authentication` | ~8.0.8   | Native Apple Sign-In button/flow (iOS)                             |
| `expo-auth-session`         | ~7.0.10  | OAuth session handling                                             |
| `expo-web-browser`          | ~15.0.9  | Opens browser for OAuth flows                                      |

**Install command:**

```bash
npx expo install @clerk/clerk-expo expo-secure-store expo-apple-authentication expo-auth-session expo-web-browser
npm install @clerk/backend svix
```

---

## 2. Environment Variables

```env
# Clerk
CLERK_JWT_ISSUER_DOMAIN=https://<your-clerk-instance>.clerk.accounts.dev
CLERK_WEBHOOK_SECRET=whsec_<your-webhook-secret>
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_<your-publishable-key>

# Convex
EXPO_PUBLIC_CONVEX_URL=https://<your-convex-deployment>.convex.cloud
```

- `CLERK_JWT_ISSUER_DOMAIN` -- Used by Convex backend to validate JWTs.
- `CLERK_WEBHOOK_SECRET` -- Used by `svix` to verify webhook signatures from Clerk.
- `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` -- Public key passed to `ClerkProvider` on the client.
- `EXPO_PUBLIC_CONVEX_URL` -- Convex deployment URL for the `ConvexReactClient`.

---

## 3. Backend (Convex)

### 3.1 Auth Config

**File: `convex/auth.config.ts`**

```typescript
import { AuthConfig } from "convex/server";

export default {
  providers: [
    {
      domain: process.env.CLERK_JWT_ISSUER_DOMAIN!,
      applicationID: "convex",
    },
  ],
} satisfies AuthConfig;
```

This tells Convex to accept JWTs issued by your Clerk instance. The `applicationID: "convex"` is the standard value for the Clerk-Convex integration.

---

### 3.2 User Schema

**File: `convex/types.ts`** (user structure definition)

```typescript
const userStructure = {
  externalId: v.string(), // Clerk user ID (from identity.subject)
  name: v.optional(v.string()), // Full name from Clerk (first + last)
  nickname: v.optional(v.string()), // User-chosen nickname (set during onboarding)
  displayTag: v.optional(v.string()), // "nickname#1234" unique tag
  status: v.union(v.literal("online"), v.literal("offline")),
  image_url: v.optional(v.string()), // Clerk avatar URL
  has_image: v.optional(v.boolean()),
  email: v.optional(v.string()), // Email from Clerk
} as const;
```

**File: `convex/schema.ts`** (table + indexes)

```typescript
users: userTable
  .index("by_externalId", ["externalId"])
  .index("by_displayTag", ["displayTag"]);
```

Key point: `externalId` is the Clerk user ID. It's used to look up the Convex user record from the JWT identity.

---

### 3.3 Auth Helper (`getUser`)

**File: `convex/helpers/auth.ts`**

This is the **central auth enforcement function** called by every mutation and query that needs the current user.

```typescript
import { Id } from "../_generated/dataModel";
import { MutationCtx, QueryCtx } from "../_generated/server";

// Overload 1: Soft lookup (returns null if not authenticated)
export async function getUser(
  ctx: QueryCtx | MutationCtx,
): Promise<Id<"users"> | null>;

// Overload 2: Required auth (throws if not authenticated)
export async function getUser(
  ctx: QueryCtx | MutationCtx,
  options: { required: true; createIfMissing?: boolean },
): Promise<Id<"users">>;

// Implementation
export async function getUser(
  ctx: QueryCtx | MutationCtx,
  options?: { required?: true; createIfMissing?: boolean },
): Promise<Id<"users"> | null> {
  // 1. Extract JWT identity from the request
  const identity = await ctx.auth.getUserIdentity();

  if (!identity) {
    if (options?.required) {
      throw new Error("Not authenticated");
    }
    return null;
  }

  // 2. Look up user in Convex DB by Clerk's user ID
  const user = await ctx.db
    .query("users")
    .withIndex("by_externalId", (q) => q.eq("externalId", identity.subject))
    .unique();

  if (user) {
    return user._id;
  }

  // 3. Optionally create user record if missing
  if (options?.createIfMissing) {
    const userId = await ctx.db.insert("users", {
      externalId: identity.subject,
      name: identity.name ?? undefined,
      email: identity.email ?? undefined,
      status: "online",
    });
    return userId;
  }

  if (options?.required) {
    throw new Error("Not authenticated");
  }
  return null;
}
```

**Three usage modes:**

| Call                                                      | Behavior                                             |
| --------------------------------------------------------- | ---------------------------------------------------- |
| `getUser(ctx)`                                            | Returns user ID or `null`                            |
| `getUser(ctx, { required: true })`                        | Returns user ID or throws `"Not authenticated"`      |
| `getUser(ctx, { required: true, createIfMissing: true })` | Returns user ID, creating the record on first access |

---

### 3.4 Clerk Webhook Handler

**File: `convex/http.ts`**

Handles user lifecycle events from Clerk (created/updated/deleted):

```typescript
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { internal } from "./_generated/api";
import type { WebhookEvent } from "@clerk/backend";
import { Webhook } from "svix";

const http = httpRouter();

// Webhook signature verification
async function validateRequest(req: Request): Promise<WebhookEvent | null> {
  const payloadString = await req.text();
  const svixHeaders = {
    "svix-id": req.headers.get("svix-id")!,
    "svix-timestamp": req.headers.get("svix-timestamp")!,
    "svix-signature": req.headers.get("svix-signature")!,
  };
  const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
  try {
    return wh.verify(payloadString, svixHeaders) as unknown as WebhookEvent;
  } catch (error) {
    console.error("Error verifying webhook event", error);
    return null;
  }
}

// Webhook route
http.route({
  path: "/clerk-users-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const event = await validateRequest(request);
    if (!event) return new Response("Invalid webhook", { status: 400 });

    switch (event.type) {
      case "user.created":
      case "user.updated":
        await ctx.runMutation(internal.users.upsertFromClerk, {
          data: event.data,
        });
        break;
      case "user.deleted":
        const clerkUserId = event.data.id!;
        await ctx.runMutation(internal.users.deleteFromClerk, { clerkUserId });
        break;
    }
    return new Response(null, { status: 200 });
  }),
});

export default http;
```

**Clerk Dashboard setup required:** Create a webhook endpoint pointing to `https://<your-convex-deployment>.convex.site/clerk-users-webhook` subscribing to `user.created`, `user.updated`, `user.deleted`.

---

### 3.5 User Mutations (Internal / Webhook-Triggered)

**File: `convex/users.ts`**

These are `internalMutation` -- only callable from the server (webhooks, scheduler), never from the client.

```typescript
import { internalMutation } from "./_generated/server";
import type { UserJSON } from "@clerk/backend";

export const upsertFromClerk = internalMutation({
  args: { data: v.any() as Validator<UserJSON> },
  async handler(ctx, { data }) {
    const userAttributes = {
      name: `${data.first_name} ${data.last_name}`,
      email: data.email_addresses[0].email_address || "",
      externalId: data.id,
      image_url: data.image_url,
      has_image: data.has_image,
    };
    const user = await userByExternalId(ctx, data.id);
    if (user === null) {
      await ctx.db.insert("users", userAttributes);
    } else {
      await ctx.db.patch(user._id, userAttributes);
    }
  },
});

export const deleteFromClerk = internalMutation({
  args: { clerkUserId: v.string() },
  async handler(ctx, { clerkUserId }) {
    const user = await userByExternalId(ctx, clerkUserId);
    if (user !== null) {
      // Clean up related data (games, invites, queue entries, etc.)
      await performUserDataCleanup(ctx, user._id);
      await ctx.db.delete(user._id);
    }
  },
});

// Helper
async function userByExternalId(ctx: QueryCtx, externalId: string) {
  return await ctx.db
    .query("users")
    .withIndex("by_externalId", (q) => q.eq("externalId", externalId))
    .unique();
}
```

---

### 3.6 User Mutations (Public)

**File: `convex/users.ts`**

```typescript
export const getCurrentUser = query({
  args: {},
  handler: async (ctx) => {
    const userId = await getUser(ctx);
    if (!userId) return null;
    const user = await ctx.db.get(userId);
    return {
      id: user._id,
      nickname: user.nickname,
      displayTag: user.displayTag,
      status: user.status,
    };
  },
});

export const setUserStatus = mutation({
  args: { status: userStatusValidator },
  handler: async (ctx, args) => {
    const userId = await getUser(ctx);
    if (!userId) throw new Error("Not authenticated");
    await ctx.db.patch(userId, { status: args.status });
  },
});
```

---

### 3.7 Onboarding Mutations

**File: `convex/onboarding.ts`**

```typescript
export const hasCompletedOnboarding = query({
  args: {},
  handler: async (ctx) => {
    const userId = await getUser(ctx, {
      required: true,
      createIfMissing: true,
    });
    const user = await ctx.db.get(userId);
    return !!user?.nickname;
  },
});

export const setNickname = mutation({
  args: { nickname: v.string() },
  handler: async (ctx, args) => {
    const userId = await getUser(ctx, {
      required: true,
      createIfMissing: true,
    });
    // Validates nickname (3-20 chars, alphanumeric + underscore)
    // Generates a unique displayTag (nickname#XXXX)
    await ctx.db.patch(userId, { nickname, displayTag });
  },
});

export const skipNickname = mutation({
  args: {},
  handler: async (ctx) => {
    const userId = await getUser(ctx, {
      required: true,
      createIfMissing: true,
    });
    // Assigns a random default nickname
  },
});
```

---

### 3.8 Auth Enforcement Patterns in Mutations/Queries

Every protected endpoint calls `getUser()` at the top:

```typescript
// Pattern 1: Hard requirement (most mutations)
export const doSomething = mutation({
  handler: async (ctx, args) => {
    const userId = await getUser(ctx, { required: true });
    // userId is guaranteed non-null
  },
});

// Pattern 2: Hard requirement + auto-create (onboarding, first-time actions)
export const firstTimeAction = mutation({
  handler: async (ctx, args) => {
    const userId = await getUser(ctx, {
      required: true,
      createIfMissing: true,
    });
    // User record is created in Convex if it doesn't exist yet
  },
});

// Pattern 3: Soft check (public queries, optional features)
export const optionalData = query({
  handler: async (ctx) => {
    const userId = await getUser(ctx);
    if (!userId) return null; // Graceful unauthenticated response
    // ...
  },
});
```

---

## 4. Webhook Setup (Step-by-Step)

This section is a detailed, actionable guide for setting up the Clerk-to-Convex webhook pipeline from scratch. Webhooks keep your Convex `users` table in sync with Clerk -- when a user signs up, updates their profile, or deletes their account in Clerk, your Convex database is updated automatically.

### 4.1 Why Webhooks Are Needed

Convex validates JWTs from Clerk via `ctx.auth.getUserIdentity()`, but that only gives you the JWT claims (subject, email, name). It does **not** create a record in your database. You need a `users` table in Convex to:

- Store app-specific fields (nickname, status, preferences, etc.)
- Create relationships between users and other data (games, messages, etc.)
- Query users by ID in mutations and queries

The webhook ensures that **every Clerk user event** (create, update, delete) is reflected in your Convex `users` table in real-time.

**Data flow:**

```
User signs up in your app (via Clerk)
  --> Clerk creates the user
  --> Clerk fires a POST to your webhook endpoint
  --> Convex HTTP endpoint receives it
  --> Svix verifies the signature
  --> Internal mutation inserts/updates the user in Convex DB
```

### 4.2 Step 1: Install Dependencies

```bash
npm install @clerk/backend svix
```

| Package          | Purpose                                                      |
| ---------------- | ------------------------------------------------------------ |
| `@clerk/backend` | Provides the `WebhookEvent` and `UserJSON` TypeScript types  |
| `svix`           | Verifies webhook signatures (Clerk uses Svix under the hood) |

These are backend-only packages. They run inside Convex actions/http handlers, not on the client.

### 4.3 Step 2: Define User Schema

**File: `convex/schema.ts`**

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  users: defineTable({
    externalId: v.string(), // Clerk user ID (maps to identity.subject)
    name: v.optional(v.string()), // From Clerk: first_name + last_name
    email: v.optional(v.string()), // From Clerk: primary email
    image_url: v.optional(v.string()), // From Clerk: avatar URL
    has_image: v.optional(v.boolean()),
    // Add your app-specific fields here:
    nickname: v.optional(v.string()),
    status: v.union(v.literal("online"), v.literal("offline")),
  }).index("by_externalId", ["externalId"]), // REQUIRED: for looking up users by Clerk ID
});
```

The `by_externalId` index is **critical** -- both the webhook handler and the `getUser()` auth helper use it to find users by their Clerk ID.

### 4.4 Step 3: Create Internal Mutations

**File: `convex/users.ts`**

These are `internalMutation` -- they can only be called from the server (HTTP handlers, schedulers, other internal functions). The client **cannot** call them directly. This is a security boundary.

```typescript
import { UserJSON } from "@clerk/backend";
import { v, Validator } from "convex/values";
import { internalMutation, MutationCtx, QueryCtx } from "./_generated/server";

// ── Helper: look up user by Clerk ID ──────────────────────────────
async function userByExternalId(ctx: QueryCtx, externalId: string) {
  return await ctx.db
    .query("users")
    .withIndex("by_externalId", (q) => q.eq("externalId", externalId))
    .unique();
}

// ── Upsert: called on user.created AND user.updated ──────────────
export const upsertFromClerk = internalMutation({
  args: { data: v.any() as Validator<UserJSON> },
  async handler(ctx, { data }) {
    // Map Clerk fields to your schema fields
    const userAttributes = {
      name: `${data.first_name} ${data.last_name}`,
      email: data.email_addresses[0]?.email_address || "",
      externalId: data.id,
      image_url: data.image_url,
      has_image: data.has_image,
    };

    const existingUser = await userByExternalId(ctx, data.id);

    if (existingUser === null) {
      // First time -- insert with default app-specific fields
      await ctx.db.insert("users", {
        ...userAttributes,
        status: "online", // default status
        nickname: undefined, // set during onboarding
      });
    } else {
      // Existing user -- only update Clerk-managed fields
      // Do NOT overwrite app-specific fields like nickname, status
      await ctx.db.patch(existingUser._id, userAttributes);
    }
  },
});

// ── Delete: called on user.deleted ────────────────────────────────
export const deleteFromClerk = internalMutation({
  args: { clerkUserId: v.string() },
  async handler(ctx, { clerkUserId }) {
    const user = await userByExternalId(ctx, clerkUserId);

    if (user !== null) {
      // IMPORTANT: Clean up all related data before deleting the user.
      // This is app-specific. Examples:
      //   - Delete their games, posts, comments, etc.
      //   - Remove them from teams/organizations
      //   - Cancel pending invites
      //   - Abandon active multiplayer sessions
      await performUserDataCleanup(ctx, user._id);

      await ctx.db.delete(user._id);
    } else {
      console.warn(
        `Can't delete user, there is none for Clerk user ID: ${clerkUserId}`,
      );
    }
  },
});

// ── App-specific cleanup (customize this) ─────────────────────────
async function performUserDataCleanup(ctx: MutationCtx, userId: Id<"users">) {
  // Example: delete all solo games belonging to this user
  // const games = await ctx.db
  //   .query("solo_games")
  //   .withIndex("by_player", (q) => q.eq("player.id", userId))
  //   .collect();
  // for (const game of games) {
  //   await ctx.db.delete(game._id);
  // }
}
```

**Key design decisions:**

- `upsertFromClerk` handles both `user.created` and `user.updated` with a single mutation (idempotent).
- On insert, you set app-specific defaults (`status: "online"`).
- On patch, you only update Clerk-managed fields -- never overwrite `nickname`, `status`, etc.
- `deleteFromClerk` cleans up **all related data** before deleting the user record. This is critical to avoid orphaned records.
- `v.any() as Validator<UserJSON>` skips runtime validation (we trust Clerk's payload), but gives TypeScript the correct type.

### 4.5 Step 4: Create HTTP Endpoint

**File: `convex/http.ts`**

This is the actual webhook receiver. It validates the signature and routes events to the internal mutations.

```typescript
import type { WebhookEvent } from "@clerk/backend";
import { httpRouter } from "convex/server";
import { Webhook } from "svix";
import { internal } from "./_generated/api";
import { httpAction } from "./_generated/server";

const http = httpRouter();

// ── Webhook route ─────────────────────────────────────────────────
http.route({
  path: "/clerk-users-webhook",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    // 1. Validate the webhook signature
    const event = await validateRequest(request);
    if (!event) {
      return new Response("Invalid webhook signature", { status: 400 });
    }

    // 2. Route by event type
    switch (event.type) {
      case "user.created": // intentional fallthrough
      case "user.updated":
        await ctx.runMutation(internal.users.upsertFromClerk, {
          data: event.data,
        });
        break;

      case "user.deleted": {
        const clerkUserId = event.data.id!;
        await ctx.runMutation(internal.users.deleteFromClerk, {
          clerkUserId,
        });
        break;
      }

      default:
        console.log("Ignored Clerk webhook event", event.type);
    }

    // 3. Always return 200 to acknowledge receipt
    return new Response(null, { status: 200 });
  }),
});

// ── Signature verification using Svix ─────────────────────────────
async function validateRequest(req: Request): Promise<WebhookEvent | null> {
  const payloadString = await req.text();

  // Clerk sends these three headers with every webhook
  const svixHeaders = {
    "svix-id": req.headers.get("svix-id")!,
    "svix-timestamp": req.headers.get("svix-timestamp")!,
    "svix-signature": req.headers.get("svix-signature")!,
  };

  // Verify using the webhook secret from your environment
  const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);
  try {
    return wh.verify(payloadString, svixHeaders) as unknown as WebhookEvent;
  } catch (error) {
    console.error("Error verifying webhook event", error);
    return null;
  }
}

export default http;
```

**How signature verification works:**

1. Clerk signs every webhook payload using your `CLERK_WEBHOOK_SECRET` (via Svix).
2. Three headers are sent: `svix-id`, `svix-timestamp`, `svix-signature`.
3. The `Webhook.verify()` method re-computes the signature from the payload + secret and compares it to the header.
4. If they don't match (tampered payload, wrong secret, replay attack), it throws and we return 400.

### 4.6 Step 5: Deploy Convex

```bash
npx convex deploy
```

After deploying, note your **HTTP Actions URL**. It will be:

```
https://<your-deployment-name>.convex.site
```

**IMPORTANT:** The domain ends in `.convex.site`, NOT `.convex.cloud`. The `.cloud` domain is for queries/mutations. The `.site` domain is for HTTP actions (webhooks, file serving, etc.).

Your full webhook endpoint URL will be:

```
https://<your-deployment-name>.convex.site/clerk-users-webhook
```

### 4.7 Step 6: Configure Clerk Dashboard

1. Go to your **Clerk Dashboard** (https://dashboard.clerk.com).
2. Navigate to **Webhooks** in the left sidebar.
3. Click **Add Endpoint**.
4. Fill in:
   - **Endpoint URL:** `https://<your-deployment-name>.convex.site/clerk-users-webhook`
   - **Message Filtering:** Click on **"user"** to subscribe to all user events. This subscribes to:
     - `user.created` -- fires when a new user signs up
     - `user.updated` -- fires when user profile changes (name, email, avatar)
     - `user.deleted` -- fires when user deletes their account
5. Click **Create**.
6. After creation, you'll see a **Signing Secret** that starts with `whsec_`. **Copy this value** -- you'll need it in the next step.

### 4.8 Step 7: Set Environment Variable

Set the signing secret as an environment variable in your **Convex Dashboard**:

1. Go to your **Convex Dashboard** (https://dashboard.convex.dev).
2. Select your project.
3. Go to **Settings** --> **Environment Variables**.
4. Add:
   - **Name:** `CLERK_WEBHOOK_SECRET`
   - **Value:** `whsec_<the-value-you-copied>`
5. Save.

Alternatively, set it in your `.env.local` file (for local development with `npx convex dev`):

```env
CLERK_WEBHOOK_SECRET=whsec_<your-signing-secret>
```

### 4.9 Step 8: Verify It Works

1. In the **Clerk Dashboard** Webhooks page, click on your endpoint.
2. Go to the **Testing** tab.
3. Select event type `user.created` and click **Send Test Webhook**.
4. Check the **Logs** tab for a `200` response status.
5. In your **Convex Dashboard**, go to **Data** and check the `users` table -- you should see a test user record.

You can also verify end-to-end:

1. Sign up a real user in your app.
2. Check the Convex `users` table -- a new record should appear within 1-2 seconds.
3. Update the user's profile in Clerk Dashboard -- the Convex record should update.
4. Delete the user in Clerk Dashboard -- the Convex record (and related data) should be removed.

### 4.10 Gotchas & Troubleshooting

| Issue                                   | Cause                                                  | Fix                                                                           |
| --------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------- |
| Webhook returns 400                     | Wrong `CLERK_WEBHOOK_SECRET`                           | Re-copy the signing secret from Clerk Dashboard and update in Convex env vars |
| Webhook returns 400                     | Secret set in `.env.local` but not in Convex Dashboard | Set it in **both** places (local for dev, dashboard for production)           |
| Endpoint unreachable                    | Wrong URL domain                                       | Use `.convex.site`, not `.convex.cloud`                                       |
| User created in Clerk but not in Convex | Webhook not deployed yet                               | Run `npx convex deploy` first, then configure the webhook                     |
| `user.deleted` event but user not found | User was created before webhooks were set up           | Handle gracefully with `console.warn` (already done in the code above)        |
| Duplicate user records                  | Missing `.unique()` on query                           | Always use `.unique()` with the `by_externalId` index                         |
| Types error on `UserJSON`               | Missing `@clerk/backend`                               | Run `npm install @clerk/backend`                                              |
| `event.data.id!` TypeScript error       | Clerk types mark `id` as optional on delete events     | The `!` assertion is safe here -- Clerk always sends the ID                   |

**Timing note:** There is a slight delay (typically < 1 second) between a Clerk action and the webhook arriving. If your app queries the Convex user immediately after sign-up, the record might not exist yet. That's why the `getUser()` helper has the `createIfMissing` option -- it acts as a fallback to create the user record from the JWT identity if the webhook hasn't arrived yet.

**Retry behavior:** If your endpoint returns a non-2xx status, Clerk/Svix will retry the webhook with exponential backoff. You don't need to implement retry logic yourself.

---

## 5. Frontend (React Native / Expo)

### 5.1 Auth Provider Setup

**File: `components/auth/index.tsx`**

```typescript
import { ClerkProvider, useAuth } from "@clerk/clerk-expo";
import { ConvexReactClient } from "convex/react";
import { ConvexProviderWithClerk } from "convex/react-clerk";
import * as SecureStore from "expo-secure-store";

const convex = new ConvexReactClient(
  process.env.EXPO_PUBLIC_CONVEX_URL as string
);

// Clerk uses this to persist tokens securely on device
const tokenCache = {
  async getToken(key: string) {
    return SecureStore.getItemAsync(key);
  },
  async saveToken(key: string, value: string) {
    return SecureStore.setItemAsync(key, value);
  },
};

export default function AuthProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ClerkProvider
      tokenCache={tokenCache}
      publishableKey={process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!}
    >
      <ConvexProviderWithClerk client={convex} useAuth={useAuth}>
        {children}
      </ConvexProviderWithClerk>
    </ClerkProvider>
  );
}
```

**What happens here:**

1. `ClerkProvider` initializes Clerk, manages session state, stores tokens via `expo-secure-store`.
2. `ConvexProviderWithClerk` bridges Clerk and Convex -- it reads the Clerk JWT via `useAuth` and automatically attaches it to every Convex request. This is what makes `ctx.auth.getUserIdentity()` work on the backend.

---

### 5.2 Root Layout Integration

**File: `app/_layout.tsx`**

```typescript
import AuthProvider from "@/components/auth";

export default function RootLayout() {
  return (
    <KeyboardProvider>
      <AuthProvider>
        <GestureHandlerRootView>
          <HeroUINativeProvider>
            <BottomSheetModalProvider>
              <Stack screenOptions={{ headerShown: false }}>
                <Stack.Screen name="index" />
                <Stack.Screen name="Auth/sign-in" />
                <Stack.Screen name="Auth/verification" />
                <Stack.Screen name="Auth/nickname" />
                {/* ... game screens */}
              </Stack>
            </BottomSheetModalProvider>
          </HeroUINativeProvider>
        </GestureHandlerRootView>
      </AuthProvider>
    </KeyboardProvider>
  );
}
```

`AuthProvider` wraps the entire app so auth state is available everywhere.

---

### 5.3 Route Protection

**File: `app/index.tsx`**

```typescript
import { useAuth } from "@clerk/clerk-expo";
import { useQuery } from "convex/react";
import { Redirect } from "expo-router";
import { api } from "@/convex/_generated/api";

const Index = () => {
  const { isLoaded, isSignedIn } = useAuth();

  if (!isLoaded) return <LoadingScreen />;

  if (!isSignedIn) {
    // Show unauthenticated menu (with "Claim Account" button)
    return <UnauthenticatedMenu />;
  }

  return <AuthenticatedContent />;
};

const AuthenticatedContent = () => {
  const hasCompletedOnboarding = useQuery(api.onboarding.hasCompletedOnboarding);

  if (hasCompletedOnboarding === undefined) return <LoadingScreen />;

  // Force onboarding (nickname) if not completed
  if (!hasCompletedOnboarding) {
    return <Redirect href="/Auth/nickname" />;
  }

  return <Sudoku />;
};
```

**Two-layer protection:**

1. **Clerk layer:** `isSignedIn` from `useAuth()` -- checks if user has a valid session.
2. **App layer:** `hasCompletedOnboarding` from Convex query -- ensures user has set their nickname.

---

### 5.4 Sign-In / Sign-Up Form

**File: `components/auth/SignInForm.tsx`**

```typescript
import { useSignIn, useSignUp } from "@clerk/clerk-expo";

interface SignInFormProps {
  onSignInComplete: () => void;
  onSignUpNeedsVerification: (email: string) => void;
}

export default function SignInForm({
  onSignInComplete,
  onSignUpNeedsVerification,
}: SignInFormProps) {
  const {
    signIn,
    setActive: setSignInActive,
    isLoaded: isSignInLoaded,
  } = useSignIn();
  const {
    signUp,
    setActive: setSignUpActive,
    isLoaded: isSignUpLoaded,
  } = useSignUp();

  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");

  // --- SIGN IN ---
  const handleSignIn = async () => {
    const signInAttempt = await signIn.create({
      identifier: email.trim(),
      password: password.trim(),
    });

    if (signInAttempt.status === "complete") {
      await setSignInActive({ session: signInAttempt.createdSessionId });
      onSignInComplete(); // Navigate away
    }
  };

  // --- SIGN UP ---
  const handleSignUp = async () => {
    await signUp.create({
      emailAddress: email.trim(),
      password: password.trim(),
    });

    // Trigger email verification code
    await signUp.prepareEmailAddressVerification({
      strategy: "email_code",
    });

    onSignUpNeedsVerification(email.trim()); // Navigate to verification screen
  };

  // ... render email input, password input, sign-in/sign-up buttons
}
```

**Clerk error codes handled:**
| Code | Meaning |
|------|---------|
| `form_password_incorrect` | Wrong password |
| `form_identifier_not_found` | No account with this email |
| `form_param_format_invalid` | Invalid email format |
| `form_identifier_exists` | Email already registered |
| `form_password_pwned` | Password found in breached database |
| `form_password_length_too_short` | Password too short |

---

### 5.5 Email Verification Form

**File: `components/auth/VerificationForm.tsx`**

```typescript
import { useSignUp } from "@clerk/clerk-expo";

interface VerificationFormProps {
  email: string;
  onVerificationComplete: () => void;
  onBack: () => void;
}

export default function VerificationForm({
  email,
  onVerificationComplete,
  onBack,
}: VerificationFormProps) {
  const { signUp, setActive } = useSignUp();
  const [code, setCode] = useState("");

  // Verify the 6-digit code
  const handleVerifyCode = async () => {
    const signUpAttempt = await signUp.attemptEmailAddressVerification({
      code: code.trim(),
    });

    if (signUpAttempt.status === "complete") {
      await setActive({ session: signUpAttempt.createdSessionId });
      onVerificationComplete(); // Navigate to nickname screen
    }
  };

  // Resend the code
  const handleResendCode = async () => {
    await signUp.prepareEmailAddressVerification({
      strategy: "email_code",
    });
  };

  // ... render code input, verify button, resend button
}
```

---

### 5.6 Nickname / Onboarding Form

**File: `components/auth/NicknameForm.tsx`**

```typescript
import { useMutation } from "convex/react";
import { api } from "@/convex/_generated/api";

interface NicknameFormProps {
  onComplete: () => void;
}

export default function NicknameForm({ onComplete }: NicknameFormProps) {
  const setNickname = useMutation(api.onboarding.setNickname);
  const skipNickname = useMutation(api.onboarding.skipNickname);
  const [nickname, setNicknameValue] = useState("");

  const handleSubmit = async () => {
    // Validation: 3-20 chars, alphanumeric + underscore
    if (!/^[a-zA-Z0-9_]+$/.test(nickname) || nickname.length < 3) return;

    await setNickname({ nickname });
    onComplete();
  };

  const handleSkip = async () => {
    await skipNickname();
    onComplete();
  };

  // ... render nickname input, submit button, skip button
}
```

---

### 5.7 Apple Sign-In (iOS Only)

**File: `components/AppleSignInButton.tsx`**

```typescript
import { useSignInWithApple } from "@clerk/clerk-expo";
import * as AppleAuthentication from "expo-apple-authentication";
import { Platform } from "react-native";

interface AppleSignInButtonProps {
  onSignInComplete?: () => void;
}

export function AppleSignInButton({ onSignInComplete }: AppleSignInButtonProps) {
  const { startAppleAuthenticationFlow } = useSignInWithApple();

  const handleAppleSignIn = async () => {
    try {
      const { createdSessionId, setActive } =
        await startAppleAuthenticationFlow();

      if (createdSessionId && setActive) {
        await setActive({ session: createdSessionId });
        onSignInComplete?.();
      }
    } catch (err: any) {
      if (err.code === "ERR_REQUEST_CANCELED") return; // User cancelled
      Alert.alert("Error", err.message);
    }
  };

  // Only render on iOS
  if (Platform.OS !== "ios") return null;

  return (
    <AppleAuthentication.AppleAuthenticationButton
      buttonType={AppleAuthentication.AppleAuthenticationButtonType.SIGN_IN}
      buttonStyle={
        colorScheme === "dark"
          ? AppleAuthentication.AppleAuthenticationButtonStyle.WHITE
          : AppleAuthentication.AppleAuthenticationButtonStyle.BLACK
      }
      cornerRadius={20}
      style={{ width: "100%", height: 55 }}
      onPress={handleAppleSignIn}
    />
  );
}
```

**Clerk Dashboard setup required:** Enable Apple as a social connection in Clerk. Configure Apple Developer credentials (Services ID, Key, etc.).

---

### 5.8 Auth Modal (Multi-Step Flow)

**File: `components/AuthModal.tsx`**

Used on the unauthenticated home screen to present auth as a modal overlay:

```typescript
export function AuthModal({ visible, onClose }: AuthModalProps) {
  const [currentScreen, setCurrentScreen] = useState<"signIn" | "verification" | "nickname">("signIn");
  const [email, setEmail] = useState("");

  return (
    <Modal visible={visible} animationType="slide" presentationStyle="pageSheet">
      {currentScreen === "signIn" && (
        <SignInForm
          onSignInComplete={onClose}
          onSignUpNeedsVerification={(email) => {
            setEmail(email);
            setCurrentScreen("verification");
          }}
        />
      )}
      {currentScreen === "verification" && (
        <VerificationForm
          email={email}
          onVerificationComplete={() => setCurrentScreen("nickname")}
          onBack={() => setCurrentScreen("signIn")}
        />
      )}
      {currentScreen === "nickname" && (
        <NicknameForm onComplete={onClose} />
      )}
    </Modal>
  );
}
```

**Flow:** Sign In/Up --> Email Verification --> Nickname Setup --> Done

---

### 5.9 Sign-Out & Account Deletion

**File: `components/SettingsContent.tsx`**

```typescript
import { useAuth, useUser } from "@clerk/clerk-expo";

const { signOut, isSignedIn } = useAuth();
const { user: clerkUser } = useUser();

// Sign out
const handleSignOut = async () => {
  await signOut();
  // User is returned to unauthenticated view automatically
};

// Delete account (removes from Clerk, which triggers webhook to clean up Convex)
const handleDeleteAccount = async () => {
  await clerkUser?.delete();
};
```

**Account deletion cascade:**

1. `clerkUser.delete()` deletes the user in Clerk.
2. Clerk fires a `user.deleted` webhook.
3. The webhook handler calls `deleteFromClerk` in Convex.
4. Convex cleans up all user data (games, invites, queue entries) and deletes the user record.

---

### 5.10 User Online/Offline Status Hook

**File: `hooks/useUserStatus.ts`**

```typescript
import { useConvexAuth } from "convex/react";
import { AppState } from "react-native";

export const useUserStatus = () => {
  const setUserStatus = useMutation(api.users.setUserStatus);
  const { isAuthenticated } = useConvexAuth();

  useEffect(() => {
    if (isAuthenticated) {
      setUserStatus({ status: "online" });
    }

    const subscription = AppState.addEventListener("change", (nextAppState) => {
      if (isAuthenticated) {
        setUserStatus({
          status: nextAppState === "active" ? "online" : "offline",
        });
      }
    });

    return () => {
      if (isAuthenticated) setUserStatus({ status: "offline" });
      subscription.remove();
    };
  }, [isAuthenticated]);
};
```

---

## 6. Auth Flow Diagrams

### Sign-Up Flow

```
User enters email + password
  --> signUp.create({ emailAddress, password })
  --> signUp.prepareEmailAddressVerification({ strategy: "email_code" })
  --> User receives 6-digit code via email
  --> signUp.attemptEmailAddressVerification({ code })
  --> setActive({ session: createdSessionId })
  --> Clerk session is active, JWT issued
  --> ConvexProviderWithClerk attaches JWT to requests
  --> First Convex mutation calls getUser(ctx, { required: true, createIfMissing: true })
  --> User record created in Convex with externalId = Clerk user ID
  --> User sets nickname (onboarding)
  --> Full access granted
```

### Sign-In Flow

```
User enters email + password
  --> signIn.create({ identifier, password })
  --> setActive({ session: createdSessionId })
  --> Clerk session active, JWT issued
  --> Convex requests authenticated automatically
```

### Apple Sign-In Flow

```
User taps Apple Sign-In button
  --> startAppleAuthenticationFlow()
  --> Native Apple auth dialog
  --> setActive({ session: createdSessionId })
  --> Clerk session active
```

### JWT Validation (Every Backend Request)

```
Client sends request with JWT (automatic via ConvexProviderWithClerk)
  --> Convex validates JWT against CLERK_JWT_ISSUER_DOMAIN
  --> ctx.auth.getUserIdentity() returns identity with subject = Clerk user ID
  --> getUser() looks up user by externalId index
  --> Returns user._id for authorization checks
```

---

## 7. Clerk Hook Reference

| Hook                   | Package             | Returns                             | Usage                                  |
| ---------------------- | ------------------- | ----------------------------------- | -------------------------------------- |
| `useAuth()`            | `@clerk/clerk-expo` | `{ isLoaded, isSignedIn, signOut }` | Check auth state, sign out             |
| `useUser()`            | `@clerk/clerk-expo` | `{ user }`                          | Access Clerk user object (for delete)  |
| `useSignIn()`          | `@clerk/clerk-expo` | `{ signIn, setActive, isLoaded }`   | Email/password sign-in                 |
| `useSignUp()`          | `@clerk/clerk-expo` | `{ signUp, setActive, isLoaded }`   | Email/password sign-up + verification  |
| `useSignInWithApple()` | `@clerk/clerk-expo` | `{ startAppleAuthenticationFlow }`  | Apple Sign-In                          |
| `useConvexAuth()`      | `convex/react`      | `{ isAuthenticated }`               | Check if Convex has a valid auth token |

---

## 8. Security Summary

| Layer                          | Mechanism                                                        |
| ------------------------------ | ---------------------------------------------------------------- |
| **Client token storage**       | `expo-secure-store` (encrypted device storage)                   |
| **Client-to-backend auth**     | JWT automatically attached by `ConvexProviderWithClerk`          |
| **Backend JWT validation**     | Convex validates against `CLERK_JWT_ISSUER_DOMAIN`               |
| **Webhook security**           | `svix` verifies signatures using `CLERK_WEBHOOK_SECRET`          |
| **Per-endpoint authorization** | `getUser()` helper at the top of every mutation/query            |
| **Internal mutations**         | `internalMutation` prevents client from calling webhook handlers |
| **Account deletion**           | Cascading cleanup of all user data via webhook                   |
| **Token refresh**              | Handled automatically by Clerk SDK                               |
