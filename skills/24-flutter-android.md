# Skill 24: Flutter Android App

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 4-6 hours
Depends On: 03, 05

## Input Contract
- Skill 03 complete: auth API endpoints operational (register, login, refresh, logout, me)
- Skill 05 complete: image generation and CRUD API endpoints operational
- Backend accessible at a known base URL (e.g., `https://api.yourdomain.com`)
- Flutter SDK >= 3.16 installed
- Android SDK and toolchain configured
- A Google Play Console account (for Play Store submission)

## Output Contract
- Flutter project in `app/` directory with feature-first architecture
- Project structure: `features/auth`, `features/generate`, `features/gallery`, `core/network`, `core/storage`, `core/theme`
- Dio API client with auth interceptor (token injection, refresh rotation, error handling)
- Riverpod state management for all features
- Hive offline caching for image history
- Auth flow: login, signup, logout, token refresh
- Image generation screen with provider selection
- Gallery screen with grid view and detail view
- Settings screen with theme toggle (dark/light/system)
- Android signing configuration and release build setup
- Play Store submission preparation steps

## Files to Create

| File | Description |
|------|-------------|
| `app/pubspec.yaml` | Flutter dependencies and project config |
| `app/lib/main.dart` | App entry point with providers |
| `app/lib/app.dart` | MaterialApp with routing and theme |
| `app/lib/core/network/dio_client.dart` | Dio HTTP client with interceptors |
| `app/lib/core/network/api_interceptor.dart` | Auth token injection and refresh |
| `app/lib/core/network/api_result.dart` | Result type for API calls |
| `app/lib/core/storage/hive_service.dart` | Hive local storage service |
| `app/lib/core/storage/auth_storage.dart` | Secure token storage |
| `app/lib/core/theme/app_theme.dart` | Light and dark themes |
| `app/lib/core/theme/theme_provider.dart` | Riverpod theme notifier |
| `app/lib/core/router/app_router.dart` | GoRouter configuration |
| `app/lib/core/config/env.dart` | Environment configuration |
| `app/lib/features/auth/models/user_model.dart` | User data model |
| `app/lib/features/auth/models/auth_response.dart` | Auth response model |
| `app/lib/features/auth/providers/auth_provider.dart` | Riverpod auth state |
| `app/lib/features/auth/repositories/auth_repository.dart` | Auth API calls |
| `app/lib/features/auth/screens/login_screen.dart` | Login UI |
| `app/lib/features/auth/screens/signup_screen.dart` | Signup UI |
| `app/lib/features/auth/widgets/auth_form.dart` | Shared auth form widget |
| `app/lib/features/generate/models/image_model.dart` | Image data model |
| `app/lib/features/generate/providers/generate_provider.dart` | Generate state management |
| `app/lib/features/generate/repositories/image_repository.dart` | Image API calls |
| `app/lib/features/generate/screens/generate_screen.dart` | Image generation UI |
| `app/lib/features/generate/widgets/provider_selector.dart` | AI provider picker |
| `app/lib/features/gallery/providers/gallery_provider.dart` | Gallery state management |
| `app/lib/features/gallery/repositories/gallery_repository.dart` | Gallery API calls |
| `app/lib/features/gallery/screens/gallery_screen.dart` | Gallery listing UI |
| `app/lib/features/gallery/screens/image_detail_screen.dart` | Image detail UI |
| `app/lib/features/gallery/widgets/image_card.dart` | Grid image card |
| `app/lib/features/settings/screens/settings_screen.dart` | Settings with theme toggle |
| `app/android/app/build.gradle` | Android build config with signing |
| `app/android/key.properties.example` | Signing config template |
| `app/.env.example` | Environment variables template |

## Steps

### Step 1: Create Flutter Project

```bash
flutter create --org com.fastdepo --platforms android app
cd app
```

### Step 2: Configure pubspec.yaml

`app/pubspec.yaml` (add under `dependencies` and `dev_dependencies`):

```yaml
dependencies:
  flutter:
    sdk: flutter
  dio: ^5.4.0
  riverpod: ^2.4.9
  flutter_riverpod: ^2.4.9
  go_router: ^13.0.0
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  flutter_secure_storage: ^9.0.0
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1
  cached_network_image: ^3.3.0
  shimmer: ^3.0.0
  pull_to_refresh_flutter3: ^2.0.0
  image_gallery_saver: ^2.0.3
  permission_handler: ^11.1.0
  path_provider: ^2.1.1
  share_plus: ^7.2.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1
  build_runner: ^2.4.7
  freezed: ^2.4.5
  json_serializable: ^6.7.1
  hive_generator: ^2.0.1
```

### Step 3: Environment Configuration

`app/lib/core/config/env.dart`:

```dart
class Env {
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'https://api.yourdomain.com',
  );

  static const String apiVersion = '/api';

  static String get fullBaseUrl => '$apiBaseUrl$apiVersion';
}
```

### Step 4: Dio Client with Auth Interceptor

`app/lib/core/network/dio_client.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'api_interceptor.dart';
import '../config/env.dart';

final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: Env.fullBaseUrl,
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 60),
      headers: {'Content-Type': 'application/json'},
    ),
  );

  dio.interceptors.addAll([
    AuthInterceptor(ref),
    LogInterceptor(requestBody: true, responseBody: true),
  ]);

  return dio;
});
```

`app/lib/core/network/api_interceptor.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../storage/auth_storage.dart';

class AuthInterceptor extends Interceptor {
  final Ref _ref;

  AuthInterceptor(this._ref);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final authStorage = _ref.read(authStorageProvider);
    final token = await authStorage.getAccessToken();

    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }

    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      final authStorage = _ref.read(authStorageProvider);
      final refreshToken = await authStorage.getRefreshToken();

      if (refreshToken == null) {
        handler.next(err);
        return;
      }

      try {
        final dio = Dio(BaseOptions(baseUrl: Env.fullBaseUrl));
        final response = await dio.post('/auth/refresh', data: {
          'refreshToken': refreshToken,
        });

        final newAccessToken = response.data['accessToken'];
        final newRefreshToken = response.data['refreshToken'];

        await authStorage.saveTokens(
          accessToken: newAccessToken,
          refreshToken: newRefreshToken,
        );

        err.requestOptions.headers['Authorization'] = 'Bearer $newAccessToken';
        final retryResponse = await dio.fetch(err.requestOptions);
        handler.resolve(retryResponse);
      } catch (_) {
        await authStorage.clearTokens();
        handler.next(err);
      }
    } else {
      handler.next(err);
    }
  }
}
```

`app/lib/core/network/api_result.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'api_result.freezed.dart';

@freezed
class ApiResult<T> with _$ApiResult<T> {
  const factory ApiResult.success(T data) = Success<T>;
  const factory ApiResult.failure(String message, {int? statusCode}) = Failure<T>;
}
```

### Step 5: Secure Auth Storage

`app/lib/core/storage/auth_storage.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final authStorageProvider = Provider<AuthStorage>((ref) => AuthStorage());

class AuthStorage {
  static const _accessTokenKey = 'access_token';
  static const _refreshTokenKey = 'refresh_token';
  static const _userIdKey = 'user_id';

  final _storage = const FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await _storage.write(key: _accessTokenKey, value: accessToken);
    await _storage.write(key: _refreshTokenKey, value: refreshToken);
  }

  Future<String?> getAccessToken() => _storage.read(key: _accessTokenKey);
  Future<String?> getRefreshToken() => _storage.read(key: _refreshTokenKey);

  Future<void> clearTokens() async {
    await _storage.delete(key: _accessTokenKey);
    await _storage.delete(key: _refreshTokenKey);
    await _storage.delete(key: _userIdKey);
  }

  Future<bool> isAuthenticated() async {
    final token = await getAccessToken();
    return token != null;
  }
}
```

### Step 6: Hive Caching Service

`app/lib/core/storage/hive_service.dart`:

```dart
import 'package:hive_flutter/hive_flutter.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final hiveServiceProvider = Provider<HiveService>((ref) => HiveService());

class HiveService {
  static const String _imageBox = 'images';
  static const String _galleryBox = 'galleries';
  static const String _cacheBox = 'cache';

  Future<void> init() async {
    await Hive.initFlutter();
    await Hive.openBox<Map>(_imageBox);
    await Hive.openBox<Map>(_galleryBox);
    await Hive.openBox<Map>(_cacheBox);
  }

  Future<void> cacheImages(List<Map<String, dynamic>> images) async {
    final box = Hive.box<Map>(_imageBox);
    for (final image in images) {
      await box.put(image['id'], image);
    }
  }

  List<Map> getCachedImages() {
    final box = Hive.box<Map>(_imageBox);
    return box.values.toList().cast<Map>();
  }

  Future<void> cacheGalleries(List<Map<String, dynamic>> galleries) async {
    final box = Hive.box<Map>(_galleryBox);
    for (final gallery in galleries) {
      await box.put(gallery['id'], gallery);
    }
  }

  List<Map> getCachedGalleries() {
    final box = Hive.box<Map>(_galleryBox);
    return box.values.toList().cast<Map>();
  }

  Future<void> setCacheExpiry(String key, Duration duration) async {
    final box = Hive.box<Map>(_cacheBox);
    await box.put(key, {
      'expiry': DateTime.now().add(duration).toIso8601String(),
    });
  }

  bool isCacheValid(String key) {
    final box = Hive.box<Map>(_cacheBox);
    final entry = box.get(key);
    if (entry == null) return false;
    return DateTime.parse(entry['expiry']).isAfter(DateTime.now());
  }
}
```

### Step 7: Theme Configuration

`app/lib/core/theme/app_theme.dart`:

```dart
import 'package:flutter/material.dart';

class AppTheme {
  static ThemeData lightTheme() {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.light,
      colorSchemeSeed: const Color(0xFF5C7CFA),
      inputDecorationTheme: const InputDecorationTheme(
        border: OutlineInputBorder(),
        filled: true,
      ),
      cardTheme: CardThemeData(
        elevation: 2,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      ),
    );
  }

  static ThemeData darkTheme() {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.dark,
      colorSchemeSeed: const Color(0xFF5C7CFA),
      inputDecorationTheme: const InputDecorationTheme(
        border: OutlineInputBorder(),
        filled: true,
      ),
      cardTheme: CardThemeData(
        elevation: 2,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
      ),
    );
  }
}
```

`app/lib/core/theme/theme_provider.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';

enum AppThemeMode { light, dark, system }

final themeProvider = StateNotifierProvider<ThemeNotifier, AppThemeMode>((ref) {
  return ThemeNotifier();
});

class ThemeNotifier extends StateNotifier<AppThemeMode> {
  static const String _themeKey = 'app_theme';

  ThemeNotifier() : super(AppThemeMode.system) {
    _loadTheme();
  }

  Future<void> _loadTheme() async {
    final box = await Hive.openBox('settings');
    final saved = box.get(_themeKey);
    if (saved != null) {
      state = AppThemeMode.values.firstWhere(
        (m) => m.name == saved,
        orElse: () => AppThemeMode.system,
      );
    }
  }

  Future<void> setTheme(AppThemeMode mode) async {
    state = mode;
    final box = await Hive.openBox('settings');
    await box.put(_themeKey, mode.name);
  }
}
```

### Step 8: Auth Feature

`app/lib/features/auth/models/user_model.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_model.freezed.dart';
part 'user_model.g.dart';

@freezed
class UserModel with _$UserModel {
  const factory UserModel({
    required String id,
    required String name,
    required String email,
    required String role,
  }) = _UserModel;

  factory UserModel.fromJson(Map<String, dynamic> json) => _$UserModelFromJson(json);
}
```

`app/lib/features/auth/models/auth_response.dart`:

```dart
import 'package:freezed_annotation/freezed_annotation.dart';
import 'user_model.dart';

part 'auth_response.freezed.dart';
part 'auth_response.g.dart';

@freezed
class AuthResponse with _$AuthResponse {
  const factory AuthResponse({
    required UserModel user,
    required String accessToken,
    required String refreshToken,
  }) = _AuthResponse;

  factory AuthResponse.fromJson(Map<String, dynamic> json) => _$AuthResponseFromJson(json);
}
```

`app/lib/features/auth/repositories/auth_repository.dart`:

```dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/network/dio_client.dart';
import '../models/auth_response.dart';

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepository(ref.read(dioProvider));
});

class AuthRepository {
  final Dio _dio;

  AuthRepository(this._dio);

  Future<AuthResponse> login({required String email, required String password}) async {
    final response = await _dio.post('/auth/login', data: {
      'email': email,
      'password': password,
    });
    return AuthResponse.fromJson(response.data);
  }

  Future<AuthResponse> register({
    required String name,
    required String email,
    required String password,
  }) async {
    final response = await _dio.post('/auth/register', data: {
      'name': name,
      'email': email,
      'password': password,
    });
    return AuthResponse.fromJson(response.data);
  }

  Future<void> logout() async {
    await _dio.post('/auth/logout');
  }

  Future<AuthResponse> refresh(String refreshToken) async {
    final response = await _dio.post('/auth/refresh', data: {
      'refreshToken': refreshToken,
    });
    return AuthResponse.fromJson(response.data);
  }
}
```

`app/lib/features/auth/providers/auth_provider.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/storage/auth_storage.dart';
import '../models/user_model.dart';
import '../repositories/auth_repository.dart';

enum AuthStatus { initial, authenticated, unauthenticated, loading }

class AuthState {
  final AuthStatus status;
  final UserModel? user;
  final String? error;

  const AuthState({this.status = AuthStatus.initial, this.user, this.error});

  AuthState copyWith({AuthStatus? status, UserModel? user, String? error}) {
    return AuthState(
      status: status ?? this.status,
      user: user ?? this.user,
      error: error,
    );
  }
}

final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier(
    ref.read(authRepositoryProvider),
    ref.read(authStorageProvider),
  );
});

class AuthNotifier extends StateNotifier<AuthState> {
  final AuthRepository _repository;
  final AuthStorage _storage;

  AuthNotifier(this._repository, this._storage) : super(const AuthState()) {
    checkAuth();
  }

  Future<void> checkAuth() async {
    final isAuth = await _storage.isAuthenticated();
    if (isAuth) {
      // Could fetch user profile here
      state = state.copyWith(status: AuthStatus.authenticated);
    } else {
      state = state.copyWith(status: AuthStatus.unauthenticated);
    }
  }

  Future<void> login({required String email, required String password}) async {
    state = state.copyWith(status: AuthStatus.loading);
    try {
      final response = await _repository.login(email: email, password: password);
      await _storage.saveTokens(
        accessToken: response.accessToken,
        refreshToken: response.refreshToken,
      );
      state = state.copyWith(
        status: AuthStatus.authenticated,
        user: response.user,
      );
    } catch (e) {
      state = state.copyWith(
        status: AuthStatus.unauthenticated,
        error: e.toString(),
      );
    }
  }

  Future<void> register({
    required String name,
    required String email,
    required String password,
  }) async {
    state = state.copyWith(status: AuthStatus.loading);
    try {
      final response = await _repository.register(
        name: name, email: email, password: password,
      );
      await _storage.saveTokens(
        accessToken: response.accessToken,
        refreshToken: response.refreshToken,
      );
      state = state.copyWith(
        status: AuthStatus.authenticated,
        user: response.user,
      );
    } catch (e) {
      state = state.copyWith(
        status: AuthStatus.unauthenticated,
        error: e.toString(),
      );
    }
  }

  Future<void> logout() async {
    try {
      await _repository.logout();
    } finally {
      await _storage.clearTokens();
      state = const AuthState(status: AuthStatus.unauthenticated);
    }
  }
}
```

### Step 9: Generate Screen

`app/lib/features/generate/screens/generate_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/generate_provider.dart';
import '../widgets/provider_selector.dart';

class GenerateScreen extends ConsumerStatefulWidget {
  const GenerateScreen({super.key});

  @override
  ConsumerState<GenerateScreen> createState() => _GenerateScreenState();
}

class _GenerateScreenState extends ConsumerState<GenerateScreen> {
  final _promptController = TextEditingController();
  String _selectedProvider = 'dalle';

  @override
  void dispose() {
    _promptController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final generateState = ref.watch(generateProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Generate Image')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            TextField(
              controller: _promptController,
              decoration: const InputDecoration(
                labelText: 'Prompt',
                hintText: 'Describe the image you want to create...',
              ),
              maxLines: 3,
            ),
            const SizedBox(height: 16),
            ProviderSelector(
              selectedProvider: _selectedProvider,
              onProviderSelected: (provider) {
                setState(() => _selectedProvider = provider);
              },
            ),
            const SizedBox(height: 16),
            FilledButton(
              onPressed: generateState.isLoading
                  ? null
                  : () {
                      ref.read(generateProvider.notifier).generate(
                            prompt: _promptController.text,
                            provider: _selectedProvider,
                          );
                    },
              child: generateState.isLoading
                  ? const SizedBox(
                      width: 20, height: 20,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Text('Generate'),
            ),
            const SizedBox(height: 24),
            if (generateState.hasError)
              Text(generateState.error!, style: const TextStyle(color: Colors.red)),
            if (generateState.hasValue && generateState.value != null)
              Expanded(
                child: ClipRRect(
                  borderRadius: BorderRadius.circular(12),
                  child: Image.network(
                    generateState.value!.url,
                    fit: BoxFit.contain,
                    loadingBuilder: (context, child, loadingProgress) {
                      if (loadingProgress == null) return child;
                      return const Center(child: CircularProgressIndicator());
                    },
                    errorBuilder: (context, error, stackTrace) {
                      return const Center(child: Text('Failed to load image'));
                    },
                  ),
                ),
              ),
          ],
        ),
      ),
    );
  }
}
```

### Step 10: Gallery Screen

`app/lib/features/gallery/screens/gallery_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/gallery_provider.dart';
import '../widgets/image_card.dart';

class GalleryScreen extends ConsumerWidget {
  const GalleryScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final imagesAsync = ref.watch(galleryProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Gallery')),
      body: imagesAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, _) => Center(child: Text('Error: $error')),
        data: (images) {
          if (images.isEmpty) {
            return const Center(child: Text('No images yet. Generate your first image!'));
          }
          return RefreshIndicator(
            onRefresh: () => ref.refresh(galleryProvider.future),
            child: GridView.builder(
              padding: const EdgeInsets.all(8),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 2,
                mainAxisSpacing: 8,
                crossAxisSpacing: 8,
              ),
              itemCount: images.length,
              itemBuilder: (context, index) {
                return ImageCard(image: images[index]);
              },
            ),
          );
        },
      ),
    );
  }
}
```

### Step 11: Settings Screen with Theme Toggle

`app/lib/features/settings/screens/settings_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/theme/theme_provider.dart';
import '../../auth/providers/auth_provider.dart';

class SettingsScreen extends ConsumerWidget {
  const SettingsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeMode = ref.watch(themeProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Settings')),
      body: ListView(
        children: [
          ListTile(
            leading: const Icon(Icons.palette),
            title: const Text('Theme'),
            subtitle: Text(themeMode.name.toUpperCase()),
            trailing: DropdownButton<AppThemeMode>(
              value: themeMode,
              underline: const SizedBox.shrink(),
              items: const [
                DropdownMenuItem(value: AppThemeMode.light, child: Text('Light')),
                DropdownMenuItem(value: AppThemeMode.dark, child: Text('Dark')),
                DropdownMenuItem(value: AppThemeMode.system, child: Text('System')),
              ],
              onChanged: (mode) {
                if (mode != null) ref.read(themeProvider.notifier).setTheme(mode);
              },
            ),
          ),
          const Divider(),
          ListTile(
            leading: const Icon(Icons.logout),
            title: const Text('Sign Out'),
            onTap: () {
              ref.read(authProvider.notifier).logout();
            },
          ),
        ],
      ),
    );
  }
}
```

### Step 12: App Entry Point and Routing

`app/lib/core/router/app_router.dart`:

```dart
import 'package:go_router/go_router.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../features/auth/providers/auth_provider.dart';
import '../../features/auth/screens/login_screen.dart';
import '../../features/auth/screens/signup_screen.dart';
import '../../features/generate/screens/generate_screen.dart';
import '../../features/gallery/screens/gallery_screen.dart';
import '../../features/settings/screens/settings_screen.dart';

final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    redirect: (context, state) {
      final isAuthenticated = authState.status == AuthStatus.authenticated;
      final isAuthRoute = state.matchedLocation == '/login' || state.matchedLocation == '/signup';

      if (!isAuthenticated && !isAuthRoute) return '/login';
      if (isAuthenticated && isAuthRoute) return '/generate';
      return null;
    },
    routes: [
      GoRoute(path: '/login', builder: (context, state) => const LoginScreen()),
      GoRoute(path: '/signup', builder: (context, state) => const SignupScreen()),
      GoRoute(path: '/generate', builder: (context, state) => const GenerateScreen()),
      GoRoute(path: '/gallery', builder: (context, state) => const GalleryScreen()),
      GoRoute(path: '/settings', builder: (context, state) => const SettingsScreen()),
    ],
    initialLocation: '/generate',
  );
});
```

`app/lib/app.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/theme/app_theme.dart';
import 'core/theme/theme_provider.dart';
import 'core/router/app_router.dart';

class FastDepoApp extends ConsumerWidget {
  const FastDepoApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final themeMode = ref.watch(themeProvider);
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'FastDepo',
      theme: AppTheme.lightTheme(),
      darkTheme: AppTheme.darkTheme(),
      themeMode: _mapThemeMode(themeMode),
      routerConfig: router,
      debugShowCheckedModeBanner: false,
    );
  }

  ThemeMode _mapThemeMode(AppThemeMode mode) {
    switch (mode) {
      case AppThemeMode.light: return ThemeMode.light;
      case AppThemeMode.dark: return ThemeMode.dark;
      case AppThemeMode.system: return ThemeMode.system;
    }
  }
}
```

`app/lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'app.dart';
import 'core/storage/hive_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Hive.initFlutter();
  await HiveService().init();
  runApp(const ProviderScope(child: FastDepoApp()));
}
```

### Step 13: Android Signing Configuration

`app/android/key.properties.example`:

```properties
storePassword=YOUR_KEYSTORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=upload
storeFile=/path/to/your/upload-keystore.jks
```

`app/android/app/build.gradle` — add signing config:

```groovy
// Add at the top of android/app/build.gradle, after other def statements:
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    // ... existing config ...

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
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

Generate the upload keystore:

```bash
cd app/android
keytool -genkey -v -keystore upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias upload

# Move to a secure location (NOT in version control)
mkdir -p ~/.keystores
mv upload-keystore.jks ~/.keystores/

# Update key.properties with actual paths
# storeFile=/Users/you/.keystores/upload-keystore.jks
```

### Step 14: Build and Release

```bash
# Generate freezed and JSON serialization code
cd app
dart run build_runner build --delete-conflicting-outputs

# Build release APK
flutter build apk --release

# Build app bundle (for Play Store)
flutter build appbundle --release

# Verify the build
flutter install --release

# Run on connected device
flutter run --release
```

### Step 15: Play Store Submission Steps

```markdown
## Play Store Submission Checklist

1. **Prepare Store Listing**
   - App name: FastDepo - AI Image Generator
   - Short description (80 chars): Generate stunning images with AI
   - Full description (4000 chars): Detailed feature description
   - App icon: 512x512 PNG
   - Feature graphic: 1024x500 PNG
   - Screenshots: At least 2 per form factor (phone, 7" tablet, 10" tablet)

2. **Content Rating**
   - Complete the IARC questionnaire
   - Expected rating: PEGI 3 / Everyone

3. **Privacy Policy**
   - Required URL: https://yourdomain.com/privacy
   - Must cover: data collection, image storage, analytics

4. **App Bundle**
   - Upload `build/app/outputs/bundle/release/app-release.aab`
   - Enable Google Play App Signing

5. **Internal Testing Track**
   - Create a closed testing track
   - Add 20+ testers
   - Test for at least 14 days

6. **Production Rollout**
   - Start with 10% rollout
   - Monitor crash rate < 1%
   - Gradually increase to 100%
```

## Verification

```bash
# 1. Run code generation
cd app && dart run build_runner build --delete-conflicting-outputs

# 2. Run static analysis
flutter analyze

# 3. Run unit tests
flutter test

# 4. Build debug APK
flutter build apk --debug

# 5. Build release APK
flutter build apk --release

# 6. Build app bundle
flutter build appbundle --release

# 7. Verify on device
flutter run --release

# 8. Check app size
flutter build apk --analyze-size

# Expected: APK < 30MB, AAB < 25MB
```

## Rollback

```bash
# Remove the entire Flutter project
rm -rf app/

# Remove specific feature if needed
rm -rf app/lib/features/gallery/
# Then remove the route from app_router.dart and pubspec.yaml dependencies
```

## ADR-024: Flutter Mobile Architecture

**Decision**: Use Flutter with Riverpod for state management, Dio for networking, Hive for offline cache, and GoRouter for navigation in a feature-first architecture.

**Reason**: Riverpod is compile-safe and more testable than Provider. Dio provides robust interceptor support for auth token management. Hive offers fast local storage without native dependencies. GoRouter supports deep linking and declarative routing. Feature-first architecture keeps related code together as the app grows.

**Consequences**:
- Requires `build_runner` for freezed/json_serializable code generation
- Two state management approaches: Riverpod for app state, Hive for persistence
- Flutter Secure Storage uses encrypted shared prefs on Android (requires API 23+)
- Feature-first architecture has some boilerplate but scales well
- GoRouter is still evolving API-wise but is the Flutter team's recommended router

**Alternatives Considered**:
- BLoC pattern: More boilerplate, steeper learning curve, overkill for this app size
- GetX: Popular but has anti-pattern concerns and God-object tendency
- Provider: Simpler but less type-safe and harder to test than Riverpod
- sqflite: More powerful local DB but adds complexity Hive avoids for simple caching
- Navigator 2.0: Lower-level, more error-prone than GoRouter