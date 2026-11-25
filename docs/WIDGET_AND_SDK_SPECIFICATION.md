# Widget and SDK Specification

**Status:** Normative MVP embed contract.

## 1. Goals

Provide a small embeddable chat client for customer websites while keeping customer secrets and privileged authorization on customer servers. The widget is isolated from the host page, supports guest sessions and authenticated identity handoff, and uses a versioned message protocol.

## 2. Embedding model

MVP uses a cross-origin iframe served from the SaaS widget origin. A small loader script creates the iframe and transfers only public configuration and a short-lived session token. Do not inject application code into the host DOM beyond the loader/container. Do not expose workspace API keys, provider keys or connector credentials to the browser.

Illustrative embed:

```html
<div id="support-agent"></div>
<script
  src="https://widget.example.com/loader.js"
  data-public-key="pub_workspace_key"
  data-container="#support-agent"
  async
></script>
```

The public key identifies publishable widget configuration only. It is not a secret and cannot authorize workspace administration or direct privileged API calls.

## 3. Session bootstrap

Guest flow:
1. Loader requests a widget session using public key, origin and anti-abuse signals.
2. Server validates configured allowed origins, rate limits and widget status.
3. Server returns a short-lived, narrowly scoped session token.
4. Iframe uses token for chat session and streaming.
5. Token is bound to workspace, agent, origin, expiry and allowed operations.

Authenticated flow:
1. Customer page asks its own backend for a signed identity assertion.
2. Backend calls the SaaS identity exchange endpoint using its server credential.
3. SaaS returns a short-lived user session token.
4. Customer passes token to loader through a supported initialization method or secure postMessage handshake.
5. SaaS validates actor and claims; customer remains authorization authority.

Never accept a raw end-user ID from JavaScript as proof of identity.

## 4. postMessage protocol

Messages must include protocol version, message type, request ID and payload. Validate exact origin and iframe window source on every message. Use a strict allow-list of message types; reject unknown privileged operations.

Host → widget examples:
- `widget.init`
- `session.update`
- `theme.update`
- `chat.open`
- `chat.close`
- `chat.send` (optional host-controlled use case)

Widget → host examples:
- `widget.ready`
- `chat.opened`
- `chat.closed`
- `message.sent`
- `message.received`
- `action.requested` (informational only)
- `widget.error`

Do not expose arbitrary method invocation, DOM access or privileged tool execution over this bridge. Validate schema and payload size. Use an explicit target origin, never `*` for sensitive messages.

## 5. Configuration

Public configuration may include:
- Theme colors and light/dark mode
- Launcher position, label and welcome text
- Locale and supported language list
- Display name and logo asset reference
- Allowed domains/origins
- Public feature flags and safe UI limits

Private configuration includes provider credentials, integration tokens, system prompts, internal tool policy and workspace secrets; it is never returned to widget config.

## 6. Accessibility and UX

- Keyboard accessible launcher, dialog, message composer and controls.
- Visible focus indicators and semantic labels.
- Screen-reader announcements for new messages and errors.
- Respect reduced motion and color contrast.
- Responsive iframe sizing and mobile safe areas.
- Clear loading, reconnecting, rate-limit, failure and empty states.
- Distinguish assistant text, citations, tool activity and user messages.
- Never display hidden reasoning or raw tool payloads.

## 7. Uploads and attachments

Uploads use short-lived, scoped upload grants. Enforce content-type allow-list, actual file signature inspection, size/count limits, malware scanning where available and private object storage. Do not fetch arbitrary attachment URLs from the browser. Files are linked to the session and workspace; authorization is checked on every read.

## 8. SDK surface

A small framework-agnostic loader API is preferred. Proposed methods:
- `mount(element, options)`
- `open()`, `close()`
- `setTheme(theme)`
- `setLocale(locale)`
- `setSessionToken(token)`
- `sendMessage(text)` (optional, gated)
- `destroy()`

Events: `ready`, `open`, `close`, `message`, `error`, `action`. Exact names and types are finalized in a versioned TypeScript declaration and protocol schema before release. The SDK must not contain privileged secrets.

## 9. Versioning and compatibility

Loader and iframe are deployed independently. Protocol version negotiation is mandatory. Maintain backward compatibility for at least the documented support window; reject unsupported major versions with a clear error. Cache-bust loader releases and publish a changelog. Customers must be able to pin a stable major version.

## 10. Security checklist

- Strict origin allow-list and postMessage source validation.
- Short-lived scoped tokens and replay protection.
- CSP and frame-ancestors policy.
- No API/provider secret in browser bundles or network payloads.
- Rate limits, abuse detection and bot protection.
- Safe rendering of Markdown/HTML; sanitize links and disallow script execution.
- Clickjacking and open-redirect protections.
- No host-page DOM scraping or collection beyond explicitly supplied fields.

## 11. Acceptance criteria

- Guest visitor can start a session only from an allowed origin.
- Authenticated session requires verified server-signed identity.
- Cross-workspace or expired tokens are rejected.
- No privileged credentials are present in loader, iframe or browser storage.
- Protocol rejects wrong-origin messages and unknown privileged message types.
- Widget is keyboard and screen-reader usable.
