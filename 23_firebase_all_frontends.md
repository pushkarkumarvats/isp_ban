# Firebase With All Frontends

## The Firebase ISP Problem

Firebase services use multiple domains that can be blocked:
- `firebaseio.com`   Realtime Database
- `firestore.googleapis.com`   Firestore
- `identitytoolkit.googleapis.com`   Firebase Auth
- `storage.googleapis.com`   Firebase Storage
- `fcm.googleapis.com`   Push notifications
- `*.cloudfunctions.net`   Cloud Functions

Unlike Supabase (one domain to proxy), Firebase uses many Google-owned domains. This makes complete proxying harder but not impossible.

---

## Strategy 1: Custom Auth Domain (Quick Win for Auth)

Firebase Auth supports custom domains natively. Users authenticate via `auth.yourproduct.com` instead of `your-app.firebaseapp.com`.

```bash
# Firebase CLI
firebase hosting:channel:deploy --only hosting

# In Firebase Console:
# Authentication → Settings → Authorized Domains → Add your domain
# Hosting → Custom Domain → auth.yourproduct.com
```

```typescript
// Firebase client config
import { getAuth } from 'firebase/auth'
import { initializeApp } from 'firebase/app'

const app = initializeApp({
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: 'auth.yourproduct.com',  // Custom domain, not .firebaseapp.com
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
})

export const auth = getAuth(app)
```

---

## Strategy 2: Cloud Functions as Backend Proxy

Move all Firestore/Storage operations to Cloud Functions. Your frontend calls only your custom function URL.

```typescript
// Firebase Cloud Function (server-side   not blocked)
// functions/src/index.ts
import * as functions from 'firebase-functions'
import * as admin from 'firebase-admin'

admin.initializeApp()
const db = admin.firestore()

export const getPosts = functions.region('asia-south1').https.onCall(
  async (data, context) => {
    if (!context.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'Login required')
    }

    const snapshot = await db.collection('posts')
      .orderBy('createdAt', 'desc')
      .limit(20)
      .get()

    return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }))
  }
)
```

### Proxy CF Function URL through Cloudflare

Cloud Function URLs are `*.cloudfunctions.net`   potentially blocked. Route through Cloudflare:

```javascript
// Cloudflare Worker for Cloud Functions
export default {
  async fetch(request, env) {
    const url = new URL(request.url)
    url.hostname = env.CF_FUNCTIONS_HOST  // e.g. asia-south1-your-project.cloudfunctions.net
    
    const newReq = new Request(url.toString(), request)
    newReq.headers.set('host', url.hostname)
    return fetch(newReq)
  }
}
```

---

## Strategy 3: Firebase Hosting as CDN

Firebase Hosting (https://your-app.web.app or your custom domain) uses Google's CDN. Static files served from Firebase Hosting are rarely blocked. Use Firebase Hosting for the frontend.

```bash
# firebase.json
{
  "hosting": {
    "public": "dist",
    "rewrites": [
      {
        "source": "/api/**",
        "function": "api"  // Cloud Function
      }
    ]
  }
}
```

---

## Strategy 4: Next.js + Firebase Admin (Server-Side)

Never call Firebase from the browser. Use Firebase Admin SDK in Next.js API routes.

```typescript
// lib/firebaseAdmin.ts
import { initializeApp, getApps, cert } from 'firebase-admin/app'
import { getFirestore } from 'firebase-admin/firestore'

if (!getApps().length) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    }),
  })
}

export const adminDb = getFirestore()
```

```typescript
// app/api/posts/route.ts
import { adminDb } from '@/lib/firebaseAdmin'

export async function GET() {
  // Server-side Firebase call   no ISP blocking
  const snapshot = await adminDb.collection('posts')
    .orderBy('createdAt', 'desc')
    .limit(20)
    .get()

  const posts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }))
  return Response.json(posts)
}
```

---

## Strategy 5: React Native + Firebase (Mobile)

For React Native apps, device-level DNS is used (not browser). Jio SIM cards may have forced DNS. Recommended: Use Firebase in server-side mode in a backend that mobile app calls.

```typescript
// Mobile app: calls your API, not Firebase directly
const API_BASE = 'https://api.yourproduct.com'

async function getPosts() {
  return fetch(`${API_BASE}/api/posts`).then(r => r.json())
}
```

---

## Strategy 6: Firebase Alternative   Supabase

If Firebase blocking becomes a persistent problem, Supabase is a self-hostable Firebase alternative with a proven bypass solution. Migration tools exist:

```bash
# Firestore → Supabase migration
npm install -g firebase-to-supabase
firebase-to-supabase migrate --project your-project-id --output ./migration
```

---

## Firebase Emulator for Development (Bypass DNS Issues in Dev)

During development, run Firebase emulators locally. Your browser connects to `localhost`   no ISP interference.

```typescript
// lib/firebase.ts
import { connectAuthEmulator, getAuth } from 'firebase/auth'
import { connectFirestoreEmulator, getFirestore } from 'firebase/firestore'

const app = initializeApp(firebaseConfig)
const auth = getAuth(app)
const db = getFirestore(app)

if (process.env.NODE_ENV === 'development') {
  connectAuthEmulator(auth, 'http://localhost:9099')
  connectFirestoreEmulator(db, 'localhost', 8080)
}

export { auth, db }
```

---

## Diagnosis: Is Firebase Blocked?

```javascript
// Test Firebase connectivity
async function testFirebaseConnectivity() {
  const services = [
    'https://identitytoolkit.googleapis.com',   // Auth
    'https://firestore.googleapis.com',          // Firestore
    'https://storage.googleapis.com',            // Storage
  ]

  for (const url of services) {
    try {
      await fetch(url, { method: 'HEAD', mode: 'no-cors' })
      console.log(`✓ ${url}`)
    } catch {
      console.error(`�  BLOCKED: ${url}`)
    }
  }
}
```


