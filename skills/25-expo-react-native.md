# Skill 25: Expo (React Native) Mobile App

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 4-6 hours
Depends On: 03-auth, 05-images-crud

## Input Contract
- ✅ Backend API running at a known URL (skill 01-05 complete)
- ✅ Auth endpoints working (/api/v1/auth/signup, /login, /refresh)
- ✅ Image generation endpoint working (/api/v1/images/generate)
- ✅ Node.js 18+ installed
- ✅ Expo CLI installed (`npm install -g expo-cli`)
- ✅ iOS Simulator (Mac only) or Android Emulator set up

## Output Contract
- 📁 Complete Expo app project with TypeScript
- 📁 Feature-based folder structure
- 📁 Auth flow (login, signup, refresh token handling)
- 📁 Image generation flow
- 📁 Gallery with offline caching
- 📁 Push notifications configured
- 📁 EAS Build configuration for release
- 🔌 App connects to your backend API

## Files to Create

```
app/
├── App.tsx                           ← Entry point with providers
├── app.json                          ← Expo config
├── babel.config.js                   ← Babel config with path aliases
├── tsconfig.json                     ← TypeScript strict config
├── eas.json                          ← EAS Build configuration
├── .env                              ← Environment variables
├── .env.example                      ← Environment template
├── package.json                      ← Dependencies
├── assets/
│   ├── icon.png                      ← 1024x1024 app icon
│   ├── splash.png                    ← Splash screen
│   └── adaptive-icon.png            ← Android adaptive icon
├── src/
│   ├── api/
│   │   ├── client.ts                 ← Axios instance with interceptors
│   │   ├── auth.api.ts              ← Login, signup, refresh, logout
│   │   ├── images.api.ts            ← Generate, list, delete, favorite
│   │   ├── gallery.api.ts           ← Gallery CRUD, share
│   │   └── types.ts                 ← API response types
│   ├── auth/
│   │   ├── AuthContext.tsx           ← Auth state provider
│   │   ├── useAuth.ts               ← Auth hook
│   │   ├── SecureStorage.ts          ← Token storage (SecureStore)
│   │   └── ProtectedRoute.tsx       ← Auth guard component
│   ├── screens/
│   │   ├── LoginScreen.tsx
│   │   ├── SignupScreen.tsx
│   │   ├── HomeScreen.tsx           ← Image generation
│   │   ├── GalleryScreen.tsx        ← Saved images grid
│   │   ├── ImageDetailScreen.tsx    ← Full view + actions
│   │   ├── SettingsScreen.tsx       ← Theme, account, plan
│   │   └── ExploreScreen.tsx        ← Public gallery
│   ├── components/
│   │   ├── PromptInput.tsx
│   │   ├── ModelSelector.tsx
│   │   ├── StyleSelector.tsx
│   │   ├── ImageCard.tsx
│   │   ├── LoadingOverlay.tsx
│   │   └── ErrorView.tsx
│   ├── navigation/
│   │   ├── AppNavigator.tsx         ← Root navigation
│   │   ├── AuthNavigator.tsx        ← Login/signup stack
│   │   └── MainNavigator.tsx        ← Tab navigator
│   ├── stores/
│   │   ├── authStore.ts             ← Zustand auth state
│   │   ├── generationStore.ts       ← Zustand generation state
│   │   └── themeStore.ts            ← Dark/light mode
│   ├── hooks/
│   │   ├── useImages.ts             ← TanStack Query hook
│   │   ├── useGallery.ts
│   │   └── useOffline.ts            ← NetInfo offline detection
│   ├── utils/
│   │   ├── constants.ts             ← API URLs, colors, sizes
│   │   ├── formatters.ts            ← Currency, date formatting
│   │   └── permissions.ts           ← Camera, notifications
│   └── theme/
│       ├── colors.ts                 ← Color palette
│       ├── typography.ts             ← Font sizes/weights
│       └── spacing.ts               ← Spacing scale
└── google-services.json             ← Firebase config (Android)
```

## Steps

### Step 1: Create Expo Project

```bash
npx create-expo-app genai-mobile --template expo-template-blank-typescript
cd genai-mobile
```

### Step 2: Install Dependencies

```bash
# Navigation
npx expo install expo-router react-native-screens react-native-safe-area-context

# Auth & Storage
npx expo install expo-secure-store expo-auth-session expo-random

# HTTP & State
npm install axios @tanstack/react-query zustand

# UI
npx expo install expo-image expo-linear-gradient expo-blur
npm install react-native-paper  # Material Design components

# Push Notifications
npx expo install expo-notifications expo-device expo-constants

# Camera (for receipt photos)
npx expo install expo-camera expo-image-picker

# Offline & Cache
npx expo install @react-native-async-storage/async-storage @react-native-community/netinfo

# Fonts
npx expo install expo-font

# EAS Build
npm install -g eas-cli
eas login
eas build:configure
```

### Step 3: Configure app.json

```json
{
  "expo": {
    "name": "GenAI",
    "slug": "genai",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "automatic",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#0F0F23"
    },
    "ios": {
      "bundleIdentifier": "com.genai.app",
      "buildNumber": "1",
      "supportsTablet": true,
      "infoPlist": {
        "NSCameraUsageDescription": "GenAI needs camera access to scan receipts",
        "NSPhotoLibraryUsageDescription": "GenAI needs photo access to attach images"
      }
    },
    "android": {
      "package": "com.genai.app",
      "versionCode": 1,
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#0F0F23"
      },
      "permissions": [
        "CAMERA",
        "NOTIFICATIONS"
      ]
    },
    "plugins": [
      "expo-router",
      [
        "expo-notifications",
        {
          "icon": "./assets/notification-icon.png",
          "color": "#6C63FF"
        }
      ]
    ],
    "scheme": "genai",
    "extra": {
      "eas": {
        "projectId": "YOUR_PROJECT_ID"
      }
    }
  }
}
```

### Step 4: Create API Client

```typescript
// src/api/client.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import * as SecureStore from 'expo-secure-store';
import { router } from 'expo-router';

const API_BASE_URL = process.env.EXPO_PUBLIC_API_URL || 'http://10.0.2.2:3001/api/v1';
// 10.0.2.2 = Android emulator's host machine
// Use your actual API URL in production

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 60000,
  headers: { 'Content-Type': 'application/json' },
});

// Request interceptor: attach access token
apiClient.interceptors.request.use(async (config: InternalAxiosRequestConfig) => {
  const token = await SecureStore.getItemAsync('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor: handle 401 → refresh token → retry
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as any;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = await SecureStore.getItemAsync('refresh_token');
        if (!refreshToken) {
          // No refresh token → force login
          await SecureStore.deleteItemAsync('access_token');
          await SecureStore.deleteItemAsync('refresh_token');
          router.replace('/login');
          return Promise.reject(error);
        }

        const { data } = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refreshToken,
        });

        await SecureStore.setItemAsync('access_token', data.accessToken);
        await SecureStore.setItemAsync('refresh_token', data.refreshToken);

        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        await SecureStore.deleteItemAsync('access_token');
        await SecureStore.deleteItemAsync('refresh_token');
        router.replace('/login');
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

### Step 5: Create Auth Context

```typescript
// src/auth/AuthContext.tsx
import React, { createContext, useCallback, useEffect, useState } from 'react';
import * as SecureStore from 'expo-secure-store';
import authApi from '../api/auth.api';

interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  plan: string;
  emailVerified: boolean;
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  signup: (email: string, password: string, name: string) => Promise<void>;
  logout: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextType>(null!);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const loadUser = useCallback(async () => {
    try {
      const token = await SecureStore.getItemAsync('access_token');
      if (!token) { setUser(null); return; }
      const { data } = await authApi.me();
      setUser(data.user);
    } catch {
      setUser(null);
      await SecureStore.deleteItemAsync('access_token');
      await SecureStore.deleteItemAsync('refresh_token');
    } finally {
      setIsLoading(false);
    }
  }, []);

  useEffect(() => { loadUser(); }, [loadUser]);

  const login = async (email: string, password: string) => {
    const { data } = await authApi.login({ email, password });
    await SecureStore.setItemAsync('access_token', data.accessToken);
    await SecureStore.setItemAsync('refresh_token', data.refreshToken);
    setUser(data.user);
  };

  const signup = async (email: string, password: string, name: string) => {
    const { data } = await authApi.signup({ email, password, name });
    await SecureStore.setItemAsync('access_token', data.accessToken);
    await SecureStore.setItemAsync('access_token', data.refreshToken);
    setUser(data.user);
  };

  const logout = async () => {
    const refreshToken = await SecureStore.getItemAsync('refresh_token');
    await authApi.logout(refreshToken || undefined).catch(() => {});
    await SecureStore.deleteItemAsync('access_token');
    await SecureStore.deleteItemAsync('refresh_token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, isLoading, isAuthenticated: !!user, login, signup, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### Step 6: Create Navigation Structure

```typescript
// src/navigation/AppNavigator.tsx
import { useAuth } from '../auth/AuthContext';
import { ActivityIndicator, View } from 'react-native';
import { AuthNavigator } from './AuthNavigator';
import { MainNavigator } from './MainNavigator';

export function AppNavigator() {
  const { isAuthenticated, isLoading } = useAuth();

  if (isLoading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" color="#6C63FF" />
      </View>
    );
  }

  return isAuthenticated ? <MainNavigator /> : <AuthNavigator />;
}
```

```typescript
// src/navigation/MainNavigator.tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { HomeScreen } from '../screens/HomeScreen';
import { GalleryScreen } from '../screens/GalleryScreen';
import { ExploreScreen } from '../screens/ExploreScreen';
import { SettingsScreen } from '../screens/SettingsScreen';

const Tab = createBottomTabNavigator();

export function MainNavigator() {
  return (
    <Tab.Navigator screenOptions={{ headerShown: true }}>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Gallery" component={GalleryScreen} />
      <Tab.Screen name="Explore" component={ExploreScreen} />
      <Tab.Screen name="Settings" component={SettingsScreen} />
    </Tab.Navigator>
  );
}
```

### Step 7: Create Image Generation Screen

```typescript
// src/screens/HomeScreen.tsx
import React, { useState } from 'react';
import { View, ScrollView, KeyboardAvoidingView, Platform } from 'react-native';
import { Button, TextInput, Card, Title, Paragraph, ActivityIndicator } from 'react-native-paper';
import { useMutation } from '@tanstack/react-query';
import { imagesApi } from '../api/images.api';
import { useGenerationStore } from '../stores/generationStore';

const MODELS = [
  { id: 'pollinations-ai', name: 'Pollinations (Free)', icon: '🎨' },
  { id: 'dall-e-3', name: 'DALL-E 3', icon: '🖼️' },
  { id: 'flux-pro', name: 'Flux Pro', icon: '✨' },
];

const STYLES = [
  { id: 'none', name: 'None' },
  { id: 'realistic', name: 'Realistic' },
  { id: 'artistic', name: 'Artistic' },
  { id: 'anime', name: 'Anime' },
  { id: '3d-render', name: '3D Render' },
  { id: 'sketch', name: 'Sketch' },
  { id: 'abstract', name: 'Abstract' },
];

export function HomeScreen() {
  const [prompt, setPrompt] = useState('');
  const { model, style, setModel, setStyle } = useGenerationStore();
  const [resultImage, setResultImage] = useState<string | null>(null);

  const generateMutation = useMutation({
    mutationFn: () => imagesApi.generate({ prompt, model, style }),
    onSuccess: (data) => {
      setResultImage(data.image.imageUrl);
    },
  });

  const handleGenerate = () => {
    if (!prompt.trim()) return;
    generateMutation.mutate();
  };

  return (
    <KeyboardAvoidingView behavior={Platform.OS === 'ios' ? 'padding' : 'height'} style={{ flex: 1 }}>
      <ScrollView contentContainerStyle={{ padding: 16 }}>
        <TextInput
          label="Describe your image..."
          value={prompt}
          onChangeText={setPrompt}
          mode="outlined"
          multiline
          numberOfLines={3}
          style={{ marginBottom: 16 }}
        />

        <Title style={{ marginBottom: 8 }}>Model</Title>
        <ScrollView horizontal showsHorizontalScrollIndicator={false} style={{ marginBottom: 16 }}>
          {MODELS.map(m => (
            <Button
              key={m.id}
              mode={model === m.id ? 'contained' : 'outlined'}
              onPress={() => setModel(m.id)}
              style={{ marginRight: 8 }}
            >
              {m.icon} {m.name}
            </Button>
          ))}
        </ScrollView>

        <Title style={{ marginBottom: 8 }}>Style</Title>
        <ScrollView horizontal showsHorizontalScrollIndicator={false} style={{ marginBottom: 16 }}>
          {STYLES.map(s => (
            <Button
              key={s.id}
              mode={style === s.id ? 'contained' : 'outlined'}
              onPress={() => setStyle(s.id)}
              style={{ marginRight: 8 }}
            >
              {s.name}
            </Button>
          ))}
        </ScrollView>

        <Button
          mode="contained"
          onPress={handleGenerate}
          loading={generateMutation.isPending}
          disabled={!prompt.trim() || generateMutation.isPending}
          style={{ marginBottom: 16 }}
        >
          Generate Image
        </Button>

        {generateMutation.isPending && (
          <ActivityIndicator size="large" style={{ marginVertical: 16 }} />
        )}

        {resultImage && (
          <Card style={{ marginTop: 16 }}>
            <Card.Cover source={{ uri: resultImage }} />
            <Card.Actions>
              <Button onPress={() => {/* save to gallery */}}>Save to Gallery</Button>
              <Button onPress={() => {/* share */}}>Share</Button>
            </Card.Actions>
          </Card>
        )}

        {generateMutation.isError && (
          <Paragraph style={{ color: 'red', marginTop: 8 }}>
            {generateMutation.error?.message || 'Generation failed. Please try again.'}
          </Paragraph>
        )}
      </ScrollView>
    </KeyboardAvoidingView>
  );
}
```

### Step 8: Configure Push Notifications

```typescript
// src/utils/pushNotifications.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import { Platform } from 'react-native';
import apiClient from '../api/client';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

export async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) return null;

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') return null;

  const token = (await Notifications.getExpoPushTokenAsync()).data;

  // Send token to backend
  await apiClient.post('/auth/device-token', { token, platform: Platform.OS });

  return token;
}

// Listen for notifications while app is in foreground
Notifications.addNotificationReceivedListener((notification) => {
  console.log('Notification received:', notification);
});

// Handle notification tap when app is in background
Notifications.addNotificationResponseReceivedListener((response) => {
  const { type, imageId } = response.notification.request.content.data;
  if (type === 'generation_complete' && imageId) {
    // Navigate to image detail
    // router.navigate(`/image/${imageId}`);
  }
});
```

### Step 9: Configure EAS Build

```bash
eas login
eas build:configure
```

```json
// eas.json
{
  "cli": {
    "version": ">= 5.9.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_API_URL": "http://10.0.2.2:3001/api/v1"
      }
    },
    "preview": {
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_API_URL": "https://api.yourdomain.com/api/v1"
      }
    },
    "production": {
      "env": {
        "EXPO_PUBLIC_API_URL": "https://api.yourdomain.com/api/v1"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your-apple-id@email.com",
        "ascAppId": "1234567890"
      }
    }
  }
}
```

### Step 10: Build and Submit

```bash
# Build for development (test on device)
eas build --profile development --platform android
eas build --profile development --platform ios

# Install on connected device
eas build:run --profile development --platform android

# Build preview build
eas build --profile preview --platform android

# Build production release
eas build --profile production --platform android
eas build --profile production --platform ios

# Submit to stores
eas submit --platform android  # Google Play
eas submit --platform ios      # App Store
```

## Verification

1. **Dev server starts**: `npx expo start` → QR code appears, app loads on device
2. **Login works**: Enter credentials → API returns tokens → stored in SecureStore → redirected to Home
3. **Image generation**: Type prompt → select model → generate → image appears
4. **Offline detection**: Turn off WiFi → app shows offline banner → cached images still visible
5. **Push notifications**: App registers for notifications → token sent to backend
6. **Android build**: `eas build --profile production --platform android` succeeds
7. **iOS build**: `eas build --profile production --platform ios` succeeds (requires Apple Developer account)

## Rollback

1. Delete the `genai-mobile/` directory
2. Remove Expo project from EAS dashboard
3. Remove Firebase project if created
4. No backend changes needed — API is unchanged

## ADR-025: Expo (React Native) vs Flutter vs Tauri for Mobile

**Decision**: Use Expo (React Native) for mobile development alongside Flutter
**Reason**: Expo provides the fastest development cycle for React developers (same ecosystem as web), OTA updates without app store review, and excellent cloud build service (EAS). React Native shares knowledge with the web frontend (React, TypeScript, hooks).
**Consequences**:
- Faster development if team knows React
- OTA updates possible (no app store review for JS changes)
- Smooth CI/CD with EAS Build
- Slightly lower performance than Flutter for complex animations
- iOS builds require Apple Developer account ($99/year)
- Some native modules require custom native code (expo-dev-client)
**Alternatives Considered**:
- Flutter: Higher performance, single codebase for iOS+Android, but different language (Dart)
- Tauri: Not for mobile (desktop only)
- Native (Kotlin/Swift): Best performance but two codebases