# Secrets

Read this when a spec needs credentials — never hardcode them in specs.

## Reading secrets in interpolation

```drun
run "deploy --token {secret('api_key')}"
get "{$globals.api_base}/me" with auth bearer "{secret('api_token')}"
info "{secret('webhook_url', 'https://default.example')}"   # with default
```

Secrets resolve through the drun secret store (project, global, or custom
namespaces), never through the spec file itself.

## Managing secrets from tasks

```drun
secret set "api_key" to "my_secret_key_123"
secret set "api_key" to "project_a_key" in namespace "project-a"
secret exists "api_key"
secret list
secret list from namespace "project-a"
secret delete "api_key"
secret delete "api_key" from namespace "project-a"
```

Namespaces isolate secrets per project/environment; reads with
`{secret('name')}` resolve against the active namespace chain.

## Tips

- Pair `secret exists "x"` with a clear `fail "..."` message for friendlier
  preflight errors than a mid-run undefined-secret failure.
- Wrap first-run secret use in `try:`/`catch as $e:` to point the user at the
  setup task (see `error-handling.md`).

Upstream examples: `examples/57-secrets-basic.drun`,
`examples/58-secrets-namespaces.drun`, `examples/60-secrets-interpolation.drun`,
`examples/71-cli-secret-management.drun`.
