---
paths:
  - "**/data-sources/**"
---

# React: Data Sources

A Data Source is a plain class that wraps one SDK or browser API, behind an interface. `src/wiring/` creates one instance per container, so it lives as long as the app.

## Layout

```text
src/data-sources/
  http/
    http-client.data-source.int.ts
    axios-http-client.data-source.ts
  auth/
    auth-provider.data-source.int.ts
    firebase-auth.data-source.ts
  token-storage/
    token-storage.data-source.int.ts
    local-token-storage.data-source.ts
```

Put the interface in a sibling `*.data-source.int.ts` file. Repositories and tests depend on the interface only.

## Shape

```ts
// http-client.data-source.int.ts
export interface HttpClient {
  get(path: string): Promise<unknown>;
  post(path: string, body: unknown): Promise<unknown>;
}
```

```ts
// axios-http-client.data-source.ts
import axios, { type AxiosInstance, isAxiosError } from "axios";
import { ApiError, NetworkError } from "@/contracts/errors/app-errors";
import type { SignOutSignal } from "@/contracts/signals/sign-out.signal";
import type { HttpClient } from "./http-client.data-source.int";
import type { TokenStorage } from "../token-storage/token-storage.data-source.int";

export class AxiosHttpClient implements HttpClient {
  private readonly http: AxiosInstance;

  constructor(
    baseURL: string,
    private readonly tokens: TokenStorage,
    private readonly signOut: SignOutSignal,
  ) {
    this.http = axios.create({ baseURL, timeout: 15_000 });
    this.http.interceptors.request.use(async (config) => {
      const token = await this.tokens.read();
      if (token) config.headers.Authorization = `Bearer ${token}`;
      return config;
    });
  }

  get(path: string): Promise<unknown> {
    return this.send(() => this.http.get(path));
  }

  post(path: string, body: unknown): Promise<unknown> {
    return this.send(() => this.http.post(path, body));
  }

  private async send(call: () => Promise<{ data: unknown }>): Promise<unknown> {
    try {
      return (await call()).data;
    } catch (error) {
      if (isAxiosError(error) && error.response) {
        if (error.response.status === 401) this.signOut.request("session-expired");
        throw new ApiError(error.response.status, error.message);
      }
      throw new NetworkError("[AxiosHttpClient] Request failed");
    }
  }
}
```

## Rules

- Import SDK packages (`axios`, `firebase/*`, `idb`) here and nowhere above this layer.
- Convert SDK types to Contracts before returning them. Never return an `AxiosResponse`, a Firebase `User`, or a `UserCredential`.
- Catch SDK errors and throw a class from `src/contracts/errors/`. SDK error codes, such as `auth/popup-closed-by-user`, never leave this layer; map them to a Contract error or a Contract result.
- Read tokens through another Data Source, such as token storage. Never import from `src/use-cases/`.
- When a failure must trigger higher-level work, raise a signal from `src/contracts/signals/`, as the 401 branch does above. See `stack/wiring.md`.
- Hold connections, SDK instances and handles in the instance, never in module-level variables, so tests can create a fresh one.
- Touch browser-only APIs (`window`, `localStorage`, `indexedDB`, Firebase Auth) inside methods, never in constructors. Next.js also evaluates client modules on the server during pre-rendering.
