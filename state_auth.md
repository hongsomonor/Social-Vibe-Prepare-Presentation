# SocialVibe – Auth & State Management Cheat Sheet (2026-03-05)

## Stack & Conventions
- Flutter with GetX (`GetMaterialApp`, `GetPage`, `GetxController`, `Obx`).
- HTTP via custom `ApiClient` wrapping `http` + GetX `Response`.
- Firebase initialized in `lib/main.dart` (currently only core is used).
- No persistent storage for auth token yet; token lives in memory inside `ApiClient`.

## Auth Flow (email/password + Google)
- Screens: welcome → login (`lib/features/auth/views/login_screen.dart`) → register → OTP → main shell.
- Controllers are registered in `AuthBinding` and injected per route.
  - `LoginController.submit()` calls `AuthService.login().execute()`, sets bearer token on success, navigates `Get.offAllNamed(AppRoutes.main)`.
  - `RegisterController.submit()` posts to `/api/auth/register`, then routes to OTP with the email in `Get.arguments`.
  - `OtpController.verify()` posts `/api/auth/verify-otp`, then forces a fresh login screen.
- Google: `GoogleSignService.signInAndGetAccessToken()` obtains OAuth token; `AuthService.loginWithGoogleToken()` posts it to `/api/auth/google/token`; on 200 response the UI navigates to `AppRoutes.home` (note: token is not stored—likely a gap to fix).
- Logout/Delete: `AuthService.logout()` and `ProfileController.logout()` call `/api/auth/logout` (or delete), clear token via `ApiClient.setAuthToken(null)`, then `Get.offAllNamed(AppRoutes.login)`.

## Endpoints & Token Handling
- Base URL: `https://social-api.hushstackcambodia.site` (`lib/core/constants/api_endpoints.dart`).
- Key paths: register, login, verify-otp, resend-otp, logout, google/token, profile, posts, notifications.
- `ApiClient` builds JSON requests with `Authorization: Bearer <token>` when set; throws `ApiException` on non‑2xx with parsed message/errors.
- Token persistence: in‑memory only; app restart will lose the session.

## State Management Pattern (GetX)
- Controllers hold reactive state using `.obs`, `Rxn`, `RxList`, exposed to the UI with `Obx` widgets.
- Dependency injection via `Get.put`/`Get.lazyPut` inside `Bindings` (see `lib/routes/app_pages.dart` and `lib/features/auth/bindings/auth_binding.dart`).
- Navigation/state coupling: routes pass arguments (e.g., email to OTP, updated profile back to main tab) and may refresh data when a tab opens (`MainShellController.changeTab` triggers profile fetch on index 3).
- Examples:
  - `LoginController`: form validation, `isLoading` spinner, inline field error strings (`RxnString`).
  - `HomeController`: `posts` as `RxList<HomePost>`, optimistic `toggleLike` with revert on failure.
  - `NotificationController`: `isLoading`, `errorMessage`, `notifications` list, refreshed on pull.
  - `ProfileController`: `profile` as `Rxn`, fetch on init, logout clears token and route stack.
  - `ProfileEditController`: combines form state + picked images (`Rxn<File>`) and navigates back with updated model.

## Route Map (auth-related)
- `/` welcome (no binding).
- `/login`, `/register`, `/otp` → `AuthBinding` provides `LoginController`, `RegisterController`, `OtpController`.
- `/main` → `MainShellController` + `ProfileController` (for profile tab).
- `/profile` → `ProfileController`; `/profile-edit` → `ProfileEditController`.

## What to emphasize if asked
- “We use GetX for both navigation and reactive state. Each screen has a controller injected by a route binding; UI listens with `Obx`.”
- “Auth tokens are stored in-memory inside `ApiClient`; clearing the token (logout/delete) immediately removes the bearer header.”
- “Registration requires OTP: email is passed through `Get.arguments`; OTP verify then redirects to login.”
- “Google sign-in obtains an access token and posts it to our backend; we should extend it to save the returned auth token like the email/password path.”
- “Optimistic updates (likes) and tab-driven refresh (profile tab) are implemented through GetX reactive lists.”

## Quick File Pointers
- `lib/main.dart` – app bootstrap, Firebase init, GetMaterialApp.
- `lib/routes/app_pages.dart` / `app_routes.dart` – route + binding wiring.
- `lib/features/auth/services/auth_service.dart` – all auth HTTP calls + OTP helpers.
- `lib/features/auth/controllers/*.dart` – login/register/otp logic & form state.
- `lib/features/auth/services/google_sigin_service.dart` – Google OAuth token fetch.
- `lib/core/network/api_client.dart` – shared HTTP client & token storage.
- `lib/features/*/controllers/*.dart` – GetX state per feature (home, profile, notifications, main shell).
