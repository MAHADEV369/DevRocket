# Frontend Build Guide: React + Vite + TypeScript

Complete step-by-step guide to building the web frontend for the FastDepo image generation application.

---

## 1. Project Creation with Vite

```bash
npm create vite@latest fastdepo-web -- --template react-ts
cd fastdepo-web
npm install
```

This scaffolds a React + TypeScript project with Vite as the bundler. Vite provides instant HMR, fast builds, and native ESM support.

Open `vite.config.ts` and set the base path and dev server proxy:

```typescript
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
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          query: ['@tanstack/react-query'],
          ui: ['zustand'],
        },
      },
    },
  },
});
```

Install core dependencies:

```bash
npm install react react-dom react-router-dom
npm install @tanstack/react-query axios zustand
npm install clsx tailwind-merge lucide-react
npm install -D @types/react @types/react-dom
npm install -D tailwindcss @tailwindcss/vite
```

---

## 2. Folder Structure

```
src/
├── api/                    # API client and endpoint definitions
│   ├── client.ts            # Axios instance with interceptors
│   ├── auth.ts              # Auth API calls
│   ├── images.ts            # Image generation API calls
│   └── gallery.ts           # Gallery API calls
├── auth/                    # Authentication logic
│   ├── AuthProvider.tsx      # Context provider for auth state
│   ├── useAuth.ts           # Hook to access auth context
│   ├── ProtectedRoute.tsx   # Route guard component
│   └── tokens.ts            # Token storage utilities
├── hooks/                   # Custom React hooks
│   ├── useTheme.ts          # Theme toggle hook
│   ├── useMediaQuery.ts     # Responsive breakpoint hook
│   └── useDebounce.ts       # Debounce hook for inputs
├── stores/                  # Zustand stores
│   ├── themeStore.ts        # Dark/light theme state
│   ├── generationStore.ts   # Image generation state
│   └── uiStore.ts           # UI state (modals, toasts, sidebar)
├── components/
│   ├── ui/                  # Primitives
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Card.tsx
│   │   ├── Modal.tsx
│   │   ├── Toast.tsx
│   │   ├── Spinner.tsx
│   │   └── Dropdown.tsx
│   └── features/            # Domain-specific components
│       ├── ImageCard.tsx
│       ├── GenerationForm.tsx
│       ├── GalleryGrid.tsx
│       └── Navbar.tsx
├── pages/                   # Route-level components
│   ├── LoginPage.tsx
│   ├── SignupPage.tsx
│   ├── GeneratePage.tsx
│   ├── GalleryPage.tsx
│   └── SettingsPage.tsx
├── lib/                     # Utilities
│   ├── cn.ts                # clsx + tailwind-merge utility
│   └── constants.ts         # App-wide constants
├── App.tsx                  # Root component with providers
├── main.tsx                 # Entry point
└── index.css                # Tailwind directives + globals
```

---

## 3. Tailwind CSS Setup

Install Tailwind CSS v4 with the Vite plugin:

```bash
npm install -D tailwindcss @tailwindcss/vite
```

Update `vite.config.ts` to add the plugin:

```typescript
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  // ...
});
```

In `src/index.css`, add the Tailwind import and custom theme:

```css
@import "tailwindcss";

@theme {
  --color-primary-50: #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-200: #bfdbfe;
  --color-primary-300: #93c5fd;
  --color-primary-400: #60a5fa;
  --color-primary-500: #3b82f6;
  --color-primary-600: #2563eb;
  --color-primary-700: #1d4ed8;
  --color-primary-800: #1e40af;
  --color-primary-900: #1e3a8a;

  --color-surface: #ffffff;
  --color-surface-dark: #0f172a;
  --color-card: #f8fafc;
  --color-card-dark: #1e293b;

  --font-sans: 'Inter', ui-sans-serif, system-ui, sans-serif;
}

:root {
  --color-bg: var(--color-surface);
  --color-text: #0f172a;
  --color-border: #e2e8f0;
}

.dark {
  --color-bg: var(--color-surface-dark);
  --color-text: #f1f5f9;
  --color-border: #334155;
}
```

---

## 4. API Client with Axios

### `src/api/client.ts`

```typescript
import axios from 'axios';
import { getAccessToken, getRefreshToken, setTokens, clearTokens } from '@/auth/tokens';

const API_BASE_URL = import.meta.env.VITE_API_URL || '/api';

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

apiClient.interceptors.request.use(
  (config) => {
    const token = getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (error: unknown) => void;
}> = [];

const processQueue = (error: unknown, token: string | null = null) => {
  failedQueue.forEach((promise) => {
    if (error) {
      promise.reject(error);
    } else {
      promise.resolve(token!);
    }
  });
  failedQueue = [];
};

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then((token) => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return apiClient(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = getRefreshToken();
        if (!refreshToken) {
          throw new Error('No refresh token');
        }

        const { data } = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refreshToken,
        });

        setTokens(data.accessToken, data.refreshToken);
        processQueue(null, data.accessToken);

        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        clearTokens();
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

### `src/auth/tokens.ts`

```typescript
const ACCESS_TOKEN_KEY = 'fastdepo_access_token';
const REFRESH_TOKEN_KEY = 'fastdepo_refresh_token';

export function getAccessToken(): string | null {
  return sessionStorage.getItem(ACCESS_TOKEN_KEY);
}

export function getRefreshToken(): string | null {
  return localStorage.getItem(REFRESH_TOKEN_KEY);
}

export function setTokens(accessToken: string, refreshToken: string): void {
  sessionStorage.setItem(ACCESS_TOKEN_KEY, accessToken);
  localStorage.setItem(REFRESH_TOKEN_KEY, refreshToken);
}

export function clearTokens(): void {
  sessionStorage.removeItem(ACCESS_TOKEN_KEY);
  localStorage.removeItem(REFRESH_TOKEN_KEY);
}
```

### `src/api/auth.ts`

```typescript
import apiClient from './client';

export interface LoginRequest {
  email: string;
  password: string;
}

export interface SignupRequest {
  email: string;
  password: string;
  name: string;
}

export interface AuthResponse {
  accessToken: string;
  refreshToken: string;
  user: {
    id: string;
    email: string;
    name: string;
  };
}

export const authApi = {
  login: (data: LoginRequest) =>
    apiClient.post<AuthResponse>('/auth/login', data),

  signup: (data: SignupRequest) =>
    apiClient.post<AuthResponse>('/auth/signup', data),

  logout: () =>
    apiClient.post('/auth/logout'),

  refresh: (refreshToken: string) =>
    apiClient.post<AuthResponse>('/auth/refresh', { refreshToken }),

  me: () =>
    apiClient.get<AuthResponse['user']>('/auth/me'),
};
```

### `src/api/images.ts`

```typescript
import apiClient from './client';

export interface GenerateRequest {
  prompt: string;
  model: string;
  width?: number;
  height?: number;
  negativePrompt?: string;
}

export interface GenerationResponse {
  id: string;
  imageUrl: string;
  status: 'generating' | 'completed' | 'failed';
  prompt: string;
  createdAt: string;
}

export const imagesApi = {
  generate: (data: GenerateRequest) =>
    apiClient.post<GenerationResponse>('/images/generate', data, {
      timeout: 120000,
    }),

  getStatus: (id: string) =>
    apiClient.get<GenerationResponse>(`/images/${id}`),

  save: (id: string) =>
    apiClient.post(`/images/${id}/save`),

  delete: (id: string) =>
    apiClient.delete(`/images/${id}`),
};
```

### `src/api/gallery.ts`

```typescript
import apiClient from './client';

export interface GalleryParams {
  page?: number;
  limit?: number;
  sort?: 'newest' | 'oldest';
}

export interface GalleryImage {
  id: string;
  imageUrl: string;
  prompt: string;
  model: string;
  createdAt: string;
}

export interface GalleryResponse {
  images: GalleryImage[];
  total: number;
  page: number;
  limit: number;
}

export const galleryApi = {
  list: (params?: GalleryParams) =>
    apiClient.get<GalleryResponse>('/gallery', { params }),

  get: (id: string) =>
    apiClient.get<GalleryImage>(`/gallery/${id}`),

  delete: (id: string) =>
    apiClient.delete(`/gallery/${id}`),
};
```

---

## 5. TanStack Query Setup and Hooks

### `src/main.tsx`

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';
import './index.css';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: 2,
      refetchOnWindowFocus: false,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>
);
```

### `src/hooks/useGallery.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { galleryApi, type GalleryParams } from '@/api/gallery';

export function useGallery(params?: GalleryParams) {
  return useQuery({
    queryKey: ['gallery', params],
    queryFn: () => galleryApi.list(params).then((r) => r.data),
  });
}

export function useGalleryImage(id: string) {
  return useQuery({
    queryKey: ['gallery', id],
    queryFn: () => galleryApi.get(id).then((r) => r.data),
    enabled: !!id,
  });
}

export function useDeleteImage() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => galleryApi.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['gallery'] });
    },
  });
}
```

### `src/hooks/useImageGeneration.ts`

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { imagesApi, type GenerateRequest } from '@/api/images';
import { useGenerationStore } from '@/stores/generationStore';

export function useImageGeneration() {
  const queryClient = useQueryClient();
  const { setGenerating, setResult, setError } = useGenerationStore();

  return useMutation({
    mutationFn: (data: GenerateRequest) => {
      setGenerating(true);
      setError(null);
      return imagesApi.generate(data);
    },
    onSuccess: (response) => {
      setResult(response.data);
      setGenerating(false);
      queryClient.invalidateQueries({ queryKey: ['gallery'] });
    },
    onError: (error) => {
      setError(error instanceof Error ? error.message : 'Generation failed');
      setGenerating(false);
    },
  });
}
```

---

## 6. Zustand Stores

### `src/stores/themeStore.ts`

```typescript
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

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: 'system',
      resolvedTheme: getSystemTheme(),
      setTheme: (theme: Theme) => {
        const resolved = theme === 'system' ? getSystemTheme() : theme;
        document.documentElement.classList.toggle('dark', resolved === 'dark');
        set({ theme, resolvedTheme: resolved });
      },
    }),
    {
      name: 'fastdepo-theme',
      onRehydrateStorage: () => (state) => {
        if (state) {
          const resolved = state.theme === 'system' ? getSystemTheme() : state.theme;
          document.documentElement.classList.toggle('dark', resolved === 'dark');
          state.resolvedTheme = resolved;
        }
      },
    }
  )
);

if (typeof window !== 'undefined') {
  window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
    const store = useThemeStore.getState();
    if (store.theme === 'system') {
      const resolved = getSystemTheme();
      document.documentElement.classList.toggle('dark', resolved === 'dark');
      useThemeStore.setState({ resolvedTheme: resolved });
    }
  });
}
```

### `src/stores/generationStore.ts`

```typescript
import { create } from 'zustand';
import type { GenerationResponse } from '@/api/images';

interface GenerationState {
  isGenerating: boolean;
  result: GenerationResponse | null;
  error: string | null;
  history: GenerationResponse[];
  setGenerating: (v: boolean) => void;
  setResult: (r: GenerationResponse | null) => void;
  setError: (e: string | null) => void;
  addToHistory: (r: GenerationResponse) => void;
  clearHistory: () => void;
}

export const useGenerationStore = create<GenerationState>((set) => ({
  isGenerating: false,
  result: null,
  error: null,
  history: [],
  setGenerating: (isGenerating) => set({ isGenerating }),
  setResult: (result) =>
    set((state) => ({
      result,
      history: result ? [result, ...state.history].slice(0, 50) : state.history,
    })),
  setError: (error) => set({ error }),
  addToHistory: (r) =>
    set((state) => ({ history: [r, ...state.history].slice(0, 50) })),
  clearHistory: () => set({ history: [] }),
}));
```

### `src/stores/uiStore.ts`

```typescript
import { create } from 'zustand';

interface Toast {
  id: string;
  type: 'success' | 'error' | 'info';
  message: string;
}

interface UIState {
  sidebarOpen: boolean;
  toasts: Toast[];
  toggleSidebar: () => void;
  addToast: (toast: Omit<Toast, 'id'>) => void;
  removeToast: (id: string) => void;
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: false,
  toasts: [],
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
  addToast: (toast) =>
    set((s) => ({
      toasts: [...s.toasts, { ...toast, id: crypto.randomUUID() }],
    })),
  removeToast: (id) =>
    set((s) => ({ toasts: s.toasts.filter((t) => t.id !== id) })),
}));
```

---

## 7. Auth Provider and Protected Route

### `src/auth/AuthProvider.tsx`

```tsx
import { createContext, useEffect, useState, type ReactNode } from 'react';
import { authApi, type AuthResponse } from '@/api/auth';
import { getAccessToken, getRefreshToken, setTokens, clearTokens } from './tokens';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthContextValue {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  signup: (email: string, password: string, name: string) => Promise<void>;
  logout: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const token = getAccessToken();
    if (token) {
      authApi.me()
        .then((res) => setUser(res.data))
        .catch(() => {
          clearTokens();
          setUser(null);
        })
        .finally(() => setIsLoading(false));
    } else {
      setIsLoading(false);
    }
  }, []);

  const login = async (email: string, password: string) => {
    const { data } = await authApi.login({ email, password });
    setTokens(data.accessToken, data.refreshToken);
    setUser(data.user);
  };

  const signup = async (email: string, password: string, name: string) => {
    const { data } = await authApi.signup({ email, password, name });
    setTokens(data.accessToken, data.refreshToken);
    setUser(data.user);
  };

  const logout = async () => {
    try {
      await authApi.logout();
    } finally {
      clearTokens();
      setUser(null);
    }
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        signup,
        logout,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}
```

### `src/auth/useAuth.ts`

```typescript
import { useContext } from 'react';
import { AuthContext } from './AuthProvider';

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### `src/auth/ProtectedRoute.tsx`

```tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from './useAuth';
import { Spinner } from '@/components/ui/Spinner';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <Spinner size="lg" />
      </div>
    );
  }

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <>{children}</>;
}
```

---

## 8. Component Library

### `src/lib/cn.ts`

```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### `src/components/ui/Button.tsx`

```tsx
import { forwardRef, type ButtonHTMLAttributes } from 'react';
import { cn } from '@/lib/cn';
import { Spinner } from './Spinner';

type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
}

const variantStyles: Record<ButtonVariant, string> = {
  primary: 'bg-primary-600 text-white hover:bg-primary-700',
  secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 dark:bg-gray-800 dark:text-gray-100',
  ghost: 'text-gray-700 hover:bg-gray-100 dark:text-gray-300 dark:hover:bg-gray-800',
  danger: 'bg-red-600 text-white hover:bg-red-700',
};

const sizeStyles: Record<ButtonSize, string> = {
  sm: 'px-3 py-1.5 text-sm',
  md: 'px-4 py-2 text-base',
  lg: 'px-6 py-3 text-lg',
};

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = 'primary', size = 'md', isLoading, disabled, children, ...props }, ref) => (
    <button
      ref={ref}
      className={cn(
        'inline-flex items-center justify-center rounded-lg font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 disabled:opacity-50 disabled:pointer-events-none',
        variantStyles[variant],
        sizeStyles[size],
        className
      )}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading && <Spinner size="sm" className="mr-2" />}
      {children}
    </button>
  )
);

Button.displayName = 'Button';
```

### `src/components/ui/Input.tsx`

```tsx
import { forwardRef, type InputHTMLAttributes } from 'react';
import { cn } from '@/lib/cn';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ className, label, error, id, ...props }, ref) => (
    <div className="w-full">
      {label && (
        <label htmlFor={id} className="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1">
          {label}
        </label>
      )}
      <input
        ref={ref}
        id={id}
        className={cn(
          'w-full rounded-lg border border-gray-300 bg-white px-4 py-2 text-sm transition-colors',
          'focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500/20',
          'dark:border-gray-600 dark:bg-gray-800 dark:text-white',
          error && 'border-red-500 focus:border-red-500 focus:ring-red-500/20',
          className
        )}
        {...props}
      />
      {error && <p className="mt-1 text-sm text-red-600">{error}</p>}
    </div>
  )
);

Input.displayName = 'Input';
```

### `src/components/ui/Card.tsx`

```tsx
import type { HTMLAttributes, ReactNode } from 'react';
import { cn } from '@/lib/cn';

interface CardProps extends HTMLAttributes<HTMLDivElement> {
  children: ReactNode;
}

export function Card({ className, children, ...props }: CardProps) {
  return (
    <div
      className={cn(
        'rounded-xl border border-gray-200 bg-white shadow-sm dark:border-gray-700 dark:bg-gray-900',
        className
      )}
      {...props}
    >
      {children}
    </div>
  );
}

export function CardHeader({ className, children, ...props }: CardProps) {
  return (
    <div className={cn('px-6 py-4 border-b border-gray-200 dark:border-gray-700', className)} {...props}>
      {children}
    </div>
  );
}

export function CardContent({ className, children, ...props }: CardProps) {
  return (
    <div className={cn('px-6 py-4', className)} {...props}>
      {children}
    </div>
  );
}
```

### `src/components/ui/Modal.tsx`

```tsx
import { useEffect, useRef, type ReactNode } from 'react';
import { cn } from '@/lib/cn';

interface ModalProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  children: ReactNode;
  className?: string;
}

export function Modal({ isOpen, onClose, title, children, className }: ModalProps) {
  const overlayRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden';
    } else {
      document.body.style.overflow = '';
    }
    return () => {
      document.body.style.overflow = '';
    };
  }, [isOpen]);

  useEffect(() => {
    const handleEsc = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    if (isOpen) document.addEventListener('keydown', handleEsc);
    return () => document.removeEventListener('keydown', handleEsc);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div
      ref={overlayRef}
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm"
      onClick={(e) => {
        if (e.target === overlayRef.current) onClose();
      }}
      role="dialog"
      aria-modal="true"
      aria-label={title}
    >
      <div className={cn('bg-white dark:bg-gray-900 rounded-xl shadow-xl max-w-lg w-full mx-4', className)}>
        {title && (
          <div className="flex items-center justify-between px-6 py-4 border-b border-gray-200 dark:border-gray-700">
            <h2 className="text-lg font-semibold">{title}</h2>
            <button onClick={onClose} className="text-gray-500 hover:text-gray-700" aria-label="Close">
              &times;
            </button>
          </div>
        )}
        <div className="px-6 py-4">{children}</div>
      </div>
    </div>
  );
}
```

### `src/components/ui/Toast.tsx`

```tsx
import { useEffect } from 'react';
import { useUIStore } from '@/stores/uiStore';
import { cn } from '@/lib/cn';

const typeStyles = {
  success: 'bg-green-500 text-white',
  error: 'bg-red-500 text-white',
  info: 'bg-blue-500 text-white',
};

export function ToastContainer() {
  const { toasts, removeToast } = useUIStore();

  return (
    <div className="fixed bottom-4 right-4 z-50 flex flex-col gap-2" aria-live="polite">
      {toasts.map((toast) => (
        <ToastItem key={toast.id} toast={toast} onClose={() => removeToast(toast.id)} />
      ))}
    </div>
  );
}

function ToastItem({ toast, onClose }: { toast: { id: string; type: 'success' | 'error' | 'info'; message: string }; onClose: () => void }) {
  useEffect(() => {
    const timer = setTimeout(onClose, 5000);
    return () => clearTimeout(timer);
  }, [onClose]);

  return (
    <div className={cn('px-4 py-3 rounded-lg shadow-lg text-sm', typeStyles[toast.type])} role="alert">
      {toast.message}
    </div>
  );
}
```

### `src/components/ui/Spinner.tsx`

```tsx
import { cn } from '@/lib/cn';

interface SpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}

const sizeStyles = {
  sm: 'h-4 w-4',
  md: 'h-6 w-6',
  lg: 'h-10 w-10',
};

export function Spinner({ size = 'md', className }: SpinnerProps) {
  return (
    <svg
      className={cn('animate-spin text-current', sizeStyles[size], className)}
      xmlns="http://www.w3.org/2000/svg"
      fill="none"
      viewBox="0 0 24 24"
      aria-label="Loading"
    >
      <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
      <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z" />
    </svg>
  );
}
```

### `src/components/ui/Dropdown.tsx`

```tsx
import { useState, useRef, useEffect, type ReactNode } from 'react';
import { cn } from '@/lib/cn';

interface DropdownProps {
  trigger: ReactNode;
  children: ReactNode;
  align?: 'left' | 'right';
}

export function Dropdown({ trigger, children, align = 'right' }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const handleClickOutside = (e: MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) {
        setIsOpen(false);
      }
    };
    document.addEventListener('mousedown', handleClickOutside);
    return () => document.removeEventListener('mousedown', handleClickOutside);
  }, []);

  return (
    <div ref={ref} className="relative inline-block">
      <div onClick={() => setIsOpen(!isOpen)} className="cursor-pointer">
        {trigger}
      </div>
      {isOpen && (
        <div
          className={cn(
            'absolute z-50 mt-2 min-w-[180px] rounded-lg border border-gray-200 bg-white shadow-lg dark:border-gray-700 dark:bg-gray-900',
            align === 'right' ? 'right-0' : 'left-0'
          )}
          onClick={() => setIsOpen(false)}
        >
          {children}
        </div>
      )}
    </div>
  );
}

export function DropdownItem({ className, children, ...props }: React.ComponentProps<'button'>) {
  return (
    <button
      className={cn(
        'block w-full px-4 py-2 text-left text-sm hover:bg-gray-100 dark:hover:bg-gray-800',
        className
      )}
      {...props}
    >
      {children}
    </button>
  );
}
```

---

## 9. Dark/Light Theme with System Detection

The theme implementation uses the `useThemeStore` from above plus a `useTheme` convenience hook:

### `src/hooks/useTheme.ts`

```typescript
import { useThemeStore } from '@/stores/themeStore';

export function useTheme() {
  const { theme, resolvedTheme, setTheme } = useThemeStore();

  const toggle = () => {
    if (theme === 'system') {
      setTheme(resolvedTheme === 'dark' ? 'light' : 'dark');
    } else {
      setTheme(theme === 'dark' ? 'light' : 'dark');
    }
  };

  return { theme, resolvedTheme, setTheme, isDark: resolvedTheme === 'dark', toggle };
}
```

Apply theme on app mount:

```tsx
import { useEffect } from 'react';
import { useThemeStore } from '@/stores/themeStore';

export function ThemeInitializer() {
  const setTheme = useThemeStore((s) => s.setTheme);

  useEffect(() => {
    const stored = localStorage.getItem('fastdepo-theme');
    setTheme(stored ? JSON.parse(stored).state.theme : 'system');
  }, [setTheme]);

  return null;
}
```

---

## 10. Pages

### `src/pages/LoginPage.tsx`

```tsx
import { useState, type FormEvent } from 'react';
import { Link, useNavigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/auth/useAuth';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';

export function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const from = (location.state as { from?: { pathname: string } })?.from?.pathname || '/';

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);
    try {
      await login(email, password);
      navigate(from, { replace: true });
    } catch (err) {
      setError('Invalid email or password');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center px-4">
      <div className="w-full max-w-sm">
        <h1 className="text-2xl font-bold text-center mb-6">Sign in to FastDepo</h1>
        {error && <div className="mb-4 rounded-lg bg-red-50 p-3 text-sm text-red-600">{error}</div>}
        <form onSubmit={handleSubmit} className="space-y-4">
          <Input id="email" label="Email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
          <Input id="password" label="Password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
          <Button type="submit" className="w-full" isLoading={loading}>Sign In</Button>
        </form>
        <p className="mt-4 text-center text-sm text-gray-600 dark:text-gray-400">
          Don&apos;t have an account? <Link to="/signup" className="text-primary-600 hover:underline">Sign up</Link>
        </p>
      </div>
    </div>
  );
}
```

### `src/pages/SignupPage.tsx`

```tsx
import { useState, type FormEvent } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useAuth } from '@/auth/useAuth';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';

export function SignupPage() {
  const { signup } = useAuth();
  const navigate = useNavigate();
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);
    try {
      await signup(email, password, name);
      navigate('/');
    } catch {
      setError('Signup failed. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center px-4">
      <div className="w-full max-w-sm">
        <h1 className="text-2xl font-bold text-center mb-6">Create your account</h1>
        {error && <div className="mb-4 rounded-lg bg-red-50 p-3 text-sm text-red-600">{error}</div>}
        <form onSubmit={handleSubmit} className="space-y-4">
          <Input id="name" label="Full name" value={name} onChange={(e) => setName(e.target.value)} required />
          <Input id="email" label="Email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
          <Input id="password" label="Password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required minLength={8} />
          <Button type="submit" className="w-full" isLoading={loading}>Create Account</Button>
        </form>
        <p className="mt-4 text-center text-sm text-gray-600 dark:text-gray-400">
          Already have an account? <Link to="/login" className="text-primary-600 hover:underline">Sign in</Link>
        </p>
      </div>
    </div>
  );
}
```

### `src/pages/GeneratePage.tsx`

```tsx
import { useState, type FormEvent } from 'react';
import { useImageGeneration } from '@/hooks/useImageGeneration';
import { useGenerationStore } from '@/stores/generationStore';
import { Button } from '@/components/ui/Button';
import { Input } from '@/components/ui/Input';
import { Card, CardContent } from '@/components/ui/Card';
import { Spinner } from '@/components/ui/Spinner';

const MODELS = ['stable-diffusion-xl', 'dall-e-3', 'flux-schnell'];

export function GeneratePage() {
  const [prompt, setPrompt] = useState('');
  const [model, setModel] = useState(MODELS[0]);
  const generate = useImageGeneration();
  const { isGenerating, result, error } = useGenerationStore();

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    if (!prompt.trim()) return;
    generate.mutate({ prompt, model });
  };

  return (
    <div className="max-w-3xl mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Generate Image</h1>
      <form onSubmit={handleSubmit} className="space-y-4 mb-8">
        <div>
          <label className="block text-sm font-medium mb-1">Prompt</label>
          <textarea
            className="w-full rounded-lg border border-gray-300 bg-white px-4 py-3 text-sm dark:border-gray-600 dark:bg-gray-800 dark:text-white focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500/20"
            rows={4}
            value={prompt}
            onChange={(e) => setPrompt(e.target.value)}
            placeholder="Describe the image you want to create..."
            required
          />
        </div>
        <div>
          <label className="block text-sm font-medium mb-1">Model</label>
          <select
            className="w-full rounded-lg border border-gray-300 bg-white px-4 py-2 text-sm dark:border-gray-600 dark:bg-gray-800 dark:text-white"
            value={model}
            onChange={(e) => setModel(e.target.value)}
          >
            {MODELS.map((m) => (
              <option key={m} value={m}>{m}</option>
            ))}
          </select>
        </div>
        <Button type="submit" isLoading={isGenerating} disabled={isGenerating}>
          {isGenerating ? 'Generating...' : 'Generate'}
        </Button>
      </form>

      {error && (
        <div className="mb-4 rounded-lg bg-red-50 p-4 text-sm text-red-600 dark:bg-red-900/20 dark:text-red-400">
          {error}
        </div>
      )}

      {result && (
        <Card>
          <CardContent>
            <img
              src={result.imageUrl}
              alt={result.prompt}
              className="w-full rounded-lg"
              loading="lazy"
            />
            <p className="mt-3 text-sm text-gray-600 dark:text-gray-400">{result.prompt}</p>
          </CardContent>
        </Card>
      )}
    </div>
  );
}
```

### `src/pages/GalleryPage.tsx`

```tsx
import { useGallery, useDeleteImage } from '@/hooks/useGallery';
import { useUIStore } from '@/stores/uiStore';
import { Card } from '@/components/ui/Card';
import { Spinner } from '@/components/ui/Spinner';
import { Button } from '@/components/ui/Button';

export function GalleryPage() {
  const { data, isLoading, error } = useGallery({ limit: 20 });
  const deleteImage = useDeleteImage();
  const addToast = useUIStore((s) => s.addToast);

  if (isLoading) return <div className="flex justify-center py-12"><Spinner size="lg" /></div>;
  if (error) return <div className="text-center py-12 text-red-600">Failed to load gallery</div>;

  return (
    <div className="max-w-6xl mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Gallery</h1>
      {data?.images.length === 0 && (
        <p className="text-center text-gray-500 py-12">No images yet. Generate your first image!</p>
      )}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
        {data?.images.map((img) => (
          <Card key={img.id} className="overflow-hidden group">
            <img src={img.imageUrl} alt={img.prompt} className="w-full aspect-square object-cover" loading="lazy" />
            <div className="p-3">
              <p className="text-sm text-gray-600 dark:text-gray-400 line-clamp-2">{img.prompt}</p>
              <p className="text-xs text-gray-400 mt-1">{img.model}</p>
              <Button
                variant="danger"
                size="sm"
                className="mt-2 opacity-0 group-hover:opacity-100 transition-opacity"
                onClick={() => {
                  deleteImage.mutate(img.id, {
                    onSuccess: () => addToast({ type: 'success', message: 'Image deleted' }),
                  });
                }}
              >
                Delete
              </Button>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

### `src/pages/SettingsPage.tsx`

```tsx
import { useTheme } from '@/hooks/useTheme';
import { useAuth } from '@/auth/useAuth';
import { useUIStore } from '@/stores/uiStore';
import { Card, CardContent, CardHeader } from '@/components/ui/Card';
import { Button } from '@/components/ui/Button';

export function SettingsPage() {
  const { theme, setTheme, isDark } = useTheme();
  const { user, logout } = useAuth();

  return (
    <div className="max-w-2xl mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Settings</h1>

      <Card className="mb-6">
        <CardHeader>Appearance</CardHeader>
        <CardContent>
          <div className="space-y-3">
            {(['system', 'light', 'dark'] as const).map((t) => (
              <label key={t} className="flex items-center gap-3 cursor-pointer">
                <input
                  type="radio"
                  name="theme"
                  value={t}
                  checked={theme === t}
                  onChange={() => setTheme(t)}
                  className="h-4 w-4 text-primary-600"
                />
                <span className="capitalize">{t}</span>
                {t === 'system' && <span className="text-sm text-gray-500">(currently {isDark ? 'dark' : 'light'})</span>}
              </label>
            ))}
          </div>
        </CardContent>
      </Card>

      <Card className="mb-6">
        <CardHeader>Account</CardHeader>
        <CardContent>
          <p className="text-sm text-gray-600 dark:text-gray-400">Signed in as {user?.email}</p>
          <Button variant="danger" className="mt-4" onClick={logout}>Sign Out</Button>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## 11. App Routing

### `src/App.tsx`

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from '@/auth/AuthProvider';
import { ProtectedRoute } from '@/auth/ProtectedRoute';
import { ThemeInitializer } from '@/hooks/useTheme';
import { ToastContainer } from '@/components/ui/Toast';
import { Navbar } from '@/components/features/Navbar';
import { LoginPage } from '@/pages/LoginPage';
import { SignupPage } from '@/pages/SignupPage';
import { GeneratePage } from '@/pages/GeneratePage';
import { GalleryPage } from '@/pages/GalleryPage';
import { SettingsPage } from '@/pages/SettingsPage';

export default function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <ThemeInitializer />
        <ToastContainer />
        <div className="min-h-screen bg-[var(--color-bg)] text-[var(--color-text)]">
          <Navbar />
          <main className="pt-16">
            <Routes>
              <Route path="/login" element={<LoginPage />} />
              <Route path="/signup" element={<SignupPage />} />
              <Route path="/generate" element={<ProtectedRoute><GeneratePage /></ProtectedRoute>} />
              <Route path="/gallery" element={<ProtectedRoute><GalleryPage /></ProtectedRoute>} />
              <Route path="/settings" element={<ProtectedRoute><SettingsPage /></ProtectedRoute>} />
              <Route path="*" element={<Navigate to="/generate" replace />} />
            </Routes>
          </main>
        </div>
      </AuthProvider>
    </BrowserRouter>
  );
}
```

---

## 12. Accessibility Basics

- All images must have `alt` text describing the content
- Interactive elements have visible focus indicators (`focus-visible:ring-2`)
- Modals trap focus and respond to Escape key
- Buttons and links have descriptive text or `aria-label`
- Color contrast meets WCAG AA (4.5:1 for text)
- Forms associate labels with inputs via `htmlFor`/`id`
- Toast notifications use `role="alert"` and `aria-live="polite"`
- Keyboard navigation works for all interactive elements
- `prefers-reduced-motion` media query disables animations:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 13. PWA Considerations

Install the Vite PWA plugin:

```bash
npm install -D vite-plugin-pwa
```

Add to `vite.config.ts`:

```typescript
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'apple-touch-icon.png'],
      manifest: {
        name: 'FastDepo',
        short_name: 'FastDepo',
        description: 'AI Image Generation',
        theme_color: '#2563eb',
        background_color: '#ffffff',
        display: 'standalone',
        icons: [
          { src: '/pwa-192x192.png', sizes: '192x192', type: 'image/png' },
          { src: '/pwa-512x512.png', sizes: '512x512', type: 'image/png' },
        ],
      },
      workbox: {
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\./i,
            handler: 'NetworkFirst',
            options: { cacheName: 'api-cache', expiration: { maxEntries: 100, maxAgeSeconds: 300 } },
          },
        ],
      },
    }),
  ],
});
```

---

## 14. Environment Configuration

Create environment files:

### `.env`

```
VITE_API_URL=http://localhost:3000/api
VITE_APP_NAME=FastDepo
```

### `.env.staging`

```
VITE_API_URL=https://staging-api.fastdepo.com/api
VITE_APP_NAME=FastDepo (Staging)
```

### `.env.production`

```
VITE_API_URL=https://api.fastdepo.com/api
VITE_APP_NAME=FastDepo
```

Access in code via `import.meta.env.VITE_API_URL`. Never put secrets in these files — they are embedded in the client bundle.

---

## 15. Build and Deploy

### Development

```bash
npm run dev
```

### Production Build

```bash
npm run build
```

This outputs to `dist/`. Preview the production build locally:

```bash
npm run preview
```

### Deploy to Vercel

```bash
npm install -g vercel
vercel --prod
```

Or connect the GitHub repo to Vercel dashboard for automatic deploys. Set environment variables in the Vercel project settings.

### Deploy to Netlify

Create `netlify.toml`:

```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### Docker Deployment

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

`nginx.conf` for SPA routing:

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://backend:3000;
    }
}
```

Build and run:

```bash
docker build -t fastdepo-web .
docker run -p 8080:80 fastdepo-web
```