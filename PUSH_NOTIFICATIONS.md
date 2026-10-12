# Push Notifications Implementation

This document explains how push notifications are implemented in IssueFlow using the Web Push API.

## Architecture Overview

Push notifications in IssueFlow follow the standard Web Push protocol:

1. **Client** (browser) subscribes to push notifications and receives a subscription object
2. **Frontend** sends subscription to the **Backend** for storage
3. **Backend** stores subscriptions in PostgreSQL
4. When an event occurs, **Backend** retrieves subscriptions and sends push notifications via web-push
5. **Service Worker** receives push events and displays native browser notifications

## Technology Stack

- **Frontend**: React, Vite, Workbox (service worker)
- **Backend**: NestJS, Prisma, web-push
- **Database**: PostgreSQL
- **Protocol**: Web Push (VAPID)

---

## Frontend Implementation

### 1. Service Worker (`apps/web/src/sw.js`)

The service worker handles incoming push events and notification clicks:

```typescript
self.addEventListener('push', function (event) {
  const data = event.data.json();
  
  self.registration.showNotification(data.title, {
    body: data.body,
    icon: '/android-chrome-192x192.png',
    badge: '/favicon-32x32.png',
    vibrate: [200, 100, 200],
    data: { url: data.url || '/' },
  });
});

self.addEventListener('notificationclick', function (event) {
  event.notification.close();
  const targetUrl = event.notification.data.url || '/';
  
  // Focus existing window or open new one
  clients.matchAll({ type: 'window' }).then(/* ... */);
});
```

Key features:
- Parses JSON payload from push event
- Shows native browser notification with icon and badge
- Handles notification click to navigate to the relevant URL

### 2. Push Notifications Hook (`apps/web/src/hooks/usePushNotifications.ts`)

The `usePushNotifications` hook manages the subscription lifecycle:

```typescript
export function usePushNotifications() {
  // Detect browser support
  if ('serviceWorker' in navigator && 'PushManager' in window) {
    // ...
  }

  // Subscribe to push notifications
  const subscribe = async () => {
    // Request permission
    const permission = await Notification.requestPermission();
    
    // Subscribe with VAPID public key
    const sub = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(publicVapidKey)
    });
    
    // Send subscription to backend
    await api.post('/notifications/push/subscribe', sub.toJSON());
  };

  // Unsubscribe
  const unsubscribe = async () => {
    await api.delete('/notifications/push/unsubscribe', {
      data: { endpoint: subscription.endpoint }
    });
    await subscription.unsubscribe();
  };
}
```

Key functions:
- `urlBase64ToUint8Array()`: Converts VAPID key from base64 to Uint8Array for the Push API
- `subscribe()`: Requests permission, subscribes to PushManager, stores subscription on backend
- `unsubscribe()`: Removes subscription from backend and browser

### 3. UI Component (`apps/web/src/components/PushNotificationManager.tsx`)

A simple UI component that shows:
- "Push Enabled" status with disable button (when subscribed)
- "Enable Push Alerts" button (when not subscribed)
- Error state when permission is denied
- Loading state while checking

---

## Backend Implementation

### 1. Push Service (`apps/api/src/modules/notifications/push.service.ts`)

The `PushService` handles subscription storage and sending notifications:

```typescript
@Injectable()
export class PushService {
  constructor(private prisma: PrismaService, private config: ConfigService) {
    // Configure VAPID keys for web-push
    webpush.setVapidDetails(
      subject,
      config.get('VAPID_PUBLIC_KEY'),
      config.get('VAPID_PRIVATE_KEY')
    );
  }

  // Save subscription to database
  async saveSubscription(userId: string, subscription: any) {
    return this.prisma.pushSubscription.create({
      data: {
        userId,
        endpoint: subscription.endpoint,
        p256dh: subscription.keys.p256dh,
        auth: subscription.keys.auth,
      },
    });
  }

  // Send push notification
  async sendPushNotification(userId: string, payload: any) {
    const subscriptions = await this.prisma.pushSubscription.findMany({
      where: { userId },
    });

    for (const sub of subscriptions) {
      try {
        await webpush.sendNotification(pushSubscription, JSON.stringify(payload));
      } catch (error) {
        if (error.statusCode === 410 || error.statusCode === 404) {
          // Remove expired subscription
          await this.removeSubscription(sub.endpoint);
        }
      }
    }
  }
}
```

Key features:
- Configures VAPID keys from environment variables
- Saves subscriptions with encryption keys (p256dh, auth)
- Sends push notifications using web-push
- Automatically removes expired/invalid subscriptions (410 Gone)

### 2. API Endpoints (`apps/api/src/modules/notifications/notifications.controller.ts`)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/notifications/push/subscribe` | POST | Save push subscription |
| `/notifications/push/unsubscribe` | DELETE | Remove push subscription |

---

## Database Schema

```prisma
model PushSubscription {
  id        String   @id @default(uuid())
  userId    String
  endpoint  String   @unique
  p256dh    String   // Encryption key (ECDH)
  auth      String   // Authentication secret
  createdAt DateTime @default(now())

  @@index([userId])
}
```

The subscription stores:
- `endpoint`: The push service URL
- `p256dh`: ECDH public key for encryption
- `auth`: Authentication secret

---

## Environment Variables

### Frontend (`apps/web/.env`)

```
VITE_VAPID_PUBLIC_KEY=BPc253jx...
```

### Backend (`apps/api/.env`)

```
VAPID_PUBLIC_KEY=BPc253jx...
VAPID_PRIVATE_KEY=-cQ253jx...
VAPID_SUBJECT=mailto:admin@issueflow.com
```

The VAPID keys are generated once using the `web-push` CLI:

```bash
npx web-push generateVapidKeys
```

---

## Usage Flow

1. **User visits dashboard** - The `PushNotificationManager` component checks if push is supported
2. **User clicks "Enable Push Alerts"** - The hook requests notification permission
3. **Browser generates subscription** - Include VAPID public key
4. **Subscription sent to backend** - Stored in PostgreSQL
5. **Event triggers notification** (e.g., user mentioned in comment):
   - Backend calls `pushService.sendPushNotification(userId, payload)`
   - Push payload: `{ title: 'New Mention', body: 'You were mentioned in...', url: '/issues/123' }`
5. **Service worker shows notification** - Native browser notification with click handler

---

## Security Considerations

- VAPID keys authenticate the server to push services
- Only one subscription per endpoint (unique constraint)
- Expired subscriptions removed automatically (410 handling)
- Subscription scoped to user and organization

---

## Dependencies

### Frontend
- `workbox-precaching` (service worker)
- `vite-plugin-pwa` (PWA support)

### Backend
- `web-push` (sending push notifications)
- `@nestjs/config` (environment config)