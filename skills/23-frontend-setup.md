# Skill 23: Frontend Setup (React + Vite + TypeScript + TanStack Query + Zustand + Tailwind)

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 3-4 hours
Depends On: 03

## Input Contract
- Skill 03 complete: auth API endpoints operational (register, login, refresh, logout, me, etc.)
- Backend running at a known URL (e.g., `http://localhost:3000`)
- API contract understood: auth endpoints, image endpoints, gallery endpoints
- Node.js >= 18 installed

## Output Contract
- React + Vite + TypeScript project in `client/` directory
- Project structure: `src/api/`, `src/auth/`, `src/hooks/`, `src/stores/`, `src/components/ui/`, `src/components/features/`, `src/pages/`
- Axios API client with interceptors (auth token injection, refresh token rotation, error handling)
- Auth provider with login, signup, logout, and refresh functionality
- Zustand theme store (dark/light mode)
- TanStack Query hooks for images and gallery
- Protected route component
- Tailwind CSS with dark mode support
- All pages wired up and functional

## Files to Create

| File | Description |
|------|-------------|
| `client/package.json` | Dependencies and scripts |
| `client/vite.config.ts` | Vite configuration with proxy |
| `client/tsconfig.json` | TypeScript configuration |
| `client/tailwind.config.js` | Tailwind with dark mode |
| `client/postcss.config.js` | PostCSS for Tailwind |
| `client/index.html` | HTML entry point |
| `client/src/main.tsx` | App bootstrap |
| `client/src/App.tsx` | Router and providers |
| `client/src/api/client.ts` | Axios instance with interceptors |
| `client/src/api/auth.ts` | Auth API functions |
| `client/src/api/images.ts` | Image API functions |
| `client/src/api/gallery.ts` | Gallery API functions |
| `client/src/auth/provider.tsx` | Auth context and provider |
| `client/src/auth/protected-route.tsx` | Route guard component |
| `client/src/stores/theme.ts` | Zustand theme store |
| `client/src/hooks/use-images.ts` | TanStack Query hook for images |
| `client/src/hooks/use-gallery.ts` | TanStack Query hook for gallery |
| `client/src/components/ui/button.tsx` | Button component |
| `client/src/components/ui/input.tsx` | Input component |
| `client/src/components/ui/card.tsx` | Card component |
| `client/src/components/ui/modal.tsx` | Modal component |
| `client/src/components/ui/loading.tsx` | Loading spinner component |
| `client/src/components/ui/toast.tsx` | Toast notification component |
| `client/src/pages/login.tsx` | Login page |
| `client/src/pages/signup.tsx` | Signup page |
| `client/src/pages/dashboard.tsx` | Dashboard with image generation |
| `client/src/pages/gallery.tsx` | Gallery listing page |
| `client/src/pages/settings.tsx` | Settings with theme toggle |
| `client/src/styles/index.css` | Tailwind base styles |
| `client/src/types/index.ts` | Shared TypeScript types |
| `client/.env.example` | Environment variables template |

## Steps

### Step 1: Scaffold the Vite Project

```bash
cd /path/to/project
npm create vite@latest client -- --template react-ts
cd client
```

### Step 2: Install Dependencies

```bash
npm install axios @tanstack/react-query zustand react-router-dom
npm install -D tailwindcss postcss autoprefixer @types/node
npx tailwindcss init -p
```

### Step 3: Configure Vite

`client/vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
});
```

### Step 4: Configure Tailwind with Dark Mode

`client/tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#f0f4ff',
          100: '#dbe4ff',
          200: '#bac8ff',
          300: '#91a7ff',
          400: '#748ffc',
          500: '#5c7cfa',
          600: '#4c6ef5',
          700: '#4263eb',
          800: '#3b5bdb',
          900: '#364fc7',
        },
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
    },
  },
  plugins: [],
};
```

### Step 5: Base CSS Entry

`client/src/styles/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --color-bg: #ffffff;
    --color-text: #1a1a2e;
    --color-surface: #f8f9fa;
    --color-border: #e5e7eb;
  }

  .dark {
    --color-bg: #1a1a2e;
    --color-text: #e5e7eb;
    --color-surface: #16213e;
    --color-border: #2d3748;
  }

  body {
    @apply bg-[var(--color-bg)] text-[var(--color-text)] transition-colors duration-200;
  }
}
```

### Step 6: API Client with Interceptors

`client/src/api/client.ts`:

```ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_URL || '/api';

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: { 'Content-Type': 'application/json' },
});

let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (error: unknown) => void;
}> = [];

const processQueue = (error: unknown, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token!);
    }
  });
  failedQueue = [];
};

apiClient.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = localStorage.getItem('accessToken');
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({
            resolve: (token: string) => {
              originalRequest.headers.Authorization = `Bearer ${token}`;
              resolve(apiClient(originalRequest));
            },
            reject,
          });
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        if (!refreshToken) {
          throw new Error('No refresh token');
        }

        const { data } = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refreshToken,
        });

        localStorage.setItem('accessToken', data.accessToken);
        localStorage.setItem('refreshToken', data.refreshToken);

        processQueue(null, data.accessToken);
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

### Step 7: Auth API Functions

`client/src/api/auth.ts`:

```ts
import apiClient from './client';

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RegisterRequest {
  name: string;
  email: string;
  password: string;
}

export interface AuthResponse {
  user: {
    id: string;
    name: string;
    email: string;
    role: string;
  };
  accessToken: string;
  refreshToken: string;
}

export const authApi = {
  login: (data: LoginRequest) =>
    apiClient.post<AuthResponse>('/auth/login', data),

  register: (data: RegisterRequest) =>
    apiClient.post<AuthResponse>('/auth/register', data),

  refresh: (refreshToken: string) =>
    apiClient.post<AuthResponse>('/auth/refresh', { refreshToken }),

  logout: () => apiClient.post('/auth/logout'),

  me: () =>
    apiClient.get<{ user: AuthResponse['user'] }>('/auth/me'),

  forgotPassword: (email: string) =>
    apiClient.post('/auth/forgot-password', { email }),

  resetPassword: (token: string, password: string) =>
    apiClient.post('/auth/reset-password', { token, password }),

  verifyEmail: (token: string) =>
    apiClient.post('/auth/verify-email', { token }),
};
```

### Step 8: Image and Gallery API Functions

`client/src/api/images.ts`:

```ts
import apiClient from './client';

export interface GenerateImageRequest {
  prompt: string;
  provider?: 'dalle' | 'flux' | 'pollinations';
  width?: number;
  height?: number;
  galleryId?: string;
}

export interface ImageResponse {
  id: string;
  prompt: string;
  url: string;
  provider: string;
  userId: string;
  createdAt: string;
}

export const imagesApi = {
  generate: (data: GenerateImageRequest) =>
    apiClient.post<ImageResponse>('/images/generate', data),

  list: (params?: { page?: number; limit?: number }) =>
    apiClient.get<{ images: ImageResponse[]; total: number }>('/images', { params }),

  getById: (id: string) =>
    apiClient.get<ImageResponse>(`/images/${id}`),

  delete: (id: string) =>
    apiClient.delete(`/images/${id}`),

  toggleFavorite: (id: string) =>
    apiClient.patch(`/images/${id}/favorite`),
};
```

`client/src/api/gallery.ts`:

```ts
import apiClient from './client';

export interface CreateGalleryRequest {
  name: string;
  description?: string;
  isPublic?: boolean;
}

export interface GalleryResponse {
  id: string;
  name: string;
  description: string | null;
  isPublic: boolean;
  images: Array<{ id: string; url: string; prompt: string }>;
  createdAt: string;
}

export const galleryApi = {
  create: (data: CreateGalleryRequest) =>
    apiClient.post<GalleryResponse>('/gallery', data),

  list: (params?: { page?: number; limit?: number }) =>
    apiClient.get<{ galleries: GalleryResponse[]; total: number }>('/gallery', { params }),

  getById: (id: string) =>
    apiClient.get<GalleryResponse>(`/gallery/${id}`),

  update: (id: string, data: Partial<CreateGalleryRequest>) =>
    apiClient.patch<GalleryResponse>(`/gallery/${id}`, data),

  delete: (id: string) =>
    apiClient.delete(`/gallery/${id}`),

  addImage: (galleryId: string, imageId: string) =>
    apiClient.post(`/gallery/${galleryId}/images`, { imageId }),

  removeImage: (galleryId: string, imageId: string) =>
    apiClient.delete(`/gallery/${galleryId}/images/${imageId}`),

  explore: (params?: { page?: number; limit?: number }) =>
    apiClient.get<{ galleries: GalleryResponse[]; total: number }>('/gallery/explore', { params }),
};
```

### Step 9: Auth Provider

`client/src/auth/provider.tsx`:

```tsx
import { createContext, useContext, useEffect, useState, useCallback, ReactNode } from 'react';
import { authApi, AuthResponse } from '@/api/auth';

interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshUser: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const storeTokens = (data: AuthResponse) => {
    localStorage.setItem('accessToken', data.accessToken);
    localStorage.setItem('refreshToken', data.refreshToken);
  };

  const clearTokens = () => {
    localStorage.removeItem('accessToken');
    localStorage.removeItem('refreshToken');
  };

  const login = useCallback(async (email: string, password: string) => {
    const { data } = await authApi.login({ email, password });
    storeTokens(data);
    setUser(data.user);
  }, []);

  const register = useCallback(async (name: string, email: string, password: string) => {
    const { data } = await authApi.register({ name, email, password });
    storeTokens(data);
    setUser(data.user);
  }, []);

  const logout = useCallback(async () => {
    try {
      await authApi.logout();
    } finally {
      clearTokens();
      setUser(null);
    }
  }, []);

  const refreshUser = useCallback(async () => {
    try {
      const { data } = await authApi.me();
      setUser(data.user);
    } catch {
      clearTokens();
      setUser(null);
    }
  }, []);

  useEffect(() => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      refreshUser().finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, [refreshUser]);

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        register,
        logout,
        refreshUser,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

### Step 10: Protected Route Component

`client/src/auth/protected-route.tsx`:

```tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/auth/provider';
import { Loading } from '@/components/ui/loading';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return <Loading />;
  }

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <>{children}</>;
}
```

### Step 11: Zustand Theme Store

`client/src/stores/theme.ts`:

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type Theme = 'light' | 'dark' | 'system';

interface ThemeState {
  theme: Theme;
  resolvedTheme: 'light' | 'dark';
  setTheme: (theme: Theme) => void;
}

function getSystemTheme(): 'light' | 'dark' {
  if (typeof window === 'undefined') return 'light';
  return window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';
}

function applyTheme(resolved: 'light' | 'dark') {
  const root = document.documentElement;
  if (resolved === 'dark') {
    root.classList.add('dark');
  } else {
    root.classList.remove('dark');
  }
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: 'system',
      resolvedTheme: getSystemTheme(),
      setTheme: (theme: Theme) => {
        const resolved = theme === 'system' ? getSystemTheme() : theme;
        applyTheme(resolved);
        set({ theme, resolvedTheme: resolved });
      },
    }),
    {
      name: 'theme-storage',
      onRehydrateStorage: () => (state) => {
        if (state) {
          const resolved = state.theme === 'system' ? getSystemTheme() : state.theme;
          applyTheme(resolved);
          state.resolvedTheme = resolved;
        }
      },
    }
  )
);

if (typeof window !== 'undefined') {
  window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
    const { theme } = useThemeStore.getState();
    if (theme === 'system') {
      const resolved = getSystemTheme();
      applyTheme(resolved);
      useThemeStore.setState({ resolvedTheme: resolved });
    }
  });
}
```

### Step 12: TanStack Query Hooks

`client/src/hooks/use-images.ts`:

```ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { imagesApi, GenerateImageRequest, ImageResponse } from '@/api/images';

export function useImages(page = 1, limit = 20) {
  return useQuery({
    queryKey: ['images', page, limit],
    queryFn: async () => {
      const { data } = await imagesApi.list({ page, limit });
      return data;
    },
  });
}

export function useImage(id: string) {
  return useQuery({
    queryKey: ['image', id],
    queryFn: async () => {
      const { data } = await imagesApi.getById(id);
      return data;
    },
    enabled: !!id,
  });
}

export function useGenerateImage() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (params: GenerateImageRequest) => {
      const { data } = await imagesApi.generate(params);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['images'] });
    },
  });
}

export function useDeleteImage() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: string) => {
      await imagesApi.delete(id);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['images'] });
    },
  });
}

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: string) => {
      const { data } = await imagesApi.toggleFavorite(id);
      return data;
    },
    onSuccess: (_data, id) => {
      queryClient.invalidateQueries({ queryKey: ['images'] });
      queryClient.invalidateQueries({ queryKey: ['image', id] });
    },
  });
}
```

`client/src/hooks/use-gallery.ts`:

```ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { galleryApi, CreateGalleryRequest, GalleryResponse } from '@/api/gallery';

export function useGalleries(page = 1, limit = 20) {
  return useQuery({
    queryKey: ['galleries', page, limit],
    queryFn: async () => {
      const { data } = await galleryApi.list({ page, limit });
      return data;
    },
  });
}

export function useGallery(id: string) {
  return useQuery({
    queryKey: ['gallery', id],
    queryFn: async () => {
      const { data } = await galleryApi.getById(id);
      return data;
    },
    enabled: !!id,
  });
}

export function useExploreGalleries(page = 1, limit = 20) {
  return useQuery({
    queryKey: ['explore', page, limit],
    queryFn: async () => {
      const { data } = await galleryApi.explore({ page, limit });
      return data;
    },
  });
}

export function useCreateGallery() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (params: CreateGalleryRequest) => {
      const { data } = await galleryApi.create(params);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['galleries'] });
    },
  });
}

export function useDeleteGallery() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: string) => {
      await galleryApi.delete(id);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['galleries'] });
    },
  });
}
```

### Step 13: UI Components

`client/src/components/ui/button.tsx`:

```tsx
import { ButtonHTMLAttributes, forwardRef } from 'react';

type Variant = 'primary' | 'secondary' | 'danger' | 'ghost';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: Variant;
  isLoading?: boolean;
}

const variantClasses: Record<Variant, string> = {
  primary: 'bg-primary-600 hover:bg-primary-700 text-white',
  secondary: 'bg-gray-200 dark:bg-gray-700 hover:bg-gray-300 dark:hover:bg-gray-600',
  danger: 'bg-red-600 hover:bg-red-700 text-white',
  ghost: 'hover:bg-gray-100 dark:hover:bg-gray-800',
};

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', isLoading, className = '', children, disabled, ...props }, ref) => (
    <button
      ref={ref}
      className={`inline-flex items-center justify-center rounded-lg px-4 py-2 text-sm font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-primary-500 disabled:opacity-50 ${variantClasses[variant]} ${className}`}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading && (
        <svg className="mr-2 h-4 w-4 animate-spin" viewBox="0 0 24 24" fill="none">
          <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
          <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
        </svg>
      )}
      {children}
    </button>
  )
);
```

`client/src/components/ui/input.tsx`:

```tsx
import { InputHTMLAttributes, forwardRef } from 'react';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, className = '', id, ...props }, ref) => (
    <div className="w-full">
      {label && (
        <label htmlFor={id} className="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300">
          {label}
        </label>
      )}
      <input
        ref={ref}
        id={id}
        className={`w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm shadow-sm transition-colors focus:border-primary-500 focus:outline-none focus:ring-1 focus:ring-primary-500 dark:border-gray-600 dark:bg-gray-800 dark:text-gray-100 ${error ? 'border-red-500' : ''} ${className}`}
        {...props}
      />
      {error && <p className="mt-1 text-xs text-red-500">{error}</p>}
    </div>
  )
);
```

### Step 14: Login and Signup Pages

`client/src/pages/login.tsx`:

```tsx
import { useState } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useAuth } from '@/auth/provider';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';

export default function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);
    try {
      await login(email, password);
      navigate('/dashboard');
    } catch (err: any) {
      setError(err.response?.data?.message || 'Login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center bg-[var(--color-bg)] px-4">
      <div className="w-full max-w-md space-y-8">
        <div className="text-center">
          <h1 className="text-3xl font-bold">Welcome back</h1>
          <p className="mt-2 text-sm text-gray-500 dark:text-gray-400">
            Sign in to your account
          </p>
        </div>
        <form onSubmit={handleSubmit} className="mt-8 space-y-4">
          {error && (
            <div className="rounded-lg bg-red-50 p-3 text-sm text-red-600 dark:bg-red-900/20 dark:text-red-400">
              {error}
            </div>
          )}
          <Input label="Email" id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
          <Input label="Password" id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
          <Button type="submit" isLoading={loading} className="w-full">Sign in</Button>
        </form>
        <p className="text-center text-sm text-gray-500 dark:text-gray-400">
          Don&apos;t have an account?{' '}
          <Link to="/signup" className="text-primary-600 hover:text-primary-500">Sign up</Link>
        </p>
      </div>
    </div>
  );
}
```

### Step 15: Dashboard Page (Image Generation)

`client/src/pages/dashboard.tsx`:

```tsx
import { useState } from 'react';
import { useGenerateImage } from '@/hooks/use-images';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';

export default function DashboardPage() {
  const [prompt, setPrompt] = useState('');
  const [provider, setProvider] = useState<'dalle' | 'flux' | 'pollinations'>('dalle');
  const generate = useGenerateImage();

  const handleGenerate = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!prompt.trim()) return;
    generate.mutate({ prompt, provider }, { onSuccess: () => setPrompt('') });
  };

  return (
    <div className="mx-auto max-w-4xl px-4 py-8">
      <h1 className="mb-8 text-2xl font-bold">Generate Image</h1>
      <form onSubmit={handleGenerate} className="space-y-4">
        <Input
          label="Prompt"
          id="prompt"
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          placeholder="Describe the image you want to create..."
          required
        />
        <div className="flex gap-2">
          {(['dalle', 'flux', 'pollinations'] as const).map((p) => (
            <Button
              key={p}
              type="button"
              variant={provider === p ? 'primary' : 'secondary'}
              onClick={() => setProvider(p)}
            >
              {p.charAt(0).toUpperCase() + p.slice(1)}
            </Button>
          ))}
        </div>
        <Button type="submit" isLoading={generate.isPending}>Generate</Button>
      </form>
      {generate.data && (
        <div className="mt-8">
          <img src={generate.data.url} alt={generate.data.prompt} className="rounded-lg shadow-lg" />
          <p className="mt-2 text-sm text-gray-500">{generate.data.prompt}</p>
        </div>
      )}
      {generate.error && (
        <p className="mt-4 text-sm text-red-500">
          {(generate.error as any).response?.data?.message || 'Generation failed'}
        </p>
      )}
    </div>
  );
}
```

### Step 16: App Router and Providers

`client/src/App.tsx`:

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { AuthProvider } from '@/auth/provider';
import { ProtectedRoute } from '@/auth/protected-route';
import LoginPage from '@/pages/login';
import SignupPage from '@/pages/signup';
import DashboardPage from '@/pages/dashboard';
import GalleryPage from '@/pages/gallery';
import SettingsPage from '@/pages/settings';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: 1,
    },
  },
});

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <BrowserRouter>
          <Routes>
            <Route path="/login" element={<LoginPage />} />
            <Route path="/signup" element={<SignupPage />} />
            <Route
              path="/dashboard"
              element={
                <ProtectedRoute>
                  <DashboardPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/gallery"
              element={
                <ProtectedRoute>
                  <GalleryPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/settings"
              element={
                <ProtectedRoute>
                  <SettingsPage />
                </ProtectedRoute>
              }
            />
            <Route path="*" element={<Navigate to="/dashboard" replace />} />
          </Routes>
        </BrowserRouter>
      </AuthProvider>
    </QueryClientProvider>
  );
}
```

`client/src/main.tsx`:

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles/index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### Step 17: Environment Variables

`client/.env.example`:

```
VITE_API_URL=/api
```

### Step 18: Update package.json Scripts

Ensure `client/package.json` includes:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,tsx",
    "typecheck": "tsc --noEmit"
  }
}
```

## Verification

```bash
# 1. Install dependencies
cd client && npm install

# 2. Type check
npm run typecheck

# 3. Lint
npm run lint

# 4. Build production bundle
npm run build

# 5. Start dev server and verify
npm run dev
# Open http://localhost:5173 - should show login page

# 6. Verify dark mode toggle
# Click theme toggle in settings - should switch dark/light

# 7. Verify auth flow
# Register -> should redirect to dashboard
# Logout -> should redirect to login
# Refresh page -> should stay logged in

# 8. Verify API proxy
# Check browser network tab that /api/* requests proxy to backend
```

## Rollback

```bash
# Remove the client directory entirely
rm -rf client/

# Or remove specific files if only parts need reverting
rm client/src/hooks/use-images.ts client/src/hooks/use-gallery.ts
```

All frontend changes are contained in the `client/` directory. Deleting it restores the project to backend-only state.

## ADR-023: Frontend Stack Selection

**Decision**: Use React + Vite + TypeScript with TanStack Query, Zustand, and Tailwind CSS.

**Reason**: Vite provides fast HMR and builds. TanStack Query handles server state with caching, refetching, and optimistic updates. Zustand is lightweight for client-only state (theme, UI toggles). Tailwind enables rapid styling with dark mode support. TypeScript throughout catches errors at build time.

**Consequences**:
- Smaller bundle than Next.js (no SSR overhead)
- TanStack Query requires understanding of cache invalidation patterns
- Zustand is minimal — no devtools middleware by default (can be added)
- No SSR means SEO challenges for public gallery pages (could add later with Next.js)
- Vite proxy handles CORS in dev, but production needs proper CORS config on backend

**Alternatives Considered**:
- Next.js: Full SSR but overkill for a dashboard app, adds complexity
- Redux Toolkit: More boilerplate than Zustand for client state
- React Query v4: Rebranded to TanStack Query, v5 has breaking changes worth avoiding initially
- Svelte/SvelteKit: Excellent DX but smaller ecosystem and fewer component libraries
- CSS Modules: More scoping control but slower development velocity than Tailwind