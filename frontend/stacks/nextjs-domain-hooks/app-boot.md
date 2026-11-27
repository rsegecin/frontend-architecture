---
paths:
  - "**/app/**/layout.tsx"
  - "**/providers/**"
  - "**/proxy.ts"
  - "**/i18n/**"
---

# Next.js Domain Hooks: App Boot, Routing and Session Gate

This stack has no boot Use Case. The locale layout mounts the providers, and the `(private)` layout's `AuthProvider` gates every signed-in page.

## Tree

```text
src/proxy.ts                                # next-intl middleware: locale prefix and redirect
src/app/(frontend)/[locale]/layout.tsx      # Server: validates the locale, fonts, <html>, <body>
  NextIntlClientProvider
    ThemeProvider                           # next-themes, writes data-theme on <html>
      QueryProvider                         # getQueryClient() and the devtools
        [locale]/(private)/layout.tsx
          AuthProvider                      # session gate
```

## Proxy and locales

- `src/proxy.ts` runs the next-intl middleware. A path without a locale redirects to the locale found in the cookie or `Accept-Language`.
- The matcher skips `/api`, `/trpc`, `/_next`, `/_vercel` and any path with a file extension.
- Never protect routes in the proxy. The session lives in `localStorage`, which the proxy cannot see.
- Locales and the default live in `src/i18n/routing.ts`. An app that needs no locale in the URL sets `localePrefix: "never"`, and the `[locale]` folder stays.
- `src/i18n/request.ts` loads `messages.json` and `errors.json` for the locale and merges them. A top-level key of `errors.json` must not repeat one from `messages.json`, or it overwrites it.

## Locale layout

```tsx
export default async function RootLayout({
  children,
  params,
}: Readonly<{ children: React.ReactNode; params: Promise<{ locale: string }> }>) {
  const { locale } = await params;
  if (!hasLocale(routing.locales, locale)) notFound();

  return (
    <html lang={locale} className={`${inter.variable} ${mono.variable}`} suppressHydrationWarning>
      <body className="antialiased">
        <NextIntlClientProvider>
          <ThemeProvider>
            <QueryProvider>{children}</QueryProvider>
          </ThemeProvider>
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

A new app-wide provider mounts here. Font variables go on `<html>`, because Tailwind reads `--font-sans` at the root.

## Session gate

`src/providers/auth-provider/` follows the component folder pattern. `index.tsx` renders a loading status, an error with retry and sign-out buttons, or `children` once the user is ready. The logic lives in `use-auth-provider.ts`:

```ts
export function useAuthProvider() {
  const router = useRouter();
  const apiErrorMessage = useApiErrorMessage();
  // The whole store, not a selector: clear() must re-render even when user is already null.
  const { user, setUser } = useAuthStore();
  const hasSession = getSession() !== null;
  const currentUser = UserQueries.useGetCurrentUser({ enabled: hasSession });

  useEffect(() => {
    if (!hasSession) router.replace("/login");
  }, [hasSession, router]);

  useEffect(() => {
    if (currentUser.data) setUser(currentUser.data);
  }, [currentUser.data, setUser]);

  return {
    isReady: hasSession && user !== null,
    isError: currentUser.isError,
    errorMessage: apiErrorMessage(currentUser.error),
    isRetrying: currentUser.isFetching,
    retry: () => currentUser.refetch(),
    logout,
  };
}
```

- Inside `(private)`, the signed-in user comes from `useAuthStore((state) => state.user)`.
- A failed user load shows translated copy and lets the user retry or sign out. It never lets the app through.

## Sign-in and sign-out

| Event | What happens |
|---|---|
| Sign-in | A session-opening mutation runs `onSuccess: saveSession`, then the login ViewModel calls `router.replace` to the app's home page |
| Opening the login page with a session | The login ViewModel redirects to the home page in a mount effect |
| Session expired | `httpClient` refreshes the token, or calls `logout()`. See `stack/data-sources.md` |
| Sign-out | Always `logout()`. `AuthProvider` then redirects to `/login` |

## Theme

`ThemeProvider` wraps next-themes with `themes={["light", "dark"]}` and `defaultTheme="light"`. Pages without a dark theme, such as the login page, are listed in its light-only paths and get `forcedTheme="light"`. Read or change the theme with `useTheme()`.
