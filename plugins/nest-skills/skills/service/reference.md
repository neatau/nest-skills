# Service examples

Two worked examples using an `ArticleService` in the `news` module. Adapt names
to the actual module/service.

---

## Example 1 — Plain service (no FactoryProvider)

Standard DI only: dependencies Nest can resolve on its own. No provider file.

`news/services/article/article.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import { ArticleRepository } from '../../repositories/article.repository';

@Injectable()
export class ArticleService {
  constructor(private readonly repository: ArticleRepository) {}

  async findBySlug(slug: string) {
    return this.repository.findOneBySlug(slug);
  }
}
```

Register the class directly in the module:

```ts
import { Module } from '@nestjs/common';
import { ArticleService } from './services/article/article.service';

@Module({
  providers: [ArticleService],
  exports: [ArticleService],
})
export class NewsModule {}
```

---

## Example 2 — Service with a FactoryProvider

Here the service needs application config (an API URL/key) and a constructed
axios client. Both are non-standard DI, so a FactoryProvider is warranted. The
service receives a narrow `ArticleServiceOptions` — it never sees `ConfigService`.

`news/services/article/article.service.types.ts`

```ts
export interface ArticleServiceOptions {
  readonly apiBaseUrl: string;
  readonly apiKey: string;
  readonly timeoutMs: number;
}
```

`news/services/article/article.service.ts`

```ts
import { Injectable } from '@nestjs/common';
import type { AxiosInstance } from 'axios';
import { ArticleRepository } from '../../repositories/article.repository';
import { buildArticlePath } from './article.service.support';
import { ArticleServiceOptions } from './article.service.types';

@Injectable()
export class ArticleService {
  constructor(
    private readonly options: ArticleServiceOptions,
    private readonly http: AxiosInstance,
    private readonly repository: ArticleRepository,
  ) {}

  async fetchRemote(slug: string) {
    const { data } = await this.http.get(buildArticlePath(slug));
    return data;
  }
}
```

`news/services/article/article.service.provider.ts`

```ts
import { Provider } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios from 'axios';
import { ArticleRepository } from '../../repositories/article.repository';
import { ArticleService } from './article.service';
import { ArticleServiceOptions } from './article.service.types';

export const ArticleServiceProvider: Provider = {
  provide: ArticleService,
  inject: [ConfigService, ArticleRepository],
  useFactory: async (config: ConfigService, repository: ArticleRepository) => {
    // ConfigService is read ONLY here, at the provider boundary — never inside the service.
    const options: ArticleServiceOptions = {
      apiBaseUrl: config.getOrThrow<string>('ARTICLE_API_URL'),
      apiKey: config.getOrThrow<string>('ARTICLE_API_KEY'),
      timeoutMs: config.get<number>('ARTICLE_API_TIMEOUT_MS') ?? 5000,
    };

    const http = axios.create({
      baseURL: options.apiBaseUrl,
      timeout: options.timeoutMs,
      headers: { Authorization: `Bearer ${options.apiKey}` },
    });

    return new ArticleService(options, http, repository);
  },
};
```

`news/services/article/article.service.support.ts`

```ts
// Closely related helper that doesn't belong as a method on the service.
export function buildArticlePath(slug: string): string {
  return `/articles/${encodeURIComponent(slug)}`;
}
```

Register the **provider** (not the class) in the module. The `provide` token is
the class, so consumers still inject `ArticleService` normally and other modules
can import it via `exports`:

```ts
import { Module } from '@nestjs/common';
import { ArticleService } from './services/article/article.service';
import { ArticleServiceProvider } from './services/article/article.service.provider';

@Module({
  providers: [ArticleServiceProvider],
  exports: [ArticleService],
})
export class NewsModule {}
```

### Why this satisfies the rules

- The service constructor takes a narrow `ArticleServiceOptions`, not
  `ConfigService` — config access is confined to the provider.
- The axios instance is built in the factory and passed in, keeping construction
  logic out of the service.
- `useFactory` is `async`, so any dependency needing async setup fits the same
  shape (`await` inside the factory before `new`).
- No `@Inject()` in the constructor — token wiring lives in the provider's
  `inject` array.
- All dependencies are `private readonly`.
