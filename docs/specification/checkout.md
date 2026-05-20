<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Checkout Capability

* **Capability Name:** `dev.ucp.shopping.checkout`

## Overview

Allows platforms to facilitate checkout sessions. The checkout has to be
finalized manually by the user through a trusted UI unless the AP2 Mandates
extension is supported.

The business remains the Merchant of Record (MoR), and they don't need to become
PCI DSS compliant to accept card payments through this Capability.

### Flow overview

![High-level checkout flow sequence diagram](site:specification/images/ucp-checkout-flow.png)

### Payments

Payment handlers are discovered from the business's UCP profile at
`/.well-known/ucp` and checkout.ucp.payment_handlers. The handlers define
the processing specifications for collecting payment instruments
(e.g., Google Pay, Shop Pay). When the buyer submits payment, the platform
populates the `payment.instruments` array with the collected instrument data.

The `payment` object is optional on checkout creation and may be omitted for
use cases that don't require payment processing (e.g., quote generation, cart
management).

### Fulfillment

Fulfillment is modelled as an extension in UCP to account for diverse use cases.

Fulfillment is optional in the checkout object. This is done to enable a
platform to perform checkout for digital goods without needing to furnish
fulfillment details more relevant for physical goods.

### Checkout Status Lifecycle

The checkout `status` field indicates the current phase of the session and
determines what action is required next. The business sets the status; the
platform receives messages indicating what's needed to progress.

```text
       +------------+                         +---------------------+
       | incomplete |<----------------------->| requires_escalation |
       +-----+------+                         |   (buyer handoff    |
             |                                |  via continue_url)  |
             | all info collected             +----------+----------+
             v                                           |
    +------------------+                                 |
    |ready_for_complete|                                 |
    |                  |                                 |
    | (platform can    |                                 | continue_url
    | call Complete    |                                 |
    |   Checkout)      |                                 |
    +--------+---------+                                 |
             |                                           |
             | Complete Checkout                         |
             v                                           |
   +--------------------+                                |
   |complete_in_progress|                                |
   +---------+----------+                                |
             |                                           |
             +-----------------------+-------------------+
                                     v
                               +-------------+
                               |  completed  |
                               +-------------+

                               +-------------+
                               |  canceled   |
                               +-------------+
          (session invalid/expired - can occur from any state)
```

### Status Values

* **`incomplete`**: Checkout session is missing required information or has
    issues that need resolution. Platform should inspect `messages` array for
    context and should attempt to resolve via Update Checkout.

* **`requires_escalation`**: Checkout session requires information that
    cannot be provided via API, or buyer input is required. Platform should
    inspect `messages` to understand what's needed (see Error Handling below).
    If any `recoverable` errors exist, resolve those first.
    Then hand off to buyer via `continue_url`.

* **`ready_for_complete`**: Checkout session has all necessary information
    and platform can finalize programmatically. Platform can call
    Complete Checkout.

* **`complete_in_progress`**: Business is processing the Complete Checkout
    request.

* **`completed`**: Order placed successfully.

* **`canceled`**: Checkout session is invalid or expired. Platform should
    start a new checkout session if needed.

### Error Handling

The `messages` array contains errors, warnings, and informational messages
about the checkout state. `ucp.status` is the shape discriminator —
`"success"` means the response carries the expected payload, `"error"`
means it carries error information instead. The `severity` field on each
error message prescribes the recommended action:

| Severity                | Meaning                                          | Platform Action                                                   |
| :---------------------- | :----------------------------------------------- | :---------------------------------------------------------------- |
| `recoverable`           | Platform can resolve by modifying inputs via API | Update resource and retry                                         |
| `requires_buyer_input`  | Business requires input not available via API    | Hand off via `continue_url`                                       |
| `requires_buyer_review` | Buyer review and authorization is required       | Hand off via `continue_url`                                       |
| `unrecoverable`         | No resource exists to act on                     | Retry with new resource or inputs, or hand off via `continue_url` |

Errors with `requires_*` severity contribute to `status: requires_escalation`.
Both result in buyer handoff, but represent different checkout states.

* `requires_buyer_input` means the checkout is **incomplete** — the business
requires information their API doesn't support collecting programmatically.
* `requires_buyer_review` means the checkout is **complete** — but policy,
regulatory, or entitlement rules require buyer authorization before order
placement (e.g., high-value order approval, first-purchase policy).

When the business cannot create a new resource or the requested resource
no longer exists, the response contains `ucp.status: "error"` with
`messages` describing the failure — no resource is included in the
response body. When no resource exists to act on, messages SHOULD use
`severity: "unrecoverable"`.
For example, a business may reject a create checkout request where all
items are unavailable:

```json
{
  "ucp": { "version": "2026-01-11", "status": "error" },
  "messages": [
    {
      "type": "error",
      "code": "out_of_stock",
      "content": "All requested items are currently out of stock",
      "severity": "unrecoverable"
    }
  ],
  "continue_url": "https://merchant.com/"
}
```

See [REST](checkout-rest.md#create-checkout) and
[MCP](checkout-mcp.md#create_checkout) binding examples.

#### Error Processing Algorithm

When status is `incomplete` or `requires_escalation`, platforms should process
errors as a prioritized stack. The example below illustrates a checkout with
three error types: a recoverable error (invalid phone), a buyer input
requirement (delivery scheduling), and a review requirement (high-value order).
The latter two require handoff and serve as explicit signals to the platform.
Businesses **SHOULD** surface such messages as early as possible, and platforms
**SHOULD** prioritize resolving recoverable errors before initiating handoff.

```json
{
  "status": "requires_escalation",
  "messages": [
    {
      "type": "error",
      "code": "invalid_phone",
      "severity": "recoverable",
      "content": "Phone number format is invalid"
    },
    {
      "type": "error",
      "code": "schedule_delivery",
      "severity": "requires_buyer_input",
      "content": "Select delivery window for your purchase"
    },
    {
      "type": "error",
      "code": "high_value_order",
      "severity": "requires_buyer_review",
      "content": "Orders over $500 require additional verification"
    }
  ]
}
```

Example error processing algorithm:

```text
GIVEN response with messages array

FILTER errors FROM messages WHERE type = "error"

PARTITION errors INTO
  recoverable           WHERE severity = "recoverable"
  requires_buyer_input  WHERE severity = "requires_buyer_input"
  requires_buyer_review WHERE severity = "requires_buyer_review"
  unrecoverable         WHERE severity = "unrecoverable"

IF unrecoverable is not empty
  RETRY with new resource or inputs, or hand off via continue_url
  RETURN

IF recoverable is not empty
  FOR EACH error IN recoverable
    ATTEMPT to fix error (e.g., reformat phone number)
  CALL Update Checkout
  RETURN and re-evaluate response

IF requires_buyer_input is not empty
  handoff_context = "incomplete, additional input from buyer is required"
ELSE IF requires_buyer_review is not empty
  handoff_context = "ready for final review by the buyer"
```

#### Standard Errors

Standard errors are standardized error codes that platforms are expected to
handle with specific, appropriate UX rather than generic error treatment.

| Code                     | Description                                                                |
| :----------------------- | :------------------------------------------------------------------------- |
| `out_of_stock`           | Specific item or variant is unavailable                                    |
| `item_unavailable`       | Item cannot be purchased (e.g. delisted)                                   |
| `address_undeliverable`  | Cannot deliver to the provided address                                     |
| `payment_failed`         | Payment processing failed                                                  |
| `eligibility_invalid`    | Eligibility claim could not be verified at completion                      |

Businesses **SHOULD** mark standard errors with `severity: recoverable` to
signal that platforms should provide appropriate UX (out-of-stock messaging,
address validation prompts, payment method changes) rather than generic error
messages or deferring to checkout completion.

Example: `out_of_stock` requires specific upfront UX, whereas
`payment_required` can be handled generically at submission.

#### Eligibility Verification at Completion

Platforms provide `context.eligibility` — buyer claims about eligible benefits
such as loyalty membership, payment instrument perks, and similar. These are
claims, not verified facts. Businesses **MAY** act on recognized claims during
the session (adjusting pricing, granting product access, applying provisional
discounts), but all accepted claims **MUST** be resolved before the
transaction can complete.

Unrecognized or inapplicable claims **MUST NOT** block the checkout.
Businesses **SHOULD** notify the buyer via `messages` with `type: "warning"`
when a claim is not accepted, and **MAY** use `type: "info"` to explain
the effects of accepted claims. At completion, accepted claims that remain
unverified **MUST** result in `type: "error"` with
`code: "eligibility_invalid"` (see below).

**Eligibility message codes:**

| Type      | Code                       | When                                               |
| --------- | -------------------------- | -------------------------------------------------- |
| `warning` | `eligibility_not_accepted` | Claim not recognized or not applicable             |
| `info`    | `eligibility_accepted`     | Effect of an accepted claim                        |
| `error`   | `eligibility_invalid`      | Accepted claim could not be verified at completion |

A claim is resolved when it is either **verified** or **rescinded**:

* **Verified**: The Business confirms the claim against a proof provided at
  completion time. UCP does not prescribe how verification occurs — proof
  may come from the payment credential, an identity verification capability,
  or any other mechanism negotiated between Platform and Business.
* **Rescinded**: The Platform removes the claim from `context.eligibility`
  before completion (e.g., buyer changes payment method, withdraws a
  membership claim). Once removed, the Business recalculates without it.

Businesses **MUST NOT** complete a transaction with unresolved eligibility
claims. Unverified claims may result in incorrect pricing or unauthorized
access to restricted products.

**When verification fails:**

Verification failure **MUST** only affect the `messages` array. The
Business **MUST** return an error in `messages` with
`code: "eligibility_invalid"` and `severity: "recoverable"`. Messages
**SHOULD** use the `path` field to identify which specific claim(s) could
not be verified. The Platform **MAY** then provide valid proof and
resubmit, restructure the checkout (e.g., remove ineligible items, update
claims), or abandon the attempt.

For example, the Platform claims a store card benefit via
`context.eligibility`. The Business applies member pricing during the session.
At completion, the payment credential does not match the claimed instrument:

```json
{
  "ucp": { "version": "2026-01-11", "status": "success" },
  "id": "checkout_abc",
  "status": "ready_for_complete",
  "line_items": [ "..." ],
  "totals": [ "..." ],
  "messages": [
    {
      "type": "error",
      "code": "eligibility_invalid",
      "severity": "recoverable",
      "content": "Payment credential does not match the claimed store card benefit.",
      "path": "$.context.eligibility[0]"
    }
  ]
}
```

The Platform can resolve this by having the buyer switch to the qualifying
payment instrument, or by removing the claim from `context.eligibility` to
renegotiate the checkout (obtaining updated pricing, availability, etc.)
and then resubmitting for completion.

### Warning Presentation

The `presentation` field on warning messages controls the rendering
contract the platform **MUST** follow. When omitted, it defaults to
`"notice"`.

| | `notice` (default) | `disclosure` |
| :--- | :--- | :--- |
| Display content | **MUST** | **MUST** |
| Proximity to `path` | **MAY** | **MUST** |
| Dismissible | **MAY** | **MUST NOT** |
| Render `image_url` | **MAY** | **MUST** |
| Render `url` | **MAY** | **SHOULD** |
| Escalate if cannot honor | — | **MUST** via `continue_url` |

#### `notice` (default)

The default rendering contract for warnings. Platforms **MUST** display
the warning content to the buyer. Platforms **MAY** render notices in a
banner, tray, or toast, and **MAY** allow the buyer to dismiss them.

#### `disclosure`

Warnings with `presentation: "disclosure"` carry notices — safety
warnings, allergen declarations, compliance content, etc. — that
**MUST** follow the prescribed rendering contract below.

**Platform requirements:**

* **MUST** display the warning `content` to the buyer.
* **MUST** display the warning in proximity to the component referenced
  by `path`, preserving the association between the disclosure and its
  subject. When `path` is omitted, the disclosure applies to the response
  as a whole.
* **MUST NOT** hide, collapse, or auto-dismiss the warning.
* **MUST** render `image_url` when present (e.g., warning symbol,
  energy class label).
* **SHOULD** render `url` as a navigable reference link when present.

Warnings with `presentation: "disclosure"` **SHOULD** be given rendering
priority over notices.

Platforms that cannot honor the disclosure rendering contract **MUST**
escalate to merchant UI via `continue_url` rather than silently
downgrading to a notice.

**Business requirements:**

* **MUST** set `presentation: "disclosure"` when the warning content must
  be displayed alongside a specific component and must not be hidden or
  auto-dismissed.
* **SHOULD** use the `path` field to associate disclosures with the
  relevant component in the response.
* **SHOULD** provide a `code` that identifies the disclosure category
  (e.g., `prop65`, `allergens`, `energy_label`).
* **SHOULD** provide `image_url` when the disclosure has an associated
  visual element (e.g., warning symbol, energy class label).
* **SHOULD** provide `url` when a reference link is available for the
  buyer to learn more.

#### Disclosure and Acknowledgment

The `presentation` field controls how the warning is rendered, not
whether the checkout can proceed. When affirmative buyer acknowledgment
or authorization is also required, the business **MAY** combine the
disclosure with the escalation mechanisms described in the
[Checkout Status Lifecycle](#checkout-status-lifecycle) to ensure the
appropriate buyer input is obtained.

#### Jurisdiction and Applicability

It is the business's responsibility to determine which disclosures apply
to a given session and return only those that are relevant. Businesses
**SHOULD** use buyer-provided data (`context` and other inputs) and
product attributes to resolve jurisdiction-specific requirements.
Platforms do not affect or resolve disclosure applicability — they render
what they receive from the business.

#### Example

A checkout response containing both a recoverable error and a disclosure
warning on a line item:

```json
{
  "ucp": { "version": "{{ ucp_version }}", "status": "success" },
  "id": "chk_abc123",
  "status": "incomplete",
  "currency": "USD",
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "item_456", "title": "Artisan Nut Butter Collection", "image_url": "https://merchant.com/nut-butter.jpg" },
      "quantity": 1,
      "totals": [{ "type": "subtotal", "amount": 1299 }]
    }
  ],
  "totals": [{ "type": "total", "amount": 1299 }],
  "messages": [
    {
      "type": "error",
      "code": "field_required",
      "path": "$.buyer.email",
      "content": "Buyer email is required",
      "severity": "recoverable"
    },
    {
      "type": "warning",
      "code": "allergens",
      "path": "$.line_items[0]",
      "content": "**Contains: tree nuts.** Produced in a facility that also processes peanuts, milk, and soy.",
      "content_type": "markdown",
      "presentation": "disclosure",
      "image_url": "https://merchant.com/allergen-tree-nuts.svg",
      "url": "https://merchant.com/allergen-info"
    }
  ],
  "links": []
}
```

The platform resolves the recoverable error programmatically while
rendering the allergen disclosure in proximity to the referenced line
item.

## Actions

### Overview

An **action** is an imperative directive from a business to a platform
requesting that the platform perform a scoped interaction — typically
rendering a URL — and, where the interaction has a return value, attach
that value to checkout state at a business-specified path. Actions are
emitted on a top-level `actions` array on the checkout response.

Actions are not messages. Messages describe checkout state and carry
human-readable content for the buyer; actions are protocol directives
that carry no buyer-facing content. The two share the severity
vocabulary so platforms can reason about both through a single
prioritization stack, but they are distinct primitives with distinct
shapes.

Actions import scoped iframe-style semantics into non-iframe transports.
In Embedded Protocol contexts the business already owns a canvas and
handles equivalent interactions internally via the Embedded Protocol
delegation pattern — no host-facing `actions` array is emitted there.
On REST and MCP, actions are the mechanism by which a business can ask
the platform to render one frame or perform one redirect on its behalf
without the full handoff implied by `requires_escalation` +
`continue_url`.

### Opacity Principle

UCP defines the **transport and lifecycle** of actions — how they are
emitted, rendered, correlated, resolved, and acknowledged. UCP does
**not** define the semantic shape of what an action URL renders or
what its result value contains. The capability or handler protocol
that owns the action's `code` is the authoritative source for both.

Concretely:

* The platform **MUST NOT** inspect, transform, or interpret the
    rendered content beyond rendering it per `display`.
* The platform **MUST NOT** inspect, transform, or normalize the
    `value` returned via `postMessage` or query parameters beyond
    writing it at `result_path`.
* The business **MUST NOT** assume the platform can decode or validate
    PSP- or third-party-specific payloads in transit.

The business is a pass-through at the protocol boundary; the rendered
URL and the platform's surface together form the SDK for whatever the
`code` denotes. This is what allows one primitive to carry 3DS
challenges, captchas, offsite payment authorizations, and future
interactions without UCP needing to grow a vocabulary for each.

### Capability Commitment

When a platform supports a capability, it implicitly commits to
executing the **full action surface** that capability and its
extensions emit, including all `display.form` values listed in this
specification. Platforms that cannot render a given form (e.g., a
voice-only agent that cannot render `modal`) **MUST NOT** advertise
support for capabilities whose normative action sets require it.

This means:

* Businesses **MUST NOT** perform runtime conditional checks asking
    "does this platform support modals?" before emitting an action.
* Platforms **MUST NOT** expect runtime renegotiation when an action
    arrives whose form they cannot render. Their only recourses are
    (a) escalate via `continue_url` when the action's severity permits
    it, or (b) treat the inability as a `payment_failed`-shaped
    recoverable error and let the business pivot.
* Handler protocols and capability extensions that introduce new
    action codes inherit this commitment — a platform that supports
    a payment handler commits to that handler's action surface as a
    whole.

This mirrors the equivalent rule for payment handlers: a platform that
completes a checkout with a given handler commits to the handler's
full runtime contract.

### Action Severity

Actions reuse the error-severity vocabulary with the values that apply
to imperative directives:

| Severity               | Platform behavior                                                                                                                                                                                                  |
| :--------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `optional`             | Platform **MAY** complete for better outcomes; non-blocking. Skipping is permitted. Example: a fraud-scoring pixel that improves accuracy but does not gate the transaction.                                       |
| `recoverable`          | Platform **MUST** complete the action (or resolve through another recoverable path) before the checkout can advance. May be retried. Example: a captcha that gates the next state transition.                     |
| `requires_buyer_input` | Platform **MUST** complete. If the platform cannot render the action (e.g., a voice-only agent), it **MUST** escalate via `continue_url`. Example: a 3DS challenge or offsite payment authorization.               |

`unrecoverable` and `requires_buyer_review` do not apply to actions. An
action with no recourse is a contradiction; buyer review is a cognitive
act, not an interaction.

#### Extended Error Processing Algorithm

When `actions` are present, the prioritized stack from
[Error Processing Algorithm](#error-processing-algorithm) extends as
follows. Errors and actions are partitioned separately by source but
share the severity vocabulary:

```text
GIVEN response with messages array AND actions array

FILTER errors  FROM messages WHERE type = "error"

errors_unrecoverable     = errors  WHERE severity = unrecoverable
errors_recoverable       = errors  WHERE severity = recoverable
actions_recoverable      = actions WHERE severity = recoverable
errors_requires_input    = errors  WHERE severity = requires_buyer_input
actions_requires_input   = actions WHERE severity = requires_buyer_input
errors_requires_review   = errors  WHERE severity = requires_buyer_review
actions_optional         = actions WHERE severity = optional

IF errors_unrecoverable is not empty
  RETRY with new resource or inputs, or hand off via continue_url
  RETURN

IF errors_recoverable is not empty
  FOR EACH error IN errors_recoverable
    ATTEMPT to fix programmatically
  CALL Update Checkout
  RETURN and re-evaluate

IF actions_recoverable is not empty
  RENDER each per display; on success, CALL Update Checkout with value at result_path
  ON cancellation or platform-cannot-render: try alternative recoverable paths or escalate
  RETURN and re-evaluate

IF actions_optional is not empty
  RENDER best-effort (e.g., invisible pixels); skipping is permitted

IF actions_requires_input or errors_requires_input or errors_requires_review is not empty
  RENDER actions per display; for any the platform cannot render, ESCALATE via continue_url
```

### Action Object

| Field         | Type     | Required | Notes                                                                                                                                                                                                                                              |
| :------------ | :------- | :------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`          | string   | ✓        | Unique, unguessable identifier for this action. Echoed in the result and used by the platform to correlate the result against the outstanding action. Treat as a capability token: origin checks are defense in depth, not the trust boundary.    |
| `code`        | string   | ✓        | Reverse-domain identifier for the action category, owned by the capability or handler protocol that defines the action's semantics. See [Action Codes](#action-codes).                                                                              |
| `severity`    | string   | ✓        | `optional` \| `recoverable` \| `requires_buyer_input`. See [Action Severity](#action-severity).                                                                                                                                                    |
| `url`         | string   | ✓        | The URL to render or navigate to. **MUST** use the `https` scheme.                                                                                                                                                                                  |
| `expires_at`  | string   | optional | RFC 3339 timestamp after which the platform **SHOULD** stop rendering the surface and emit a `dev.ucp.action.abandoned` result. When absent, the business imposes no deadline; the platform **MAY** apply its own.                                  |
| `result_path` | string   | optional | RFC 9535 JSONPath identifying where the platform **MUST** attach the result value in the next `update_checkout` call (e.g., `$.payment.instruments[0].credential`). Omitted for fire-and-forget actions that have no structured return value (e.g., device-data pixels). See [Attaching the Result](#attaching-the-result) for constraints, including the restriction on writing to `$.signals.*`. |
| `display`     | object   | ✓        | Rendering form and presentation hints. See [Display](#display).                                                                                                                                                                                     |

### Display

The platform **MUST** render the action according to `display.form`. The
platform **SHOULD** honor `width` and `height` hints when present but
**MAY** override them for accessibility, responsive layout, or viewport
constraints.

| Field    | Type    | Required | Notes                                                                                                                                                                       |
| :------- | :------ | :------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `form`   | string  | ✓        | `invisible` \| `inline` \| `modal` \| `full_page` \| `redirect`. Discriminates result transport — see [Result Transport](#result-transport).                                  |
| `width`  | integer | optional | Suggested width in CSS pixels. Not meaningful for `invisible` or `redirect`.                                                                                                |
| `height` | integer | optional | Suggested height in CSS pixels. Not meaningful for `invisible` or `redirect`.                                                                                               |

**Form semantics.** `form` describes the **rendering surface** the
platform must provide, not the implementing primitive. On the web,
each form is naturally realized by an iframe (or a top-level
navigation for `redirect`); on native platforms the equivalent
surface (in-app browser, modal sheet, system browser handoff) applies.
The contract is the same regardless of substrate: the surface loads
`url`, the result returns via the transport in [Result
Transport](#result-transport), and the security model in
[Security](#security) applies.

| `form`      | Surface contract                                                                         |
| :---------- | :--------------------------------------------------------------------------------------- |
| `invisible` | Mount the URL with no visible presence (offscreen iframe on web, headless webview on native). Appropriate for data-collection pixels. |
| `inline`    | Render the surface in the checkout layout flow, near the relevant section.               |
| `modal`     | Render the surface as a modal overlay; block interaction with the rest of checkout.      |
| `full_page` | Render the surface taking the full viewport; block other checkout UI.                    |
| `redirect`  | Navigate the buyer's top-level context to the URL; resume checkout via the return-URL contract. |

### Result Transport

The platform delivers the action result back to itself in one of two
ways, discriminated by whether `display.form === "redirect"`.

#### Frame Transport (postMessage)

Used when `display.form` is `invisible`, `inline`, `modal`, or
`full_page`. The rendered page **MUST** send a `postMessage` to its
parent window with the following payload:

**Success:**

```json
{
    "ucp": { "version": "{{ ucp_version }}", "status": "success" },
    "action_id": "act_xyz",
    "value": "any-json-value-or-null"
}
```

**Cancellation or error:**

```json
{
    "ucp": { "version": "{{ ucp_version }}", "status": "error" },
    "action_id": "act_xyz",
    "messages": [
        {
            "type": "error",
            "code": "dev.ucp.action.abandoned",
            "content": "Buyer dismissed the challenge.",
            "severity": "recoverable"
        }
    ]
}
```

**Standard result codes.** Platforms **MUST** use the following codes
in the error envelope so businesses can distinguish recovery paths
without parsing free-form strings. Capability or handler protocols
**MAY** define additional codes in their own reverse-domain namespace.

| Code                            | Meaning                                                                                          |
| :------------------------------ | :----------------------------------------------------------------------------------------------- |
| `dev.ucp.action.abandoned`      | The buyer dismissed the surface, the surface closed without producing a result, or `expires_at` elapsed. The business can re-emit the action, substitute a recoverable path, or escalate. |
| `dev.ucp.action.failed`         | The surface produced a transport-level error (load failure, network error, sandbox violation). Distinct from a domain-level failure communicated in the success envelope's `value`. |
| `dev.ucp.action.unsupported`    | The platform cannot render the requested `display.form`. Indicates a capability-commitment violation; the business **SHOULD** escalate to `requires_escalation`. |

Domain-level outcomes (e.g., 3DS authentication failed, captcha
verification rejected) are **not** transport errors — they belong in
the success envelope's `value`, shaped by the capability or handler
protocol that owns the `code`. The error envelope is reserved for
transport-shaped failures.

**Correlation is the trust boundary.** The `action.id` is an
unguessable token issued by the business; the platform **MUST** match
the `action_id` in the postMessage against an outstanding action it
issued and reject mismatches. Origin checks are defense in depth, not
the primary control — many real interactions (3DS challenges, third-
party captcha providers) embed content from a different origin than
`action.url`, and a strict origin equality check breaks those flows.

Concretely, platforms **MUST**:

* Match `action_id` to an outstanding action; reject otherwise.
* Sandbox the framed origin per [Security](#security).

And **SHOULD** validate that the `event.origin` is either the origin of
`action.url` or an origin the business has declared trusted for that
action's `code` (see the relevant capability or handler protocol). For
flows where the result is produced by a third-party origin (e.g., 3DS
ACS), the business is responsible for hosting a same-origin relay on
the `action.url` origin that re-emits the result via `postMessage` to
the platform — the third-party origin **MUST NOT** be relied upon to
post directly to the platform.

#### Redirect Transport (Return URL)

Used when `display.form === "redirect"`. The platform **MUST** append a
`ucp_return_url` query parameter to `action.url` before navigating. The
business **MUST** preserve this parameter through any provider redirects
and, upon completion, navigate the buyer to `ucp_return_url` with the
following query parameters:

| Parameter        | Required                              | Notes                                                              |
| :--------------- | :------------------------------------ | :----------------------------------------------------------------- |
| `ucp_action_id`  | ✓                                     | Echoes `action.id`.                                                |
| `ucp_status`     | ✓                                     | `success` or `error`.                                              |
| `ucp_value`      | when `success` and value is non-null  | URL-encoded JSON literal.                                          |
| `ucp_messages`   | optional                              | URL-encoded JSON array of messages (typically for `error` status). |

Platforms **MUST** validate that the redirect lands on their own origin
and that `ucp_action_id` matches an outstanding action.

### Attaching the Result

When `result_path` is present, after receiving a successful result the
platform **MUST** call `update_checkout` with the `value` written at
`result_path`. When `result_path` is omitted, the action is fire-and-
forget; the platform completes the interaction and the business
observes resolution through its own side channels (e.g., webhooks).

`result_path` **MUST NOT** target `$.signals.*` unless the action
result is an independently verifiable third-party attestation that the
platform is relaying (the same constraint that applies to all signal
values; see [Signals](overview.md#signals)). A captcha verification
token from a recognized captcha provider qualifies; a buyer-asserted
value collected through the surface does not.

The business **MUST** re-emit the action in subsequent responses until
it observes the value resolved at `result_path`. Once resolved, the
business **MUST** omit the action from subsequent responses. This
implicit acknowledgment via state — rather than an explicit "action
complete" call — keeps the protocol stateless between transports.

**Correlation and convergence.** Because the platform receives the
postMessage result before it issues the corresponding `update_checkout`
call, there is a window during which the business cannot observe that
the action has been satisfied. The business **MUST NOT** treat absence
of the result in checkout state as cancellation; it relies on the next
`update_checkout` to converge. Platforms **SHOULD** issue
`update_checkout` immediately on receiving a successful result; if the
platform receives a follow-up business response that still includes
the same `id`, it **SHOULD** assume convergence is still in flight and
**MAY** suppress re-rendering on a short debounce.

If the platform reports cancellation or error, the business **MAY**
re-emit the action (possibly with a different `id`, `url`, or
`display`), substitute a different recovery path, or escalate the
checkout to `requires_escalation` with a `continue_url`.

### Action Lifecycle

The same action `id` may appear in multiple business responses while
the platform is converging. The following table is the normative state
view from both sides:

| Phase              | Business sees                                                        | Platform sees                                                                       | Next transition trigger                                                                                |
| :----------------- | :------------------------------------------------------------------- | :---------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------- |
| **Emitted**        | Action is included in the response `actions` array.                  | Receives the action in the response and selects a rendering surface per `display`.  | Platform begins rendering.                                                                             |
| **In flight**      | Continues to include the same action `id` in subsequent responses until the result is observed at `result_path`. | Surface is mounted; awaiting `postMessage` or return-URL callback. May enforce `expires_at`. | Surface produces a result (success or error envelope), surface is abandoned, or `expires_at` elapses.   |
| **Resolved**       | Observes the value written at `result_path` in the next `update_checkout` call. | Has issued `update_checkout` with the value at `result_path` (when `result_path` is present). | Business **MUST** omit the action from subsequent responses.                                            |
| **Abandoned**      | Receives `update_checkout` with no value at `result_path` (or no `update_checkout` for fire-and-forget). May receive an `update_checkout` carrying the platform's error envelope on a path the capability protocol designates. | Has reported `dev.ucp.action.abandoned`, `dev.ucp.action.failed`, or `dev.ucp.action.unsupported`. | Business **MAY** re-emit (possibly with a new `id`, `url`, or `display`), substitute a recoverable path, or escalate via `continue_url`. |
| **Garbage-collected** | Action is no longer present in any response.                       | Discards any pending surface state for that `id`.                                   | Terminal.                                                                                              |

**Convergence note.** Between the platform receiving the surface
result and issuing the corresponding `update_checkout`, the business
cannot observe resolution. Re-emission of the same `id` in this window
is expected — platforms **SHOULD** suppress re-rendering on a short
debounce, and businesses **MUST NOT** treat absence of the value as
cancellation until they have observed an explicit abandonment signal.

### Security

Surfaces rendered to satisfy actions inherit the sandboxing model
defined in [Embedded Protocol — Sandboxing](embedded-protocol.md) and
**MUST NOT** be implemented as a parallel surface. In particular:

* On web, the platform **MUST** sandbox the iframe and apply
    `credentialless` per the Embedded Protocol contract. On native
    platforms, the equivalent surface (in-app browser, system browser,
    webview) **MUST** isolate the action context from any persistent
    buyer session held by the platform.
* Businesses **MUST** set `frame-ancestors` on the action URL to permit
    only the platform origins they trust. Native platforms are out of
    scope of `frame-ancestors`; businesses **SHOULD** treat the
    `action.url` origin as a public surface and not embed buyer-bound
    credentials at that origin.
* The trust boundary for postMessage is `action.id` correlation as
    described in [Frame Transport](#frame-transport-postmessage); origin
    equality is defense in depth.

For redirect-form actions, the platform **MUST** treat the return URL as
the trust boundary: only requests landing on the platform's own origin
with a matching `ucp_action_id` are accepted.

### Out-of-Band Flows

Some authentication flows have no buyer-rendered surface — for example,
out-of-band approval via a banking app, or push-notification
confirmation. These are **out of scope** for `actions` in this
specification: every action **MUST** have a `url` and a `display.form`
the platform can render.

Businesses with out-of-band requirements have two supported paths:

* **Server-mediated resolution.** The business or its handler
    establishes a side channel (webhook, server callback) and surfaces
    the result via a subsequent capability response — no action is
    emitted. The platform sees the resolution as ordinary state
    convergence.
* **Full escalation.** The business returns `requires_escalation` with
    `continue_url`, taking over the buyer experience for the duration
    of the out-of-band step.

Future revisions **MAY** introduce a non-URL action variant (e.g., a
"waiting state" surface) once cross-protocol patterns stabilize. The
URL-required shape is deliberate today: it keeps the v1 primitive
small enough to be implemented uniformly.

### Action Codes

Action codes use reverse-domain naming, mirroring the convention used
for [signals](overview.md#signals). A code is owned by the capability
or handler protocol that defines its semantics — **not** by the
Checkout capability itself. Checkout transports the action; the
defining protocol specifies what the URL renders, what the result
shape is, and where `result_path` is expected to point.

**Checkout-owned codes** are listed below — codes whose semantics are
defined directly by this specification because they apply across
payment handlers and other extensions:

| Code                       | Description                                                              |
| :------------------------- | :----------------------------------------------------------------------- |
| `dev.ucp.checkout.captcha` | Render a captcha widget; result is an opaque third-party verification token suitable for relaying as a signal. |

**Illustrative codes owned by other protocols.** The following are
examples of action codes that other protocols are expected to define
and emit through the Checkout `actions` array. They are listed here to
illustrate the shape of actions in real payment flows — they are
**not** defined by this specification. The payment-handler protocol is
the normative source for these.

| Code (illustrative)                 | Likely owning protocol | Description                                                                |
| :---------------------------------- | :--------------------- | :------------------------------------------------------------------------- |
| `dev.ucp.payment.device_data`       | Payment handler        | Collect device or browser fingerprint data via an invisible pixel.         |
| `dev.ucp.payment.frame_challenge`   | Payment handler        | Render a hosted challenge UI (e.g., 3DS, step-up authentication).          |
| `dev.ucp.payment.offsite_authorization` | Payment handler   | Hand the buyer to an offsite payment provider; result is a payment credential. |

Businesses **MAY** define custom codes in their own reverse-domain
namespace for domain-specific actions. Platforms **SHOULD** handle
unknown codes by rendering the action per `display` and forwarding the
result to `result_path`, even when they don't recognize the semantic
intent.

**Composition with other protocols.** The payment-handler protocol is
expected to define its own normative action set for card-present
flows — device data collection, 3DS / step-up challenge, offsite
authorization, and similar — using this primitive as the transport.
That protocol owns the URL contract, the result shape, and the
recoverable error vocabulary for those codes; this specification owns
only the envelope, the lifecycle, and the security model. Other
capabilities (loyalty step-up, identity verification, age gating) are
expected to compose in the same way.

### Examples

Actions appear under the top-level `actions` array on the checkout
response:

```json
{
    "ucp": { "version": "{{ ucp_version }}", "status": "success" },
    "id": "chk_abc123",
    "status": "incomplete",
    "actions": [ /* ... action entries ... */ ]
}
```

**Device data pixel (invisible, optional, fire-and-forget):**

```json
{
    "id": "act_dd_01HZX9TZK8Q1J5N3D7P2VQ4F0M",
    "code": "dev.ucp.payment.device_data",
    "severity": "optional",
    "url": "https://psp.example.com/device-data/sess_xyz",
    "display": { "form": "invisible" }
}
```

**3DS challenge (modal, required).** The ACS lives on a different
origin than `url`; the business's `url` page is the same-origin relay
that re-emits the ACS result via `postMessage`:

```json
{
    "id": "act_3ds_01HZXA8B2P3W7Y9C6V1K0NMR4D",
    "code": "dev.ucp.payment.frame_challenge",
    "severity": "requires_buyer_input",
    "url": "https://psp.example.com/3ds-relay/sess_xyz",
    "expires_at": "2026-05-19T20:30:00Z",
    "result_path": "$.payment.instruments[0].challenge_complete",
    "display": { "form": "modal", "width": 400, "height": 600 }
}
```

**Captcha (inline, recoverable).** The result is a third-party
verification token the platform relays as a signal:

```json
{
    "id": "act_cap_01HZXAH9N5T8K2R6Y3J7BFQ4M8",
    "code": "dev.ucp.checkout.captcha",
    "severity": "recoverable",
    "url": "https://captcha.example.com/widget/abc",
    "result_path": "$.signals['com.example.captcha_token']",
    "display": { "form": "inline", "width": 300, "height": 80 }
}
```

**Offsite payment (redirect, required):**

```json
{
    "id": "act_off_01HZXAS4R7M1V3P8K0H9DCNB2L",
    "code": "dev.ucp.payment.offsite_authorization",
    "severity": "requires_buyer_input",
    "url": "https://merchant.example.com/paypal-return/sess_xyz",
    "result_path": "$.payment.instruments[0].credential",
    "display": { "form": "redirect" }
}
```

## Continue URL

The `continue_url` field enables checkout handoff from platform to business UI,
allowing the buyer to continue and finalize the checkout session.

### Availability

Businesses **MUST** provide `continue_url` when returning `status` =
`requires_escalation`. For all other non-terminal statuses (`incomplete`,
`ready_for_complete`, `complete_in_progress`), businesses **SHOULD** provide
`continue_url`. For terminal states (`completed`, `canceled`), `continue_url`
**SHOULD** be omitted.

### Format

The `continue_url` **MUST** be an absolute HTTPS URL and **SHOULD** preserve
checkout state for seamless handoff. Businesses **MAY** implement state
preservation using either approach:

#### Server-Side State (Recommended)

An opaque URL backed by server-side checkout state:

```text
https://business.example.com/checkout-sessions/{checkout_id}
```

* Server maintains checkout state tied to `checkout_id`
* Simple, secure, recommended for most implementations
* URL lifetime typically tied to `expires_at`

#### Checkout Permalink

A stateless URL that encodes checkout state directly, allowing reconstruction
without server-side persistence. Businesses **SHOULD** implement support for
this format to facilitate checkout handoff and accelerated entry—for example, a
platform can prefill checkout state when initiating a buy-now flow.

> **Note:** Checkout permalinks are a REST-specific construct that extends the
> [REST transport binding](checkout-rest.md). Accessing a permalink returns a
> redirect to the checkout UI or renders the checkout page directly.

## Scopes

The Checkout capability defines the following well-known scopes for
user-authenticated access:

| Scope | Description |
| :--- | :--- |
| `dev.ucp.shopping.checkout:manage` | All checkout operations on behalf of the authenticated user — create, update, complete, and cancel checkout sessions. |

Scope declaration, derivation, and rules for extending this set with
custom scopes are defined in [Identity Linking — Scopes](identity-linking.md#scopes).

## Guidelines

(In addition to the overarching guidelines)

### Platform

* **MAY** engage an agent to facilitate the checkout session (e.g. add items
    to the checkout session, select fulfillment address). However, the
    agent must hand over the checkout session to a trusted and
    deterministic UI for the user to review the checkout details and place the
    order.
* **MAY** send the user from the trusted, deterministic UI back to the agent
    at any time. For example, when the user decides to exit the checkout screen
    to keep adding items to the cart.
* **MAY** provide agent context when the platform indicates that the request
    was done by an agent.
* **MUST** use `continue_url` when checkout status is `requires_escalation`.
* **MAY** use `continue_url` to hand off to business UI in other situations.
* When performing handoff, **SHOULD** prefer business-provided
    `continue_url` over platform-constructed checkout permalinks.

### Business

* **MUST** send a confirmation email after the checkout has been completed.
* **SHOULD** provide accurate error messages.
* Logic handling the checkout sessions **MUST** be deterministic.
* **MUST** provide `continue_url` when returning `status` =
    `requires_escalation`.
* **MUST** include at least one message with `severity` of
    `requires_buyer_input` or `requires_buyer_review` when returning
    `status` = `requires_escalation`.
* **SHOULD** provide `continue_url` in all non-terminal checkout responses.
* After a checkout session reaches the state "completed", it is considered
    immutable.

## Capability Schema Definition <span id="checkout"></span>

{{ schema_fields('checkout_resp', 'checkout') }}

## Operations

The Checkout capability defines the following logical operations.

| Operation             | Description                                                                        |
| :-------------------- | :--------------------------------------------------------------------------------- |
| **Create Checkout**   | Initiates a new checkout session. Called as soon as a user adds an item to a cart. |
| **Get Checkout**      | Retrieves the current state of a checkout session.                                 |
| **Update Checkout**   | Updates a checkout session.                                                        |
| **Complete Checkout** | Finalizes the checkout and places the order.                                       |
| **Cancel Checkout**   | Cancels a checkout session.                                                        |

### Create Checkout

To be invoked by the platform when the user has expressed purchase intent
(e.g., click on Buy) to initiate the checkout session with the item details.

**Recommendation**: To minimize discrepancies and a streamlined user experience,
product data (price/title etc.) provided by the business through the feeds
**SHOULD** match the actual attributes returned in the response.

{{ method_fields('create_checkout', 'rest.openapi.json', 'checkout') }}

### Get Checkout

It provides the latest state of the checkout resource. After cancellation or
completion it is up to the business on what to return (i.e this can be a long
lived state or expire after a particular TTL - resulting in a 'not found'
error). From the platform there is no specific enforcement for a TTL of the
checkout.

The platform will honor the TTL provided by the business via `expires_at` at the
time of checkout session creation.

{{ method_fields('get_checkout', 'rest.openapi.json', 'checkout') }}

### Update Checkout

Performs a full replacement of the checkout resource.
The platform is **REQUIRED** to send the entire checkout resource containing any
data updates to write-only data fields. The resource provided in the request
will replace the existing checkout session state on the business side.

{{ method_fields('update_checkout', 'rest.openapi.json', 'checkout') }}

### Complete Checkout

This is the final checkout placement call. To be invoked when the user has
committed to pay and place an order for the chosen items. The response of this
call is the checkout object with the `order` field populated in it. The returned
`order` provides necessary identifiers, such as `id` and `permalink_url`,
that can be used to reference the full state of the placed order.
At the time of order persistence, fields from `Checkout` **MAY** be used
to construct the order representation (i.e. information like `line_items`,
`fulfillment` will be used to create the initial order representation).

After this call, other details will be updated through subsequent events
as the order, and its associated items, moves through the supply chain.

{{ method_fields('complete_checkout', 'rest.openapi.json', 'checkout') }}

### Cancel Checkout

This operation will be used to cancel a checkout session, if it can be canceled.
If the checkout session cannot be canceled (e.g. checkout session is
already canceled or completed), then businesses **SHOULD** send back an error
indicating the operation is not allowed. Any checkout session with a status
that is not equal to `completed` or `canceled` **SHOULD** be cancelable.

{{ method_fields('cancel_checkout', 'rest.openapi.json', 'checkout') }}

## Transport Bindings

The abstract operations above are bound to specific transport protocols as
defined below:

* [REST Binding](checkout-rest.md): RESTful API mapping using standard HTTP verbs and JSON payloads.
* [MCP Binding](checkout-mcp.md): Model Context Protocol mapping for agentic interaction.
* [A2A Binding](checkout-a2a.md): Agent-to-Agent Protocol mapping for agentic interactions.
* [Embedded Checkout Binding](embedded-checkout.md): JSON-RPC for powering embedded checkout.

## Entities

### Buyer

{{ schema_fields('buyer', 'checkout') }}

### Context

Context signals are provisional—not authoritative data. Businesses SHOULD use
these values when verified inputs (e.g., shipping address) are absent, and MAY
ignore or down-rank them if inconsistent with higher-confidence signals
(authenticated account, risk detection) or regulatory constraints (export
controls). Eligibility and policy enforcement MUST occur at checkout time using
binding transaction data.

{{ schema_fields('context', 'checkout') }}

### Signals

Environment data provided by the platform to support authorization
and abuse prevention. Unlike `context` (buyer-asserted preferences) and `buyer`
(self-reported identity), signal values MUST NOT be buyer-asserted claims —
platforms provide signals based on direct observation or by relaying
independently verifiable third-party attestations. See
[Signals](overview.md#signals) for details and privacy
requirements.

{{ schema_fields('types/signals', 'checkout') }}

### Attribution

Platform-provided referral and conversion-event context — campaign IDs,
click identifiers, and source/medium markers communicated by the platform.
See [Attribution](overview.md#attribution) for details and consent
requirements.

{{ schema_fields('types/attribution', 'checkout') }}

### Fulfillment Option

{{ extension_schema_fields('fulfillment.json#/$defs/fulfillment_option', 'checkout') }}

### Item

#### Item Create Request

{{ schema_fields('types/item_create_req', 'checkout') }}

#### Item Update Request

{{ schema_fields('types/item_update_req', 'checkout') }}

#### Item

{{ schema_fields('types/item_resp', 'checkout') }}

### Line Item

#### Line Item Create Request

{{ schema_fields('types/line_item_create_req', 'checkout') }}

#### Line Item Update Request

{{ schema_fields('types/line_item_update_req', 'checkout') }}

#### Line Item

{{ schema_fields('types/line_item_resp', 'checkout') }}

### Link

{{ schema_fields('types/link', 'checkout') }}

#### Well-Known Link Types

Businesses **SHOULD** provide all relevant links for the transaction. The
following are the recommended well-known types:

| Type               | Description                                       |
| :----------------- | :------------------------------------------------ |
| `privacy_policy`   | Link to the business's privacy policy             |
| `terms_of_service` | Link to the business's terms of service           |
| `refund_policy`    | Link to the business's refund policy              |
| `shipping_policy`  | Link to the business's shipping policy            |
| `faq`              | Link to the business's frequently asked questions |

Businesses **MAY** define custom types for domain-specific needs. Platforms
**SHOULD** handle unknown types gracefully by displaying them using the `title`
field or omitting them.

### Message

{{ schema_fields('message', 'checkout') }}

### Message Error

{{ schema_fields('types/message_error', 'checkout') }}

#### Error Code

{{ schema_fields('types/error_code', 'checkout') }}

### Message Info

{{ schema_fields('types/message_info', 'checkout') }}

### Message Warning

{{ schema_fields('types/message_warning', 'checkout') }}

### Action

{{ schema_fields('types/action', 'checkout') }}

### Payment

{{ schema_fields('payment', 'checkout') }}

#### Selected Payment Instrument

{{ extension_schema_fields('types/payment_instrument.json#/$defs/selected_payment_instrument', 'checkout') }}

### Payment Credential

{{ schema_fields('payment_credential', 'checkout') }}

### Postal Address

{{ schema_fields('postal_address', 'checkout') }}

### Response

{{ extension_schema_fields('capability.json#/$defs/response_schema', 'checkout') }}

### Total {: #totals }

{{ schema_fields('types/total_resp', 'checkout') }}

#### Rendering Contract

Businesses are the authoritative source for presented totals — their content
and order — because the correct presentation is subject to regional, product,
and regulatory requirements that the business is obligated to satisfy (e.g.,
multi-jurisdiction tax itemization, mandatory fee disclosures).

Platforms MUST render all top-level entries in the order provided:

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
```

Platforms MAY render sub-lines as supplementary detail:

```python
for entry in totals:
    render_line(entry.display_text, entry.amount)
    if entry.lines:
        for sub in entry.lines:
            render_detail_line(sub.display_text, sub.amount)
```

Platforms MUST NOT interpret, filter, reorder, aggregate, or apply display
logic of their own.

Invariants of `totals[]`:

* Every entry carries a `type` and an `amount`. Platforms SHOULD use
  `display_text` when provided. Well-known types have default display labels
  as fallback (see table below); unknown types MUST include `display_text`.
* Amounts are signed integers — negative values are subtractive (e.g.,
  discounts), positive values are additive. The sign IS the direction.
* Exactly one `type: "subtotal"` MUST be present.
* Exactly one `type: "total"` MUST be present.

#### Verification

Platforms MUST NOT substitute their own computed totals for the business's
values. Platforms MAY verify the provided totals:

```python
assert sum(e.amount for e in totals if e.type != "total") == total_entry.amount
```

If the computed sum does not match the `type: "total"` entry, the platform
MUST NOT alter the rendered output — the business's presented totals are
authoritative for display. However, platforms MUST NOT autonomously complete
a checkout with mismatched totals. Platforms SHOULD reject the checkout or
escalate and ask for buyer review via `continue_url`.

#### Well-Known Types

| Type              | Sign | Default label    | Meaning                                   |
| ----------------- | ---- | ---------------- | ----------------------------------------- |
| `subtotal`        | +    | Subtotal         | Sum of line item prices                   |
| `discount`        | −    | Discount         | Order or line-item level discount         |
| `items_discount`  | −    | Item Discounts   | Rollup of line-item discounts             |
| `fulfillment`     | +    | Shipping         | Shipping, delivery, or pickup charges     |
| `tax`             | +    | Tax              | Tax charges                               |
| `fee`             | +    | Fee              | Fees and surcharges                       |
| `total`           | =    | Total            | Authoritative grand total (exactly one)   |

When `display_text` is provided, platforms MUST use it. When omitted on a
well-known type, platforms SHOULD use the default label above. The sign
convention for well-known types is schema-enforced: subtractive types
(discount, items_discount) MUST have negative amounts; additive types
(subtotal, fulfillment, tax, fee) MUST have non-negative amounts.

The `type` field is an open string — businesses MAY use values beyond the
well-known set. Unknown types MUST include `display_text` (schema-enforced)
and the sign on the amount is self-describing.

#### Repeating Types

All types except `subtotal` and `total` MAY appear multiple times —
for example, multi-jurisdiction tax lines or itemized fees.

#### Sub-Lines (`lines`)

Each top-level entry MAY include a `lines` array. Sub-lines share the same
base shape as top-level entries — `display_text` and `amount` — providing an
itemized breakdown under the parent.

**Invariant:** `sum(lines[].amount)` MUST equal the parent entry's `amount`.

The business controls what MUST be rendered (top-level entries) versus what
MAY be optionally surfaced (sub-lines). Platforms SHOULD render sub-lines
when provided.

#### Examples

**Split tax, itemized at top-level:**

```json
"totals": [
  { "type": "subtotal",    "display_text": "Subtotal",    "amount": 5750 },
  { "type": "fulfillment", "display_text": "Shipping",    "amount": 899 },
  { "type": "tax",         "display_text": "Federal Tax", "amount": 332 },
  { "type": "tax",         "display_text": "State Tax",   "amount": 465 },
  { "type": "total",       "display_text": "Total",       "amount": 7446 }
]
```

**Collapsed fees with optional breakdown:**

```json
"totals": [
  { "type": "subtotal", "display_text": "Subtotal", "amount": 4999 },
  {
    "type": "fee", "display_text": "Fees", "amount": 549,
    "lines": [
      { "display_text": "Service Fee", "amount": 399 },
      { "display_text": "Recycling Fee", "amount": 150 }
    ]
  },
  { "type": "tax",   "display_text": "Tax",   "amount": 444 },
  { "type": "total", "display_text": "Total", "amount": 5992 }
]
```

**Discount and account credit — negative amounts:**

```json
"totals": [
  { "type": "subtotal",       "display_text": "Subtotal",       "amount": 10000 },
  { "type": "discount",       "display_text": "Summer Sale",    "amount": -1500 },
  { "type": "tax",            "display_text": "Tax",            "amount": 680 },
  { "type": "account_credit", "display_text": "Account Credit", "amount": -2500 },
  { "type": "total",          "display_text": "Amount Due",     "amount": 6680 }
]
```

### UCP Response Checkout {: #ucp-response-checkout-schema }

{{ extension_schema_fields('ucp.json#/$defs/response_checkout_schema', 'checkout') }}

### Order Confirmation

{{ schema_fields('order_confirmation', 'checkout') }}

### Error Response <span id="error-response"></span>

{{ schema_fields('types/error_response', 'checkout') }}
