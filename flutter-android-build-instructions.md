# Flutter Android App Build Guide

Complete step-by-step guide to building the FastDepo Android app using Flutter.

---

## 1. Flutter SDK Setup

### Install Flutter

```bash
# macOS
brew install --cask flutter

# Or manually:
git clone https://github.com/flutter/flutter.git -b stable ~/flutter
export PATH="$HOME/flutter/bin:$PATH"

# Verify installation
flutter doctor
```

### Install Android Studio

1. Download from https://developer.android.com/studio
2. Install Android SDK via Android Studio SDK Manager
3. Accept licenses:

```bash
flutter doctor --android-licenses
```

### Verify Setup

```bash
flutter doctor -v
```

All items should show green checkmarks. Address any missing components.

---

## 2. Project Creation

```bash
flutter create --org com.fastdepo --project-name fastdepo --platforms android fastdepo_app
cd fastdepo_app
```

### Install Dependencies

Add to `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.7.0
  flutter_riverpod: ^2.6.0
  riverpod_annotation: ^2.6.0
  go_router: ^14.8.0
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  path_provider: ^2.1.5
  firebase_core: ^3.12.0
  firebase_messaging: ^15.2.5
  flutter_local_notifications: ^18.0.1
  cached_network_image: ^3.4.1
  shimmer: ^3.0.0
  share_plus: ^10.1.4
  url_launcher: ^6.3.1
  intl: ^0.19.0
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.14
  freezed: ^2.5.8
  json_serializable: ^6.9.4
  riverpod_generator: ^2.6.3
  hive_generator: ^2.0.1
  flutter_lints: ^5.0.0
```

```bash
flutter pub get
```

---

## 3. Folder Structure

```
lib/
├── core/
│   ├── api/
│   │   ├── dio_client.dart          # Dio instance setup
│   │   ├── auth_interceptor.dart     # Token injection + refresh
│   │   └── api_result.dart           # Success/error wrapper
│   ├── auth/
│   │   ├── auth_service.dart         # Login/signup/logout
│   │   └── token_storage.dart        # Secure token storage
│   ├── theme/
│   │   ├── app_theme.dart            # Material 3 theme
│   │   └── theme_provider.dart       # Dark/light mode
│   └── router/
│       └── app_router.dart           # GoRouter configuration
├── features/
│   ├── auth/
│   │   ├── screens/
│   │   │   ├── login_screen.dart
│   │   │   └── signup_screen.dart
│   │   └── providers/
│   │       └── auth_provider.dart
│   ├── generation/
│   │   ├── screens/
│   │   │   ├── home_screen.dart
│   │   │   └── generation_result_screen.dart
│   │   ├── providers/
│   │   │   └── generation_provider.dart
│   │   └── widgets/
│   │       ├── prompt_input.dart
│   │       └── model_selector.dart
│   ├── gallery/
│   │   ├── screens/
│   │   │   ├── gallery_screen.dart
│   │   │   └── image_detail_screen.dart
│   │   ├── providers/
│   │   │   └── gallery_provider.dart
│   │   └── widgets/
│   │       └── image_card.dart
│   └── settings/
│       ├── screens/
│       │   └── settings_screen.dart
│       └── providers/
│           └── settings_provider.dart
├── shared/
│   ├── widgets/
│   │   ├── app_button.dart
│   │   ├── app_input.dart
│   │   ├── loading_overlay.dart
│   │   └── error_widget.dart
│   └── extensions/
│       ├── context_extensions.dart
│       └── string_extensions.dart
└── main.dart
```

---

## 4. Dio API Client with Auth Interceptor

### `lib/core/api/dio_client.dart`

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'auth_interceptor.dart';

final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: const String.fromEnvironment('API_URL', defaultValue: 'https://api.fastdepo.com/api'),
    connectTimeout: const Duration(seconds: 30),
    receiveTimeout: const Duration(seconds: 120),
    sendTimeout: const Duration(seconds: 30),
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
    },
  ));

  dio.interceptors.addAll([
    AuthInterceptor(ref),
    LogInterceptor(
      requestBody: true,
      responseBody: true,
    ),
  ]);

  return dio;
});
```

### `lib/core/api/auth_interceptor.dart`

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../auth/token_storage.dart';

class AuthInterceptor extends Interceptor {
  final Ref _ref;

  AuthInterceptor(this._ref);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final tokenStorage = _ref.read(tokenStorageProvider);
    final accessToken = await tokenStorage.getAccessToken();

    if (accessToken != null) {
      options.headers['Authorization'] = 'Bearer $accessToken';
    }

    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      final tokenStorage = _ref.read(tokenStorageProvider);
      final refreshToken = await tokenStorage.getRefreshToken();

      if (refreshToken == null) {
        await tokenStorage.clear();
        handler.reject(err);
        return;
      }

      try {
        final dio = Dio(BaseOptions(
          baseUrl: err.requestOptions.baseUrl,
          headers: {'Content-Type': 'application/json'},
        ));

        final response = await dio.post('/auth/refresh', data: {
          'refreshToken': refreshToken,
        });

        final newAccessToken = response.data['accessToken'] as String;
        final newRefreshToken = response.data['refreshToken'] as String;

        await tokenStorage.saveTokens(newAccessToken, newRefreshToken);

        final retryOptions = err.requestOptions;
        retryOptions.headers['Authorization'] = 'Bearer $newAccessToken';

        final retryResponse = await dio.fetch(retryOptions);
        handler.resolve(retryResponse);
      } on DioException catch (refreshError) {
        await tokenStorage.clear();
        handler.reject(refreshError);
      }
    } else {
      handler.next(err);
    }
  }
}
```

### `lib/core/api/api_result.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_result.freezed.dart';

@freezed
sealed class ApiResult<T> with _$ApiResult<T> {
  const factory ApiResult.success(T data) = Success<T>;
  const factory ApiResult.failure(String message, {int? statusCode}) = Failure<T>;
  const factory ApiResult.loading() = Loading<T>;
}
```

### `lib/core/auth/token_storage.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class TokenStorage {
  static const _accessTokenKey = 'fastdepo_access_token';
  static const _refreshTokenKey = 'fastdepo_refresh_token';

  final FlutterSecureStorage _storage;

  TokenStorage(this._storage);

  Future<String?> getAccessToken() async {
    return await _storage.read(key: _accessTokenKey);
  }

  Future<String?> getRefreshToken() async {
    return await _storage.read(key: _refreshTokenKey);
  }

  Future<void> saveTokens(String accessToken, String refreshToken) async {
    await _storage.write(key: _accessTokenKey, value: accessToken);
    await _storage.write(key: _refreshTokenKey, value: refreshToken);
  }

  Future<void> clear() async {
    await _storage.delete(key: _accessTokenKey);
    await _storage.delete(key: _refreshTokenKey);
  }

  Future<bool> hasAccessToken() async {
    return await getAccessToken() != null;
  }
}

final tokenStorageProvider = Provider<TokenStorage>((ref) {
  const storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  return TokenStorage(storage);
});
```

Add `flutter_secure_storage` to `pubspec.yaml`:

```yaml
dependencies:
  flutter_secure_storage: ^9.2.4
```

### `lib/core/auth/auth_service.dart`

```dart
import 'package:dio/dio.dart';
import 'api_result.dart';
import '../auth/token_storage.dart';

class AuthService {
  final Dio _dio;
  final TokenStorage _tokenStorage;

  AuthService(this._dio, this._tokenStorage);

  Future<ApiResult<Map<String, dynamic>>> login({
    required String email,
    required String password,
  }) async {
    try {
      final response = await _dio.post('/auth/login', data: {
        'email': email,
        'password': password,
      });

      final data = response.data as Map<String, dynamic>;
      await _tokenStorage.saveTokens(
        data['accessToken'] as String,
        data['refreshToken'] as String,
      );

      return ApiResult.success(data['user'] as Map<String, dynamic>);
    } on DioException catch (e) {
      return ApiResult.failure(
        e.response?.data['message'] ?? 'Login failed',
        statusCode: e.response?.statusCode,
      );
    }
  }

  Future<ApiResult<Map<String, dynamic>>> signup({
    required String email,
    required String password,
    required String name,
  }) async {
    try {
      final response = await _dio.post('/auth/signup', data: {
        'email': email,
        'password': password,
        'name': name,
      });

      final data = response.data as Map<String, dynamic>;
      await _tokenStorage.saveTokens(
        data['accessToken'] as String,
        data['refreshToken'] as String,
      );

      return ApiResult.success(data['user'] as Map<String, dynamic>);
    } on DioException catch (e) {
      return ApiResult.failure(
        e.response?.data['message'] ?? 'Signup failed',
        statusCode: e.response?.statusCode,
      );
    }
  }

  Future<void> logout() async {
    try {
      await _dio.post('/auth/logout');
    } finally {
      await _tokenStorage.clear();
    }
  }

  Future<ApiResult<Map<String, dynamic>>> getCurrentUser() async {
    try {
      final response = await _dio.get('/auth/me');
      return ApiResult.success(response.data as Map<String, dynamic>);
    } on DioException catch (e) {
      return ApiResult.failure(
        e.response?.data['message'] ?? 'Failed to get user',
        statusCode: e.response?.statusCode,
      );
    }
  }
}
```

---

## 5. Riverpod State Management

### `lib/features/auth/providers/auth_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/api/dio_client.dart';
import '../../../core/auth/auth_service.dart';
import '../../../core/auth/token_storage.dart';

enum AuthStatus { initial, authenticated, unauthenticated, loading }

class AuthState {
  final AuthStatus status;
  final Map<String, dynamic>? user;
  final String? error;

  const AuthState({
    this.status = AuthStatus.initial,
    this.user,
    this.error,
  });

  AuthState copyWith({
    AuthStatus? status,
    Map<String, dynamic>? user,
    String? error,
  }) {
    return AuthState(
      status: status ?? this.status,
      user: user ?? this.user,
      error: error ?? this.error,
    );
  }
}

class AuthNotifier extends StateNotifier<AuthState> {
  final AuthService _authService;
  final TokenStorage _tokenStorage;

  AuthNotifier(this._authService, this._tokenStorage) : super(const AuthState()) {
    _checkAuth();
  }

  Future<void> _checkAuth() async {
    final hasToken = await _tokenStorage.hasAccessToken();
    if (!hasToken) {
      state = state.copyWith(status: AuthStatus.unauthenticated);
      return;
    }

    final result = await _authService.getCurrentUser();
    result.when(
      success: (user) {
        state = AuthState(
          status: AuthStatus.authenticated,
          user: user,
        );
      },
      failure: (message, _) {
        state = AuthState(
          status: AuthStatus.unauthenticated,
          error: message,
        );
      },
      loading: () {},
    );
  }

  Future<void> login(String email, String password) async {
    state = state.copyWith(status: AuthStatus.loading);

    final result = await _authService.login(email: email, password: password);
    result.when(
      success: (user) {
        state = AuthState(status: AuthStatus.authenticated, user: user);
      },
      failure: (message, _) {
        state = AuthState(status: AuthStatus.unauthenticated, error: message);
      },
      loading: () {},
    );
  }

  Future<void> signup(String email, String password, String name) async {
    state = state.copyWith(status: AuthStatus.loading);

    final result = await _authService.signup(email: email, password: password, name: name);
    result.when(
      success: (user) {
        state = AuthState(status: AuthStatus.authenticated, user: user);
      },
      failure: (message, _) {
        state = AuthState(status: AuthStatus.unauthenticated, error: message);
      },
      loading: () {},
    );
  }

  Future<void> logout() async {
    await _authService.logout();
    state = const AuthState(status: AuthStatus.unauthenticated);
  }
}

final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  final dio = ref.read(dioProvider);
  final tokenStorage = ref.read(tokenStorageProvider);
  return AuthNotifier(AuthService(dio, tokenStorage), tokenStorage);
});
```

### `lib/features/generation/providers/generation_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:dio/dio.dart';
import '../../../core/api/dio_client.dart';

class GenerationState {
  final bool isGenerating;
  final Map<String, dynamic>? result;
  final String? error;

  const GenerationState({
    this.isGenerating = false,
    this.result,
    this.error,
  });

  GenerationState copyWith({
    bool? isGenerating,
    Map<String, dynamic>? result,
    String? error,
  }) {
    return GenerationState(
      isGenerating: isGenerating ?? this.isGenerating,
      result: result ?? this.result,
      error: error,
    );
  }
}

class GenerationNotifier extends StateNotifier<GenerationState> {
  final Dio _dio;

  GenerationNotifier(this._dio) : super(const GenerationState());

  Future<void> generate({
    required String prompt,
    required String model,
    String? negativePrompt,
    int? width,
    int? height,
  }) async {
    state = const GenerationState(isGenerating: true);

    try {
      final response = await _dio.post('/images/generate', data: {
        'prompt': prompt,
        'model': model,
        if (negativePrompt != null) 'negativePrompt': negativePrompt,
        if (width != null) 'width': width,
        if (height != null) 'height': height,
      });

      state = GenerationState(
        isGenerating: false,
        result: response.data as Map<String, dynamic>,
      );
    } on DioException catch (e) {
      state = GenerationState(
        isGenerating: false,
        error: e.response?.data['message'] ?? 'Generation failed',
      );
    }
  }

  void clearResult() {
    state = const GenerationState();
  }
}

final generationProvider = StateNotifierProvider<GenerationNotifier, GenerationState>((ref) {
  return GenerationNotifier(ref.read(dioProvider));
});
```

### `lib/features/gallery/providers/gallery_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:dio/dio.dart';
import '../../../core/api/dio_client.dart';

final galleryProvider = FutureProvider.family<Map<String, dynamic>, int>((ref, page) async {
  final dio = ref.read(dioProvider);
  final response = await dio.get('/gallery', queryParameters: {
    'page': page,
    'limit': 20,
  });
  return response.data as Map<String, dynamic>;
});

final deleteImageProvider = FutureProvider.family<void, String>((ref, id) async {
  final dio = ref.read(dioProvider);
  await dio.delete('/gallery/$id');
  ref.invalidate(galleryProvider);
});
```

---

## 6. Hive for Offline Caching

### Setup

```dart
// lib/main.dart
import 'package:hive_flutter/hive_flutter.dart';
import 'package:path_provider/path_provider.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();

  // Register adapters if using custom types
  // Hive.registerAdapter(ImageModelAdapter());

  await Hive.openBox('settings');
  await Hive.openBox('cached_images');

  runApp(const ProviderScope(child: FastDepoApp()));
}
```

### Cached Image Repository

```dart
// lib/features/gallery/data/cached_gallery_repository.dart
import 'package:hive/hive.dart';

class CachedGalleryRepository {
  static const _boxName = 'cached_images';

  Future<void> cacheImages(List<Map<String, dynamic>> images) async {
    final box = await Hive.openBox(_boxName);
    for (final image in images) {
      await box.put(image['id'], image);
    }
  }

  Future<List<Map<String, dynamic>>> getCachedImages() async {
    final box = await Hive.openBox(_boxName);
    return box.values.cast<Map<String, dynamic>>().toList();
  }

  Future<void> clearCache() async {
    final box = await Hive.openBox(_boxName);
    await box.clear();
  }
}
```

---

## 7. Screens

### `lib/features/auth/screens/login_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../providers/auth_provider.dart';

class LoginScreen extends ConsumerStatefulWidget {
  const LoginScreen({super.key});

  @override
  ConsumerState<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends ConsumerState<LoginScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _formKey = GlobalKey<FormState>();

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authProvider);

    return Scaffold(
      body: SafeArea(
        child: Center(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(24),
            child: Form(
              key: _formKey,
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text('FastDepo', style: Theme.of(context).textTheme.headlineLarge),
                  const SizedBox(height: 8),
                  Text('Sign in to continue', style: Theme.of(context).textTheme.bodyMedium),
                  const SizedBox(height: 32),
                  TextFormField(
                    controller: _emailController,
                    decoration: const InputDecoration(labelText: 'Email', border: OutlineInputBorder()),
                    keyboardType: TextInputType.emailAddress,
                    validator: (v) => v?.isEmpty ?? true ? 'Enter your email' : null,
                  ),
                  const SizedBox(height: 16),
                  TextFormField(
                    controller: _passwordController,
                    decoration: const InputDecoration(labelText: 'Password', border: OutlineInputBorder()),
                    obscureText: true,
                    validator: (v) => (v?.length ?? 0) < 8 ? 'Password must be 8+ characters' : null,
                  ),
                  const SizedBox(height: 24),
                  if (authState.error != null)
                    Padding(
                      padding: const EdgeInsets.only(bottom: 16),
                      child: Text(authState.error!, style: TextStyle(color: Theme.of(context).colorScheme.error)),
                    ),
                  SizedBox(
                    width: double.infinity,
                    child: FilledButton(
                      onPressed: authState.status == AuthStatus.loading
                          ? null
                          : () {
                              if (_formKey.currentState!.validate()) {
                                ref.read(authProvider.notifier).login(
                                      _emailController.text,
                                      _passwordController.text,
                                    );
                              }
                            },
                      child: authState.status == AuthStatus.loading
                          ? const SizedBox(height: 20, width: 20, child: CircularProgressIndicator(strokeWidth: 2))
                          : const Text('Sign In'),
                    ),
                  ),
                  const SizedBox(height: 16),
                  TextButton(
                    onPressed: () => context.go('/signup'),
                    child: const Text("Don't have an account? Sign up"),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

### `lib/features/generation/screens/home_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/generation_provider.dart';
import '../widgets/prompt_input.dart';
import '../widgets/model_selector.dart';

class HomeScreen extends ConsumerStatefulWidget {
  const HomeScreen({super.key});

  @override
  ConsumerState<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends ConsumerState<HomeScreen> {
  String _selectedModel = 'stable-diffusion-xl';

  @override
  Widget build(BuildContext context) {
    final genState = ref.watch(generationProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Generate Image')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            ModelSelector(
              selectedModel: _selectedModel,
              onModelChanged: (model) => setState(() => _selectedModel = model),
            ),
            const SizedBox(height: 16),
            PromptInput(
              isGenerating: genState.isGenerating,
              onGenerate: (prompt) {
                ref.read(generationProvider.notifier).generate(
                      prompt: prompt,
                      model: _selectedModel,
                    );
              },
            ),
            const SizedBox(height: 24),
            if (genState.isGenerating)
              const Center(child: Column(children: [CircularProgressIndicator(), SizedBox(height: 8), Text('Generating...')])),
            if (genState.error != null)
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(color: Theme.of(context).colorScheme.errorContainer, borderRadius: BorderRadius.circular(8)),
                child: Text(genState.error!, style: TextStyle(color: Theme.of(context).colorScheme.onErrorContainer)),
              ),
            if (genState.result != null) ...[
              const SizedBox(height: 16),
              ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: Image.network(
                  genState.result!['imageUrl'] as String,
                  fit: BoxFit.contain,
                  loadingBuilder: (_, child, progress) =>
                      progress == null ? child : const Center(child: CircularProgressIndicator()),
                  errorBuilder: (_, _, _) => const Icon(Icons.broken_image, size: 64),
                ),
              ),
              const SizedBox(height: 8),
              Text(
                genState.result!['prompt'] as String,
                style: Theme.of(context).textTheme.bodySmall,
                maxLines: 2,
                overflow: TextOverflow.ellipsis,
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## 8. Navigation with GoRouter

### `lib/core/router/app_router.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../features/auth/providers/auth_provider.dart';
import '../../features/auth/screens/login_screen.dart';
import '../../features/auth/screens/signup_screen.dart';
import '../../features/generation/screens/home_screen.dart';
import '../../features/gallery/screens/gallery_screen.dart';
import '../../features/settings/screens/settings_screen.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    initialLocation: '/',
    redirect: (context, state) {
      final isAuthenticated = authState.status == AuthStatus.authenticated;
      final isAuthRoute = state.matchedLocation == '/login' || state.matchedLocation == '/signup';

      if (authState.status == AuthStatus.initial) return null;

      if (!isAuthenticated && !isAuthRoute) return '/login';
      if (isAuthenticated && isAuthRoute) return '/';

      return null;
    },
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomeScreen(),
      ),
      GoRoute(
        path: '/gallery',
        builder: (context, state) => const GalleryScreen(),
      ),
      GoRoute(
        path: '/settings',
        builder: (context, state) => const SettingsScreen(),
      ),
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/signup',
        builder: (context, state) => const SignupScreen(),
      ),
    ],
  );
});
```

### `lib/main.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'core/router/app_router.dart';
import 'core/theme/app_theme.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await Hive.openBox('settings');

  runApp(const ProviderScope(child: FastDepoApp()));
}

class FastDepoApp extends ConsumerWidget {
  const FastDepoApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'FastDepo',
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system,
      routerConfig: router,
      debugShowCheckedModeBanner: false,
    );
  }
}
```

---

## 9. Theme Configuration (Material 3)

### `lib/core/theme/app_theme.dart`

```dart
import 'package:flutter/material.dart';

class AppTheme {
  static final lightTheme = ThemeData(
    useMaterial3: true,
    colorSchemeSeed: const Color(0xFF2563EB),
    brightness: Brightness.light,
    scaffoldBackgroundColor: const Color(0xFFF8FAFC),
    appBarTheme: const AppBarTheme(
      centerTitle: true,
      elevation: 0,
      backgroundColor: Colors.white,
      foregroundColor: Color(0xFF0F172A),
    ),
    cardTheme: CardTheme(
      elevation: 1,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      margin: const EdgeInsets.symmetric(vertical: 4),
    ),
    inputDecorationTheme: InputDecorationTheme(
      filled: true,
      fillColor: Colors.white,
      border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
      contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
    ),
    filledButtonTheme: FilledButtonThemeData(
      style: FilledButton.styleFrom(
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
      ),
    ),
  );

  static final darkTheme = ThemeData(
    useMaterial3: true,
    colorSchemeSeed: const Color(0xFF2563EB),
    brightness: Brightness.dark,
    scaffoldBackgroundColor: const Color(0xFF0F172A),
    appBarTheme: const AppBarTheme(
      centerTitle: true,
      elevation: 0,
      backgroundColor: Color(0xFF1E293B),
      foregroundColor: Colors.white,
    ),
    cardTheme: CardTheme(
      elevation: 1,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      margin: const EdgeInsets.symmetric(vertical: 4),
    ),
  );
}
```

---

## 10. Push Notifications with Firebase Cloud Messaging

### Setup Firebase

```bash
flutter pub add firebase_core firebase_messaging
```

Create a Firebase project at https://console.firebase.google.com, add an Android app with your package name (`com.fastdepo.fastdepo_app`), and download `google-services.json` to `android/app/`.

### `android/build.gradle` (project-level)

Add to the top:

```groovy
buildscript {
  dependencies {
    classpath 'com.google.gms:google-services:4.4.2'
  }
}
```

### `android/app/build.gradle`

```groovy
apply plugin: 'com.google.gms.google-services'
```

### Notification Handler

```dart
// lib/core/notifications/notification_service.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

class NotificationService {
  static final FlutterLocalNotificationsPlugin _localNotifications = FlutterLocalNotificationsPlugin();

  static Future<void> initialize() async {
    await Firebase.initializeApp();

    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings();
    await _localNotifications.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
    );

    const androidChannel = AndroidNotificationChannel(
      'generation_channel',
      'Image Generation',
      description: 'Notifications when images finish generating',
      importance: Importance.high,
    );
    await _localNotifications.resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(androidChannel);

    FirebaseMessaging.onMessage.listen((message) {
      final notification = message.notification;
      if (notification != null) {
        _localNotifications.show(
          message.hashCode,
          notification.title,
          notification.body,
          NotificationDetails(
            android: AndroidNotificationDetails(
              androidChannel.id,
              androidChannel.name,
              channelDescription: androidChannel.description,
              icon: '@mipmap/ic_launcher',
            ),
          ),
        );
      }
    });

    FirebaseMessaging.onMessageOpenedApp.listen((message) {
      // Handle notification tap: navigate to relevant screen
    });

    final token = await FirebaseMessaging.instance.getToken();
    // Send token to backend for push notification targeting
  }
}
```

Register in `main.dart`:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService.initialize();
  // ...
}
```

---

## 11. Android-Specific Configuration

### `android/app/src/main/AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <application
        android:label="FastDepo"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher"
        android:usesCleartextTraffic="false">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|layoutDirection|locale"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">

            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme" />

            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <meta-data
            android:name="com.google.firebase.messaging.default_notification_channel_id"
            android:value="generation_channel" />

        <meta-data
            android:name="com.google.firebase.messaging.default_notification_icon"
            android:resource="@mipmap/ic_launcher" />
    </application>
</manifest>
```

### Proguard Rules

`android/app/proguard-rules.pro`:

```
# Keep Dio and model classes
-keep class com.fastdepo.** { *; }
-keep class retrofit2.** { *; }
-keepattributes Signature
-keepattributes Exceptions

# Flutter
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }
-keep class io.flutter.** { *; }
```

Enable in `android/app/build.gradle`:

```groovy
android {
    // ...
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
    }
}
```

### Signing Configuration

Create a keystore:

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

Create `android/key.properties`:

```
storePassword=YOUR_STORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=upload
storeFile=/Users/YOUR_USER/upload-keystore.jks
```

Add to `android/app/build.gradle`:

```groovy
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

---

## 12. Building Release APK and AAB

### Release APK

```bash
flutter build apk --release
```

Output: `build/app/outputs/flutter-apk/app-release.apk`

### Release AAB (for Play Store)

```bash
flutter build appbundle --release
```

Output: `build/app/outputs/bundle/release/app-release.aab`

### Build with Specific Flavor

```bash
flutter build apk --flavor production --release
flutter build appbundle --flavor production --release
```

---

## 13. Google Play Store Submission

### Prerequisites

1. Google Play Developer account ($25 one-time fee): https://play.google.com/console/signup
2. App signing: Use Google Play App Signing (recommended) or upload your own keystore

### Store Listing

Prepare the following assets:

- **App name**: FastDepo
- **Short description** (80 chars): AI-powered image generation in seconds
- **Full description** (4000 chars): Detailed description of features
- **App icon**: 512x512 PNG
- **Feature graphic**: 1024x500 PNG
- **Screenshots**: At least 2, recommended 4-8 per phone/tablet form factor
  - Phone: 16:9 aspect ratio, minimum 320px, maximum 3840px
  - Tablet 7": 600x1024 minimum
  - Tablet 10": 800x1280 minimum
- **Category**: Art & Design
- **Content rating**: Complete IARC questionnaire (likely Everyone or Teen depending on content policy)
- **Privacy policy URL**: Required
- **Contact details**: Email, phone (optional)

### Upload and Review

```bash
# Using fastlane (optional)
gem install fastlane -NV
fastlane init

# Or manual upload via Google Play Console
# 1. Go to https://play.google.com/console
# 2. Create app > fill in details
# 3. App content > complete ratings and privacy
# 4. Production > Create new release > upload AAB
# 5. Review release > Start rollout
```

### Content Rating

Complete the IARC questionnaire:
- Violent content: None
- Sexual content: None (or minimal for AI art)
- Controlled substances: None
- Language: No profanity
- User interaction: Yes (if sharing features exist)
- Data sharing: Yes (if analytics exist)
- Digital goods purchases: Yes (if in-app purchases)

---

## 14. Testing on Real Devices

### Enable Developer Options on Android

1. Settings > About phone > Tap "Build number" 7 times
2. Settings > Developer options > Enable USB debugging

### Run on Connected Device

```bash
# List connected devices
flutter devices

# Run on device
flutter run --release

# Run on specific device
flutter run -d <device-id>
```

### Run Integration Tests

```bash
flutter test integration_test/
```

### Debug on Device

```bash
# View logs
adb logcat -s flutter

# Check network traffic
adb install proxy-cert.pem  # Install Charles/proxyman cert

# Performance overlay
flutter run --profile --dart-define=PERF_OVERLAY=true
```

---

## 15. Common Pitfalls and Debugging

### Gradle Issues

```bash
# Clean everything
flutter clean
cd android && ./gradlew clean && cd ..

# Clear gradle cache
rm -rf ~/.gradle/caches/

# Update gradle wrapper
cd android && ./gradlew wrapper --gradle-version=8.5 && cd ..
```

### Hot Reload Not Working

- Ensure `flutter run` (debug mode), not `flutter run --release`
- Check that the file is saved
- Press `r` in terminal for hot reload, `R` for hot restart

### Common Errors

| Error | Fix |
|-------|-----|
| `MIN_SDK_VERSION` | Set `minSdkVersion 21` in `android/app/build.gradle` |
| `MULTIDEX` | Add `multiDexEnabled true` and `implementation 'androidx.multidex:multidex:2.0.1'` |
| `google-services.json` missing | Download from Firebase Console and place in `android/app/` |
| Keystore not found | Verify `key.properties` paths are absolute |
| Proguard stripping classes | Add `-keep class` rules for affected classes |
| Plugin not registered | Run `flutter clean && flutter pub get` |
| Network cleartext error | Set `android:usesCleartextTraffic="true"` in debug manifest only |

### Performance Tips

- Use `const` constructors everywhere possible
- Use `ListView.builder` for long lists (not `Column` + `SingleChildScrollView`)
- Use `cached_network_image` for remote images
- Profile with `flutter run --profile` and Dart DevTools
- Avoid `setState` in deeply nested widgets; prefer Riverpod

### Build Size Optimization

```bash
# Analyze app size
flutter build apk --analyze-size

# Use app bundle (30-50% smaller than APK)
flutter build appbundle --release

# Remove未used resources (handled by shrinkResources in build.gradle)
```