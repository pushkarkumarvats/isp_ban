# Mobile App Backend Architecture

## Overview

Mobile apps (Android/iOS) use the device's DNS resolver, not the browser's. On Jio SIM cards, the device-level DNS is also poisoned. A mobile app that hardcodes `*.supabase.co` or `*.firebase.com` will fail on Jio just like a browser.

**Solution principles**:
1. Proxy all external service calls through your own API domain
2. Use certificate pinning to protect against MITM at the proxy layer
3. Implement a dynamic endpoint configuration so you can change the proxy URL without an app update
4. Build API failover at the app level (primary → secondary → tertiary endpoint)

---

## Architecture

```
Mobile App → api.yourproduct.com (CDN-proxied, not blocked)
           → calls Supabase/Firebase internally, server-side
```

The mobile app never calls Supabase/Firebase directly. Your API is the only external network call.

---

## Solution 1: Dynamic Endpoint Configuration

Store the API endpoint URL in a remote config file, not hardcoded in the app.

```json
// https://yourproduct.com/config.json (served from CDN)
{
  "api_url": "https://api.yourproduct.com",
  "api_url_fallback": "https://api2.yourproduct.com",
  "supabase_proxy": "https://your-worker.workers.dev",
  "version": "2026.03.01"
}
```

### Flutter: Remote Config

```dart
// lib/config/remote_config.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import 'package:shared_preferences/shared_preferences.dart';

class RemoteConfig {
  static const _defaultApiUrl = 'https://api.yourproduct.com';
  static String apiUrl = _defaultApiUrl;
  static String supabaseProxy = '';

  static Future<void> load() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      
      // Try to fetch fresh config
      final response = await http
          .get(Uri.parse('https://yourproduct.com/config.json'))
          .timeout(const Duration(seconds: 5));
      
      if (response.statusCode == 200) {
        final config = jsonDecode(response.body) as Map<String, dynamic>;
        apiUrl = config['api_url'] as String? ?? _defaultApiUrl;
        supabaseProxy = config['supabase_proxy'] as String? ?? '';
        
        // Cache it
        await prefs.setString('remote_config', response.body);
      }
    } catch (e) {
      // Fall back to cached config
      final prefs = await SharedPreferences.getInstance();
      final cached = prefs.getString('remote_config');
      if (cached != null) {
        final config = jsonDecode(cached) as Map<String, dynamic>;
        apiUrl = config['api_url'] as String? ?? _defaultApiUrl;
      }
    }
  }
}
```

```dart
// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await RemoteConfig.load();
  
  await Supabase.initialize(
    url: RemoteConfig.supabaseProxy.isNotEmpty 
        ? RemoteConfig.supabaseProxy 
        : 'https://your-project.supabase.co',
    anonKey: const String.fromEnvironment('SUPABASE_ANON_KEY'),
  );
  
  runApp(const MyApp());
}
```

### React Native: Remote Config

```typescript
// src/config/remoteConfig.ts
import AsyncStorage from '@react-native-async-storage/async-storage'

interface AppConfig {
  apiUrl: string
  supabaseProxy: string
}

const DEFAULT_CONFIG: AppConfig = {
  apiUrl: 'https://api.yourproduct.com',
  supabaseProxy: '',
}

export async function loadRemoteConfig(): Promise<AppConfig> {
  try {
    const response = await fetch('https://yourproduct.com/config.json', {
      headers: { 'Cache-Control': 'no-cache' },
    })
    
    if (response.ok) {
      const config = await response.json() as AppConfig
      await AsyncStorage.setItem('app_config', JSON.stringify(config))
      return config
    }
  } catch {
    const cached = await AsyncStorage.getItem('app_config')
    if (cached) return JSON.parse(cached) as AppConfig
  }
  
  return DEFAULT_CONFIG
}
```

---

## Solution 2: API Gateway Pattern

Your mobile app talks to one API gateway. The gateway routes to the appropriate service.

```typescript
// Backend API Gateway (Node.js / FastAPI)
// All these services are called server-side   no ISP blocking

POST /auth/login       → Supabase Auth
GET  /data/posts       → Supabase REST
POST /files/upload     → Supabase Storage
GET  /notifications    → Firebase Cloud Messaging (server-side)
WS   /realtime         → Supabase Realtime proxy
```

Mobile app client:

```dart
// Flutter API client
class ApiClient {
  final String baseUrl;
  final String? authToken;

  ApiClient({required this.baseUrl, this.authToken});

  Future<Map<String, dynamic>> get(String path) async {
    final response = await http.get(
      Uri.parse('$baseUrl$path'),
      headers: {
        'Authorization': 'Bearer $authToken',
        'Content-Type': 'application/json',
      },
    );
    return jsonDecode(response.body) as Map<String, dynamic>;
  }

  Future<Map<String, dynamic>> post(String path, Map<String, dynamic> body) async {
    final response = await http.post(
      Uri.parse('$baseUrl$path'),
      headers: {
        'Authorization': 'Bearer $authToken',
        'Content-Type': 'application/json',
      },
      body: jsonEncode(body),
    );
    return jsonDecode(response.body) as Map<String, dynamic>;
  }
}
```

---

## Solution 3: Certificate Pinning

Certificate pinning ensures the mobile app only connects to your server (not to a MITM proxy). This prevents attackers from intercepting proxied traffic.

### Flutter Certificate Pinning

```dart
// pubspec.yaml
dependencies:
  http: ^1.1.0

// lib/config/http_client.dart
import 'dart:io';
import 'package:http/io_client.dart';
import 'package:http/http.dart' as http;

http.Client createPinnedClient() {
  // Your certificate fingerprint (SHA-256)
  const certFingerprint = 'aa:bb:cc:dd:...';
  
  final context = SecurityContext(withTrustedRoots: false);
  // Load your cert (embed in assets)
  context.setTrustedCertificates('assets/certs/api.yourproduct.com.pem');
  
  final httpClient = HttpClient(context: context)
    ..badCertificateCallback = (cert, host, port) {
      // Additional fingerprint check
      return false;  // Reject everything not matching our cert
    };
  
  return IOClient(httpClient);
}
```

### React Native Certificate Pinning

```typescript
import { fetch } from 'react-native-ssl-pinning'

const response = await fetch('https://api.yourproduct.com/api/posts', {
  method: 'GET',
  sslPinning: {
    certs: ['api.yourproduct.com'],  // cert stored in assets
  },
  headers: {
    Authorization: `Bearer ${token}`,
  },
})
```

---

## Solution 4: Multi-Endpoint Failover

Implement automatic failover in the app's HTTP client:

```dart
// Flutter: Multi-endpoint HTTP client
class ResilientClient {
  final List<String> endpoints;
  int _currentEndpointIndex = 0;

  ResilientClient({required this.endpoints});

  String get currentEndpoint => endpoints[_currentEndpointIndex];

  Future<http.Response> get(String path) async {
    for (int i = 0; i < endpoints.length; i++) {
      final endpoint = endpoints[(_currentEndpointIndex + i) % endpoints.length];
      
      try {
        final response = await http
            .get(Uri.parse('$endpoint$path'))
            .timeout(const Duration(seconds: 5));
        
        // Success   update current endpoint
        _currentEndpointIndex = (_currentEndpointIndex + i) % endpoints.length;
        return response;
      } catch (e) {
        if (i == endpoints.length - 1) rethrow;
        // Try next endpoint
        continue;
      }
    }
    throw Exception('All endpoints unreachable');
  }
}

// Usage
final client = ResilientClient(endpoints: [
  'https://api.yourproduct.com',
  'https://api2.yourproduct.com',
  'https://api-fallback.workers.dev',
]);
```

---

## Solution 5: Push Notifications With Backup

FCM (Firebase Cloud Messaging) requires `fcm.googleapis.com`   potentially blocked by Jio. Implement a fallback via your own WebSocket.

```dart
// Flutter: FCM with WebSocket fallback
Future<void> initNotifications() async {
  try {
    // Try FCM
    final token = await FirebaseMessaging.instance.getToken();
    await registerDeviceToken(token, 'fcm');
  } catch (e) {
    // FCM blocked   fall back to WebSocket long-polling
    final ws = await WebSocket.connect('wss://api.yourproduct.com/notifications');
    ws.listen((message) {
      showLocalNotification(jsonDecode(message));
    });
  }
}
```

---

## App Store / Play Store Considerations

If you update proxy URLs via remote config, you do NOT need to submit an app update. This is the key advantage   you can rotate proxy domains within seconds without waiting for App Store review (which can take 24-72 hours).

Ensure your remote config URL is on a domain that is highly unlikely to be blocked (`yourproduct.com` itself, served from Cloudflare).


