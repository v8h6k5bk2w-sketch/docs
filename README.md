# Privata Docs

Источник правды для `docs.privataswap.com`. Mintlify, docs-as-code.

## Структура

```
privata-docs/
├── docs.json                    # конфиг Mintlify (theme, nav, footer)
├── index.mdx                    # главная
├── api-reference/
│   └── openapi.yaml             # OpenAPI 3.1 → автогенерация Reference + playground
├── guides/
│   ├── quickstart.mdx
│   ├── authentication.mdx
│   ├── order-lifecycle.mdx
│   ├── refund-flow.mdx
│   ├── sse-streaming.mdx
│   ├── webhooks.mdx
│   ├── sandbox.mdx
│   ├── rate-limits.mdx
│   ├── errors.mdx
│   ├── sla.mdx
│   ├── ops-events.mdx
│   └── changelog.mdx
├── sdk/
│   ├── js/{install,quickstart,configuration,errors,streaming}.mdx
│   └── compat/{changenow,trocador}.mdx
└── images/{logo-light.svg, logo-dark.svg, favicon.svg}
```

## Локальный preview

```bash
npm i -g mint
cd privata-docs
mint dev          # http://localhost:3000
```

## Деплой через Mintlify

1. Залить `privata-docs/` в отдельный GitHub-репо `privataswap/docs` (private OK).
2. На [mintlify.com/start](https://mintlify.com/start) подключить репо. Mintlify ставит GitHub-App, дальше каждый `git push` в `main` → автодеплой за 30 с.
3. По умолчанию сайт на `https://privata.mintlify.app`. Кастомный домен:
   - В dashboard.mintlify.com → Settings → Custom domain → `docs.privataswap.com`.
   - На Cloudflare добавить CNAME `docs` → `cname.mintlify.app` (proxied off).
   - SSL Mintlify выписывает сам, 5-10 минут.

## Обновление OpenAPI

Когда меняется `privata_wallet_api_spec.md` в основном репо:
1. Регенерировать `api-reference/openapi.yaml`.
2. PR в `privataswap/docs`.
3. Mintlify деплоит, playground обновляется автоматически.

## Что вынести из репо v3

- Логотипы из `privata-v3/client/public/` если есть финальные SVG.
- Цвета `colors.primary/light/dark` уже синхронизированы с brand-kit.
