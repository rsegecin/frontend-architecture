---
paths:
  - "**/config/**"
  - "**/lib/session.ts"
---

# Next.js Domain Hooks: Data Sources and Global Instances

Three modules wrap the external systems. Each exports a configured instance or plain functions, with no interface and no class.

| Module | Exports | Wraps |
|---|---|---|
| `src/config/axios-client.ts` | `httpClient` | The REST API, through axios |
| `src/config/query-client.ts` | `getQueryClient()` | The TanStack Query cache |
| `src/lib/session.ts` | `saveSession`, `getSession`, `logout` | Session tokens in `localStorage` |

App-wide error handling lives here once, never in each hook: the session-expired 401 in `httpClient`, and the retry policy in the `QueryClient`.

## httpClient

- The base URL comes from `NEXT_PUBLIC_API_URL`.
- Set `paramsSerializer` to the array format the API reads. `{ indexes: null }` sends `?ids=a&ids=b`.
- The request interceptor attaches `Authorization: Bearer <accessToken>` from `getSession()`.
- The response interceptor renews the session on a 401, once for the whole app:

```ts
type RetriableConfig = InternalAxiosRequestConfig & { retried?: boolean };

// One refresh at a time: the API rotates the refresh token, so a second call with the old one would fail.
let refreshing: Promise<void> | null = null;

httpClient.interceptors.response.use(undefined, async (error: unknown) => {
  const config: RetriableConfig | undefined =
    error instanceof AxiosError ? error.config : undefined;
  const session = getSession();
  // Without a token, a 401 is a wrong password or code, and the screen shows it.
  if (
    !(error instanceof AxiosError) ||
    error.response?.status !== 401 ||
    !config?.headers.has("Authorization") ||
    config.url === "/auth/refresh" ||
    !session
  ) {
    return Promise.reject(error);
  }

  if (config.retried) {
    logout();
    return Promise.reject(error);
  }

  try {
    refreshing ??= Service.auth
      .refreshSession({ refreshToken: session.refreshToken })
      .then(saveSession)
      .finally(() => {
        refreshing = null;
      });
    await refreshing;
  } catch {
    logout();
    return Promise.reject(error);
  }

  config.retried = true;
  return httpClient(config);
});
```

- Each request retries at most once. A second 401, or a failed refresh, logs out.
- `Service` here is the auth module's registry, imported from `@/domain/auth/http`. Calling `Service.auth.refreshSession` from here is the stack's only lower-to-higher call. Do not add another without asking the user.

## QueryClient

```ts
function makeQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
        // A 4xx gives the same answer on retry.
        retry: (failureCount, error) => {
          const status = error instanceof AxiosError ? error.response?.status : undefined;
          const isClientError = status !== undefined && status >= 400 && status < 500;
          return !isClientError && failureCount < 3;
        },
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

// Per request on the server, so users never share a cache there.
export function getQueryClient(): QueryClient {
  if (isServer) return makeQueryClient();
  browserQueryClient ??= makeQueryClient();
  return browserQueryClient;
}
```

Never create another `QueryClient`. Outside React, use `getQueryClient()`; inside a hook, `useQueryClient()`.

## Session

- Tokens live in `localStorage`, not in a cookie, and only `src/lib/session.ts` reads or writes them.
- `getSession()` returns `null` on the server, where `localStorage` does not exist.
- Signing out is always `logout()`. It removes the tokens, calls `getQueryClient().clear()` so the next user never sees cached data, and clears `useAuthStore`.
- `logout()` does not navigate. `AuthProvider` sees the missing session and sends the user to the login page.

## Errors

- `AxiosError` reaches the ViewModel unchanged. Only `useApiErrorMessage` and the `QueryClient` retry policy inspect it.
- `useApiErrorMessage()` in `src/lib/hooks/` reads `error.response.data.error.code`, shaped as `ApiError` in Contracts, and returns the translation under `apiErrors.<code>`, or `apiErrors.default`.

```ts
export function useApiErrorMessage(): (error: unknown) => string {
  const t = useTranslations("apiErrors");

  return (error) => {
    const data: Partial<ApiError> | undefined =
      error instanceof AxiosError ? error.response?.data : undefined;
    const code = data?.error?.code;
    return code && t.has(code) ? t(code) : t("default");
  };
}
```
