# DeepSeek Harness integration

DeepSeek Harness (DSH) can use Copilot2API as an OpenAI Responses provider while [`dsh-web-search-provider`](https://github.com/cloga/dsh-web-search-provider) enables the provider's hosted search capability.

## Verified topology

```text
DSH
  -> dsh-web-search-provider
  -> Copilot2API POST /v1/responses
  -> hosted web_search

DSH model catalog
  -> Copilot2API GET /v1/models
  -> complete upstream model metadata

DSH image input
  -> official DSH vision channel
  -> configured image-capable model route
```

The search provider adds the native Responses `web_search` tool and can request search-source metadata. Copilot2API forwards the request through `/v1/responses`; the search runs as a hosted model-provider tool rather than as a local DSH function tool.

DSH's official vision channel remains responsible for image input. Image-capable models receive images through that channel; the web-search provider does not replace or emulate it.

Copilot2API returns the raw upstream `/v1/models` response from its cache, including model capabilities and other metadata. DSH and its provider integration decide which models appear in a picker and which route is active.

## Configuration

Keep Copilot2API bound to a loopback address. Do not expose it publicly: it does not validate client API keys and would otherwise become an open proxy that consumes the authenticated Copilot account's quota.

Configure the DSH Responses route with values equivalent to:

```text
base URL: http://127.0.0.1:7777/v1
API key: <local-placeholder>
model: <model-id-from-v1-models>
```

Then configure the `web-search-provider` settings namespace:

```yaml
enabled: true
baseURL: http://127.0.0.1:7777/v1
model: <model-id-from-v1-models>
apiKeyEnv: <dsh-credential-reference>
includeSources: true
stripServerTools: true
probe: true
```

Use DSH's credential store or an environment-backed credential reference for `apiKeyEnv`. Never put a GitHub token, Copilot credential, or machine-specific path in DSH settings or committed files.

Confirm that Copilot2API exposes the selected model before enabling the route:

```bash
curl http://127.0.0.1:7777/v1/models
```

## Responsibility boundary

This integration uses existing Copilot2API behavior; it does not require Copilot2API feature or routing changes.

DSH and provider-side code own:

- model-picker filtering and active-route selection;
- replay serialization into DSH session history;
- sandbox and temporary-resource cleanup;
- synchronization between saved settings and the live provider configuration.

Copilot2API owns Copilot authentication, the OpenAI-compatible `/v1/responses` transport, hosted `web_search` forwarding, and complete `/v1/models` metadata delivery. Issues in the DSH-owned lifecycle should be fixed in DSH or the provider rather than worked around in Copilot2API.
