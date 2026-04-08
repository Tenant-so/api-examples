# Tenant.so API Examples

> Code examples and integration patterns for the [Tenant.so](https://tenant.so) platform — webhooks, authentication, member management, payments, and more.

Tenant.so is built on a modern stack: **Next.js**, **tRPC**, **Prisma**, **Stripe**, **Ably** (real-time), and **PostgreSQL**. These examples demonstrate common integration patterns for extending and automating your community.

---

## Contents

- [Authentication](#authentication)
- [Webhooks](#webhooks)
- [Member Management](#member-management)
- [Payments & Subscriptions](#payments--subscriptions)
- [Content & Posts](#content--posts)
- [Real-Time Events](#real-time-events)
- [Email Integration](#email-integration)
- [Automation Patterns](#automation-patterns)
- [Tech Stack](#tech-stack)

---

## Authentication

### Session-Based (Web)

Tenant.so uses NextAuth.js for web authentication with session cookies.

```typescript
// Checking authentication in an API route
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";

export async function GET(request: Request) {
  const session = await getServerSession(authOptions);
  
  if (!session?.user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }
  
  // session.user.id — authenticated user ID
  // session.user.tenantId — current tenant context
  return Response.json({ user: session.user });
}
```

### JWT Bearer (Mobile / API)

For mobile apps and external API access, Tenant.so supports JWT Bearer tokens.

```typescript
// Sending authenticated requests from a mobile app
const response = await fetch("https://your-community.tenant.so/api/trpc/posts.list", {
  headers: {
    "Authorization": `Bearer ${jwtToken}`,
    "x-client-type": "mobile",
    "Content-Type": "application/json",
  },
});
```

---

## Webhooks

### Stripe Webhook Handler

Handle payment events to grant/revoke community access.

```typescript
// src/app/api/webhooks/stripe/route.ts
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, signature, webhookSecret);
  } catch (err) {
    return Response.json({ error: "Invalid signature" }, { status: 400 });
  }

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.Checkout.Session;
      // Grant community access
      await grantAccess(session.customer as string, session.metadata);
      break;
    }
    case "customer.subscription.deleted": {
      const subscription = event.data.object as Stripe.Subscription;
      // Revoke access
      await revokeAccess(subscription.customer as string);
      break;
    }
  }

  return Response.json({ received: true });
}
```

### Member Event Webhooks

Listen for member lifecycle events.

```typescript
// Example webhook payload for member.joined
{
  "event": "member.joined",
  "timestamp": "2026-04-08T12:00:00Z",
  "data": {
    "userId": "user_abc123",
    "tenantId": "tenant_xyz789",
    "email": "member@example.com",
    "joinSource": "free_signup", // or "paid", "invite_link"
    "groups": ["group_public_lounge"]
  }
}

// Example webhook payload for subscription.created
{
  "event": "subscription.created",
  "timestamp": "2026-04-08T12:00:00Z",
  "data": {
    "userId": "user_abc123",
    "tenantId": "tenant_xyz789",
    "planId": "price_monthly_49",
    "status": "active",
    "currentPeriodEnd": "2026-05-08T12:00:00Z"
  }
}
```

---

## Member Management

### Fetching Member Data

```typescript
// Using tRPC client
import { api } from "@/lib/trpc";

// List members with pagination
const members = await api.members.list.query({
  page: 1,
  limit: 20,
  search: "john",
  role: "MEMBER", // MEMBER | ADMIN
});

// Get a specific member
const member = await api.members.get.query({
  userId: "user_abc123",
});
```

### Member Roles & Permissions

```typescript
// Role hierarchy
enum Role {
  MEMBER = "MEMBER",     // Basic community access
  ADMIN = "ADMIN",       // Full administrative access
}

enum SpaceRole {
  MEMBER = "MEMBER",       // Can read/post in space
  MODERATOR = "MODERATOR", // Can moderate content in space
}

// Check if user is admin
const isAdmin = membership.role === "ADMIN";

// Check if user is space moderator
const isModerator = spaceMembership.role === "MODERATOR";
```

### Free vs Paid Member Detection

```typescript
import { isFreeMember } from "@/lib/member-tier";

// Check if a user is on the free tier
const isFree = await isFreeMember(userId, tenantId);
// Returns true if: joinSource === 'free_signup' AND no active subscription

// In getCurrentUser response
const user = await api.users.getCurrentUser.query();
if (user.isFreeMember) {
  // Show upgrade prompts, limit access
}
```

---

## Payments & Subscriptions

### Creating a Checkout Session

```typescript
import Stripe from "stripe";
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

// Create a subscription checkout
const session = await stripe.checkout.sessions.create({
  mode: "subscription",
  payment_method_types: ["card"],
  line_items: [{
    price: "price_monthly_49",
    quantity: 1,
  }],
  success_url: `${baseUrl}/welcome?session_id={CHECKOUT_SESSION_ID}`,
  cancel_url: `${baseUrl}/pricing`,
  metadata: {
    tenantId: tenant.id,
    userId: user.id,
    groupId: group.id,
  },
});
```

### Subscription Status Check

```typescript
// Check active subscription
const subscription = await prisma.subscription.findFirst({
  where: {
    userId: user.id,
    status: { in: ["active", "trialing"] },
  },
});

const hasAccess = !!subscription;
```

### Payment Methods Supported

| Method | Provider | Setup |
|--------|----------|-------|
| Credit/Debit Cards | Stripe | Default |
| ACH Bank Transfer | Stripe | Enable in Stripe Dashboard |
| BNPL (Afterpay/Klarna) | Stripe | Enable in Stripe Dashboard |
| Crypto (BTC, ETH, USDC) | Built-in | Configure in admin settings |

---

## Content & Posts

### Creating Posts

```typescript
// Create a post in a space
const post = await api.posts.create.mutate({
  spaceId: "space_abc123",
  title: "Welcome to the community!",
  content: "<p>This is a rich text post with <strong>formatting</strong>.</p>",
  status: "PUBLISHED", // DRAFT | PUBLISHED | SCHEDULED | ARCHIVED
});
```

### Fetching Space Content

```typescript
// List posts in a space
const posts = await api.posts.list.query({
  spaceId: "space_abc123",
  limit: 20,
  cursor: lastPostId, // for pagination
});

// Each post includes:
// - id, title, content, status
// - author (with signed profilePhotoUrl)
// - _count.Reaction (like count)
// - _count.Comment (comment count)
// - createdAt, updatedAt
```

### Space Types

```typescript
enum SpaceType {
  FEED = "FEED",             // Discussion feed
  CHAT = "CHAT",             // Real-time chat
  COURSE = "COURSE",         // Structured course
  CALCULATOR = "CALCULATOR", // Financial calculator
  DASHBOARD = "DASHBOARD",   // Data dashboard
  EVENT = "EVENT",           // Events calendar
}
```

---

## Real-Time Events

### Ably Integration

Tenant.so uses [Ably](https://ably.com) for real-time messaging — chat, notifications, and live updates.

```typescript
import Ably from "ably";

// Client-side: Subscribe to real-time updates
const ably = new Ably.Realtime({ authUrl: "/api/ably-auth" });

// Listen for new messages in a chat space
const channel = ably.channels.get(`space:${spaceId}`);
channel.subscribe("message", (message) => {
  console.log("New message:", message.data);
  // { userId, content, timestamp }
});

// Listen for notifications
const notifChannel = ably.channels.get(`user:${userId}:notifications`);
notifChannel.subscribe("notification", (notification) => {
  console.log("New notification:", notification.data);
});
```

---

## Email Integration

### Built-in Email (No External Service Needed)

Tenant.so has built-in email campaigns — no need for ConvertKit, Mailchimp, or Resend integration.

```typescript
// Sending a campaign via the admin API
const campaign = await api.campaigns.create.mutate({
  subject: "Weekly Community Update",
  content: "<h1>This Week in Our Community</h1><p>...</p>",
  audience: "all_members", // or "paid_only", "free_only", segment ID
  scheduledAt: new Date("2026-04-15T09:00:00Z"), // or null for immediate
});

// Send a test email first
await api.campaigns.sendTest.mutate({
  campaignId: campaign.id,
  testEmail: "admin@example.com",
});
```

---

## Automation Patterns

### New Member Welcome Flow

```typescript
// Triggered when a new member joins
async function onMemberJoined(userId: string, tenantId: string) {
  // 1. Send welcome email
  await sendWelcomeEmail(userId);

  // 2. Create introduction post prompt
  await createIntroductionPrompt(userId, tenantId);

  // 3. Add to relevant groups based on signup source
  const member = await prisma.membership.findFirst({
    where: { userId, tenantId },
  });

  if (member?.joinSource === "paid") {
    await addToGroup(userId, "premium-content-group");
  }

  // 4. Notify admin
  await notifyAdmin(tenantId, `New member joined: ${userId}`);
}
```

### Engagement Monitoring

```typescript
// Check for inactive members (no activity in 14 days)
const inactiveMembers = await prisma.membership.findMany({
  where: {
    tenantId,
    lastActiveAt: {
      lt: new Date(Date.now() - 14 * 24 * 60 * 60 * 1000),
    },
  },
});

// Send re-engagement email
for (const member of inactiveMembers) {
  await sendReengagementEmail(member.userId);
}
```

---

## Tech Stack

Tenant.so is built with:

| Layer | Technology |
|-------|-----------|
| **Framework** | Next.js 15 (App Router) |
| **API** | tRPC with superjson |
| **Database** | PostgreSQL (Neon) |
| **ORM** | Prisma |
| **Auth** | NextAuth.js |
| **Payments** | Stripe |
| **Real-time** | Ably |
| **Email** | Resend (transactional) + built-in campaigns |
| **Storage** | Cloudflare R2 |
| **Video** | Vimeo |
| **AI** | OpenAI |
| **Hosting** | Vercel |
| **Styling** | Tailwind CSS + shadcn/ui |

---

## More Resources

- [Tenant.so](https://tenant.so) — The Community Operating System
- [Community Platform Comparison](https://github.com/Tenant-so/community-platform-comparison)
- [Community Builder Toolkit](https://github.com/Tenant-so/community-builder-toolkit)
- [Membership Site Checklist](https://github.com/Tenant-so/membership-site-checklist)

---

<p align="center">
  <a href="https://tenant.so">
    <img src="https://tenant.so/tenant-logo.svg" alt="Tenant.so" width="40" />
  </a>
  <br />
  <sub>Built by <a href="https://tenant.so">Tenant.so</a> — The Community Operating System</sub>
</p>
