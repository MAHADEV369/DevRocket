# Expo (React Native) Build Instructions

> Complete guide to building a mobile app with Expo that connects to your backend API. Works for iOS and Android from a single TypeScript/React codebase.

---

## What You Get

- One codebase for iOS + Android
- Hot reload development (scan QR code with phone)
- Over-the-air (OTA) updates without app store review
- Cloud build service (EAS) — no Mac needed for Android builds
- Push notifications, camera, offline caching
- Shares React/TypeScript knowledge with your web frontend

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Node.js | 18+ | nodejs.org |
| Expo CLI | Latest | `npm install -g expo-cli` |
| Expo Go | Latest | App Store / Play Store (for development) |
| EAS CLI | Latest | `npm install -g eas-cli` |
| Android Studio | Latest | For Android emulator (optional) |
| Xcode | 15+ | Mac only, for iOS emulator |

## Phase 1: Project Setup

### Create Project

```bash
npx create-expo-app genai-mobile --template expo-template-blank-typescript
cd genai-mobile
```

### Install Dependencies

```bash
# Navigation
npx expo install expo-router react-native-screens react-native-safe-area-context

# HTTP & State
npm install axios @tanstack/react-query zustand

# Auth & Secure Storage
npx expo install expo-secure-store expo-auth-session expo-random expo-crypto

# UI
npx expo install expo-image expo-linear-gradient expo-blur expo-status-bar expo-system-ui
npm install react-native-paper

# Push Notifications
npx expo install expo-notifications expo-device expo-constants

# Camera & Image Picker
npx expo install expo-camera expo-image-picker

# Offline Detection
npx expo install @react-native-async-storage/async-storage @react-native-community/netinfo

# Fonts
npx expo install expo-font

# Splash Screen
npx expo install expo-splash-screen

# Linking (for OAuth deep links)
npx expo install expo-linking

# Dev tools
npm install -D @types/react @types/react-native
```

### Configure TypeScript

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "paths": {
      "@/*": ["./src/*"]
    },
    "baseUrl": "."
  },
  "include": ["**/*.ts", "**/*.tsx", ".expo/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### Configure app.json

See `skills/25-expo-react-native.md` for the complete app.json with iOS/Android config, permissions, and plugins.

## Phase 2: Project Structure

```
genai-mobile/
├── App.tsx                           ← Entrypoint with providers
├── app.json
├── eas.json                          ← EAS Build config
├── babel.config.js
├── tsconfig.json
├── .env.example
├── assets/
│   ├── icon.png                      ← 1024x1024
│   ├── splash.png                    ← 1242x2436
│   ├── adaptive-icon.png            ← 1024x1024 (Android)
│   └── notification-icon.png        ← 96x96 white silhouette
├── src/
│   ├── api/
│   │   ├── client.ts                ← Axios instance with auth interceptors
│   │   ├── auth.api.ts              ← Login, signup, refresh, logout
│   │   ├── images.api.ts            ← Generate, list, delete, favorite
│   │   ├── gallery.api.ts           ← Gallery CRUD, share
│   │   └── types.ts                 ← TypeScript interfaces for API responses
│   ├── auth/
│   │   ├── AuthContext.tsx          ← React Context for auth state
│   │   ├── useAuth.ts              ← Hook to access auth state
│   │   ├── SecureStorage.ts         ← Expo SecureStore wrapper
│   │   └── ProtectedRoute.tsx       ← Redirect to login if not authenticated
│   ├── screens/
│   │   ├── LoginScreen.tsx
│   │   ├── SignupScreen.tsx
│   │   ├── ForgotPasswordScreen.tsx
│   │   ├── HomeScreen.tsx           ← Image generation (main screen)
│   │   ├── GalleryScreen.tsx        ← User's saved images
│   │   ├── ImageDetailScreen.tsx    ← Full view + actions
│   │   ├── ExploreScreen.tsx        ← Public gallery
│   │   └── SettingsScreen.tsx        ← Theme, account, plan, logout
│   ├── components/
│   │   ├── PromptInput.tsx          ← Text input for image prompts
│   │   ├── ModelSelector.tsx        ← Horizontal scroll of AI models
│   │   ├── StyleSelector.tsx        ← Horizontal scroll of style presets
│   │   ├── SizeSelector.tsx         ← Image size options
│   │   ├── ImageCard.tsx            ← Thumbnail with overlay info
│   │   ├── LoadingOverlay.tsx       ← Full-screen loading spinner
│   │   ├── ErrorView.tsx            ← Error display with retry button
│   │   ├── EmptyState.tsx           ← "No images yet" placeholder
│   │   └── OfflineBanner.tsx        ← Shows when device is offline
│   ├── navigation/
│   │   ├── AppNavigator.tsx         ← Root: switches between Auth and Main
│   │   ├── AuthNavigator.tsx         ← Login/Signup stack
│   │   └── MainNavigator.tsx         ← Bottom tab navigator
│   ├── stores/
│   │   ├── authStore.ts             ← Zustand: user, tokens, login/logout
│   │   ├── generationStore.ts       ← Zustand: prompt, model, style, result
│   │   └── themeStore.ts            ← Zustand: dark/light mode
│   ├── hooks/
│   │   ├── useImages.ts             ← TanStack Query: image list, generate, delete
│   │   ├── useGallery.ts            ← TanStack Query: gallery CRUD
│   │   └── useOffline.ts            ← NetInfo: is device online?
│   ├── utils/
│   │   ├── constants.ts             ← API URLs, colors, spacing
│   │   ├── formatters.ts            ← Currency, date, relative time
│   │   └── pushNotifications.ts     ← Register for push, handle incoming
│   └── theme/
│       ├── colors.ts                 ← Color palette (light + dark)
│       ├── typography.ts             ← Font sizes and weights
│       └── spacing.ts               ← Spacing scale (4, 8, 12, 16, 24, 32)
└── google-services.json             ← Firebase config (Android)
```

## Phase 3: API Client

The critical piece — this connects your mobile app to your backend:

```typescript
// src/api/client.ts
import axios from 'axios';
import * as SecureStore from 'expo-secure-store';

// Android emulator uses 10.0.2.2 for host machine
// iOS simulator uses localhost
// Production uses your real API URL
const API_BASE_URL = process.env.EXPO_PUBLIC_API_URL || 'http://10.0.2.2:3001/api/v1';

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 60000, // 60s for image generation
  headers: { 'Content-Type': 'application/json' },
});

// REQUEST INTERCEPTOR: Attach access token
apiClient.interceptors.request.use(async (config) => {
  const token = await SecureStore.getItemAsync('access_token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// RESPONSE INTERCEPTOR: Handle 401 → refresh → retry
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      try {
        const refreshToken = await SecureStore.getItemAsync('refresh_token');
        if (!refreshToken) throw new Error('No refresh token');
        const { data } = await axios.post(`${API_BASE_URL}/auth/refresh`, { refreshToken });
        await SecureStore.setItemAsync('access_token', data.accessToken);
        await SecureStore.setItemAsync('refresh_token', data.refreshToken);
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        await SecureStore.deleteItemAsync('access_token');
        await SecureStore.deleteItemAsync('refresh_token');
        // Redirect to login (use your navigation method)
        return Promise.reject(refreshError);
      }
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

## Phase 4: Auth Flow

### Secure Storage for Tokens

```typescript
// src/auth/SecureStorage.ts
import * as SecureStore from 'expo-secure-store';

export const tokenStorage = {
  async getAccessToken(): Promise<string | null> {
    return SecureStore.getItemAsync('access_token');
  },
  async setAccessToken(token: string): Promise<void> {
    return SecureStore.setItemAsync('access_token', token);
  },
  async getRefreshToken(): Promise<string | null> {
    return SecureStore.getItemAsync('refresh_token');
  },
  async setRefreshToken(token: string): Promise<void> {
    return SecureStore.setItemAsync('refresh_token', token);
  },
  async clearAll(): Promise<void> {
    await SecureStore.deleteItemAsync('access_token');
    await SecureStore.deleteItemAsync('refresh_token');
  },
};
```

### Auth Context

See `skills/25-expo-react-native.md` Step 5 for the complete AuthContext with login, signup, logout, and auto-refresh.

### Key Pattern: Authenticated Navigation

```typescript
// src/navigation/AppNavigator.tsx
import { useAuth } from '../auth/AuthContext';

export function AppNavigator() {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) return <LoadingScreen />;
  return isAuthenticated ? <MainNavigator /> : <AuthNavigator />;
}
```

## Phase 5: Screens

### Home Screen (Image Generation)

See `skills/25-expo-react-native.md` Step 7 for the complete HomeScreen with prompt input, model selector, style selector, and image display.

### Gallery Screen

```typescript
// src/screens/GalleryScreen.tsx (skeleton)
export function GalleryScreen() {
  const { data, isLoading, error } = useImages(1);

  if (isLoading) return <LoadingOverlay />;
  if (error) return <ErrorView message={error.message} onRetry={() => refetch()} />;
  if (!data?.images.length) return <EmptyState message="No images yet. Generate your first one!" />;

  return (
    <FlatList
      data={data.images}
      numColumns={2}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <ImageCard image={item} onPress={() => navigate('ImageDetail', { id: item.id })} />}
      onEndReached={() => fetchNextPage()}
      onEndReachedThreshold={0.5}
    />
  );
}
```

### Settings Screen

```typescript
// src/screens/SettingsScreen.tsx (skeleton)
export function SettingsScreen() {
  const { user, logout } = useAuth();
  const theme = useThemeStore((s) => s.theme);
  const setTheme = useThemeStore((s) => s.setTheme);

  return (
    <ScrollView>
      <List.Section>
        <List.Subheader>Account</List.Subheader>
        <List.Item title={user?.email} description={user?.plan} left={() => <List.Icon icon="account" />} />
        <List.Item title="Logout" onPress={logout} left={() => <List.Icon icon="logout" />} />
      </List.Section>

      <List.Section>
        <List.Subheader>Appearance</List.Subheader>
        <List.Item
          title="Theme"
          description={theme}
          onPress={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
          left={() => <List.Icon icon="theme-light-dark" />}
        />
      </List.Section>

      <List.Section>
        <List.Subheader>About</List.Subheader>
        <List.Item title="Version" description="1.0.0" />
      </List.Section>
    </ScrollView>
  );
}
```

## Phase 6: Offline Support

### Network Detection

```typescript
// src/hooks/useOffline.ts
import { useState, useEffect } from 'react';
import NetInfo from '@react-native-community/netinfo';

export function useOffline() {
  const [isOffline, setIsOffline] = useState(false);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsOffline(!state.isConnected);
    });
    return () => unsubscribe();
  }, []);

  return isOffline;
}
```

### Offline Banner Component

```typescript
// src/components/OfflineBanner.tsx
export function OfflineBanner() {
  const isOffline = useOffline();
  if (!isOffline) return null;
  return (
    <Banner visible icon="wifi-off" style={{ backgroundColor: '#FF6B6B' }}>
      You're offline. Some features may be unavailable.
    </Banner>
  );
}
```

### Image Caching with Async Storage

```typescript
// src/hooks/useImages.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { imagesApi } from '../api/images.api';
import AsyncStorage from '@react-native-async-storage/async-storage';

export function useImages(page: number = 1) {
  return useQuery({
    queryKey: ['images', page],
    queryFn: async () => {
      const data = await imagesApi.list(page);
      // Cache for offline access
      await AsyncStorage.setItem(`images_page_${page}`, JSON.stringify(data));
      return data;
    },
    // Show cached data when offline
    placeholderData: async () => {
      const cached = await AsyncStorage.getItem(`images_page_${page}`);
      return cached ? JSON.parse(cached) : undefined;
    },
  });
}
```

## Phase 7: Push Notifications

See `skills/25-expo-react-native.md` Step 8 for the complete push notification registration and handling code.

## Phase 8: Theming

```typescript
// src/theme/colors.ts
export const lightColors = {
  primary: '#6C63FF',
  primaryDark: '#5A52D5',
  background: '#FFFFFF',
  surface: '#F5F5F5',
  text: '#1A1A2E',
  textSecondary: '#6B7280',
  error: '#EF4444',
  success: '#10B981',
  border: '#E5E7EB',
};

export const darkColors = {
  primary: '#6C63FF',
  primaryDark: '#8B83FF',
  background: '#0F0F23',
  surface: '#1A1A2E',
  text: '#E5E7EB',
  textSecondary: '#9CA3AF',
  error: '#EF4444',
  success: '#10B981',
  border: '#374151',
};
```

## Phase 9: Build & Release

### Development

```bash
npx expo start
# Scan QR code with Expo Go app on your phone
# Or press 'i' for iOS simulator, 'a' for Android emulator
```

### EAS Build for Production

```bash
# Install EAS CLI
npm install -g eas-cli

# Login to Expo
eas login

# Configure build
eas build:configure

# Build Android APK (for testing)
eas build --profile preview --platform android

# Build Android AAB (for Play Store)
eas build --profile production --platform android

# Build iOS (requires Apple Developer account)
eas build --profile production --platform ios

# Submit to stores
eas submit --platform android   # Google Play
eas submit --platform ios       # App Store
```

### App Store Preparation

**Google Play ($25 one-time):**
1. Create developer account at play.google.com/console
2. Create app listing with screenshots (phone + tablet)
3. Upload AAB from EAS Build
4. Content rating questionnaire
5. Data safety declaration
6. Submit for review (2-7 days)

**Apple App Store ($99/year):**
1. Create Apple Developer account at developer.apple.com
2. Create app in App Store Connect
3. EAS Submit handles build upload
4. App Review guidelines compliance
5. Submit for review (1-3 days)

### OTA Updates

```bash
# Publish an OTA update (JS changes only, no native code)
eas update --branch production --message "Bug fixes and improvements"

# Users get the update next time they open the app
# No app store review needed for JS changes
```

## Phase 10: Environment Configuration

### .env files

```env
# .env.example
EXPO_PUBLIC_API_URL=http://10.0.2.2:3001/api/v1
EXPO_PUBLIC_SENTRY_DSN=
```

### Per-environment URLs

| Environment | Android Emulator | iOS Simulator | Real Device |
|---|---|---|---|
| Development | `http://10.0.2.2:3001/api/v1` | `http://localhost:3001/api/v1` | `http://YOUR_IP:3001/api/v1` |
| Staging | `https://api-staging.yourdomain.com/api/v1` | same | same |
| Production | `https://api.yourdomain.com/api/v1` | same | same |

## Common Pitfalls

| Problem | Solution |
|---|---|
| "Network request failed" on Android emulator | Use `10.0.2.2` instead of `localhost` |
| "Network request failed" on real device | Use your computer's local IP (find with `ifconfig` or `ipconfig`) |
| iOS push notifications don't work in simulator | Push notifications only work on real iOS devices |
| App crashes on launch | Check `npx expo start --clear` to reset cache |
| Build fails on EAS | Run `eas build --profile development --platform android` first for dev build |
| Images not loading | Check CORS settings on your backend — allow your app's origin |
| SecureStore items disappear | Expo Go doesn't persist SecureStore — test on dev build |

## Expo vs Flutter Decision Guide

| Factor | Expo (React Native) | Flutter |
|---|---|---|
| Language | TypeScript/React | Dart |
| Learning curve | Low if you know React | Medium (new language + paradigm) |
| Hot reload | Yes | Yes |
| Performance | Good (native bridge) | Excellent (compiled to native) |
| OTA updates | Yes (no review needed) | No (requires app store update) |
| Native modules | Many via Expo, some need dev client | Available via pub.dev |
| Web support | Yes (Expo Web) | Yes (Flutter Web) |
| Desktop support | Limited (React Native Windows/macOS) | Yes (Flutter desktop) |
| CI/CD | EAS Build (cloud, no Mac needed for Android) | Need Mac for iOS builds |
| Community | Very large (React ecosystem) | Large and growing |
| Best for | Apps where development speed matters, web team knows React | Apps where performance and custom UI matter, team willing to learn Dart |