---
"r2-explorer": minor
---

Add support for Cloudflare Workers Secrets Store bindings. The `AppEnv` type now includes `SecretsStoreSecret` alongside `R2Bucket` and `Fetcher`, so workers with `[[secrets_store_secrets]]` bindings in `wrangler.toml` are fully typed. Example configuration added to the template `wrangler.toml`.
