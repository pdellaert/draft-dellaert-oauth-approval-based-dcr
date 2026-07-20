---
title: 'OAuth 2.0 Approval-Based Dynamic Client Registration'
abbrev: 'Approval-Based DCR'
category: std
docname: draft-dellaert-oauth-approval-based-dcr-latest
submissiontype: IETF
number:
date:
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "pdellaert/draft-dellaert-oauth-approval-based-dcr"
  latest: "https://pdellaert.github.io/draft-dellaert-oauth-approval-based-dcr/draft-dellaert-oauth-approval-based-dcr.html"
keyword:
  - OAuth
  - Dynamic Client Registration
  - Approval
  - DCR

author:
  - fullname: 'Philippe Dellaert'
    organization: 'Amazon'
    email: 'pdellaer@amazon.com'

normative:
  RFC2119:
  RFC6585:
  RFC6749:
  RFC7120:
  RFC7591:
  RFC7592:
  RFC8126:
  RFC8174:
  RFC8259:
  RFC8414:
  RFC8628:
  RFC9110:
  RFC9325:
  RFC9700:

informative:
  I-D.ietf-oauth-client-id-metadata-document:
  I-D.gerber-oauth-deferred-token-response:

--- abstract

This document specifies an extension to the OAuth 2.0 Dynamic Client Registration Protocol ({{RFC7591}}) that enables registration of a client with an authorization server through an explicit approval step performed by an approving party, typically the user running the client, without requiring the client to possess an Initial Access Token (IAT) beforehand. The client submits its desired metadata to the existing client registration endpoint and signals that it can accept a deferred, approval-based registration. When the authorization server requires approval, it defers completion of the registration and returns a 202 (Accepted) response carrying a registration code and a verification challenge. The client polls the same registration endpoint, presenting the registration code. The approving party, authenticated to the authorization server through a mechanism of the authorization server's choosing, responds to the verification challenge and approves or denies the registration. On approval, a subsequent poll returns a registered client identifier and, where applicable, a client secret.

This extension preserves the authorization server operator's ability to require an authenticated, policy-controlled approval for every new client record, while removing the operational burden of issuing an Initial Access Token out of band.

--- middle

# Introduction

The OAuth 2.0 Dynamic Client Registration Protocol ({{RFC7591}}) defines a registration endpoint at which a client presents desired metadata and receives a client identifier and, where applicable, a client secret. {{RFC7591}} supports two modes: open registration, in which any caller may register, and protected registration, in which the caller must present an Initial Access Token (IAT). {{RFC7591}} Section 3 deliberately leaves the method by which an IAT is obtained out of scope.

In practice, open registration is unacceptable to many authorization server operators because it permits unbounded database growth, enables denial of service against the registration endpoint, and places no authenticated trust anchor at the root of each client record. Protected registration requires the client to obtain an IAT through some other channel.

A growing class of clients, including command-line tools, agentic software, and native applications distributed at scale, cannot feasibly obtain an IAT through a manual out-of-band process. Their authorization servers also cannot accept open registration. No standardized in-band mechanism exists today for these clients to register with an authenticated approval checkpoint.

Registration binds a specific client instance to the approving party who authorized it and gives the authorization server a per-instance record it can later manage and revoke (see {{approver-association}}, and {{RFC7592}} where supported). Making the client identifier self-describing, as the client identifier metadata document of {{I-D.ietf-oauth-client-id-metadata-document}} does, can remove the need for a stored registration record but does not by itself establish that per-instance binding; this extension targets deployments that require it.

This document specifies a third approach. The client submits its desired metadata to the client registration endpoint defined in {{RFC7591}} and signals, in the request, that it is willing to accept a deferred, approval-based registration. When authorization server policy requires approval, the server defers completion of the registration and returns a 202 (Accepted) response that carries a registration code and a verification challenge, rather than a completed registration. The client polls the same registration endpoint for the result while an approving party (typically the user running the client), separately authenticated to the authorization server, responds to the verification challenge and approves or denies the request. On approval, the authorization server returns the completed registration in response to the next poll. This use of the registration endpoint's response to defer completion, and the requirement that the client signal its willingness before the server defers, follow the pattern of the deferred token response described in {{I-D.gerber-oauth-deferred-token-response}}.

The approving party's identity and authentication mechanism are determined by authorization server policy and are not constrained by this specification. The protocol is the same regardless of who approves.

The polling pattern and short-code presentation are inspired by the OAuth 2.0 Device Authorization Grant ({{RFC8628}}) but this extension is not a device flow. The outcome is a registered client, not an access token. The initiating party is a client application, which may or may not correspond to a discrete device and may or may not run on a headless device.

This specification governs only the registration exchange. It does not constrain where or how the client's metadata is hosted, nor the form of the resulting `client_id` and `client_secret`. Where authorization server policy allows, the issued `client_id` MAY be an HTTPS URL; the client identifier metadata document mechanism of {{I-D.ietf-oauth-client-id-metadata-document}} is one such scheme, and its use with this extension is neither required nor precluded.

## Protocol Flow

~~~ ascii-art
  +----------+                                   +---------------+
  |          |>---(A)-- Registration Request: -->|               |
  |          |          Client Metadata,         |               |
  |          |          registration_mode        |               |
  |          |                                   |               |
  |          |<---(B)-- 202 Accepted (pending) -<|               |
  |          |          Registration Code,       |               |
  |          |          Verification Code,       |               |
  |          |          Verification URI         |               |
  |          |                                   |               |
  |          |>---(C)-- Polling Request -------->|               |
  |          |<---(D)-- 202 Accepted (pending) -<|               |
  |          |                                   |               |
  |  Client  |       +-----------------+         | Authorization |
  |          |       |                 |>--(E)-->|     Server    |
  |          |       | Approving Party | Review  |               |
  |          |       |                 | +       |               |
  |          |       |                 | Approve |               |
  |          |       +-----------------+         |               |
  |          |                                   |               |
  |          |>---(F)-- Polling Request -------->|               |
  |          |<---(G)-- Registration Response --<|               |
  +----------+                                   +---------------+
~~~
{: artwork-align="center" title="Approval-Based Dynamic Client Registration Flow" anchor="fig-flow"}

The approval-based dynamic client registration flow illustrated in {{fig-flow}} includes the following steps. All requests are made to the client registration endpoint of {{RFC7591}}.

(A):
: The client sends its desired client metadata to the registration endpoint, including `registration_mode` to signal that it can accept an approval-based registration.

(B):
: Because approval is required, the authorization server defers the registration and returns a 202 (Accepted) response carrying a registration code, a verification code, and a verification URI.

(C):
: The client polls the registration endpoint with the registration code.

(D):
: The authorization server returns 202 (Accepted) until the approving party acts.

(E):
: The approving party accesses the verification URI, authenticates to the authorization server, reviews the requested metadata or evaluates the client, and approves or denies.

(F):
: After approval, the client polls the registration endpoint again with the registration code.

(G):
: The authorization server returns the registration outcome.

# Terminology

{::boilerplate bcp14-tagged}

This document uses the following terms as defined in {{RFC6749}}: "access token", "authorization code", "authorization endpoint", "authorization server", "client", "client identifier", "client secret", "resource owner", "scope". This document also uses the terms "client metadata" and "Initial Access Token" (IAT) as defined in {{RFC7591}}. The `verification_uri`, `verification_uri_complete`, `expires_in`, and `interval` fields are reused from {{RFC8628}} Section 3.2 with the same meaning. The `registration_code` and `verification_code` fields play the same protocol roles as the `device_code` and `user_code` fields of {{RFC8628}}, scoped to client registration.

Approving Party:
: An entity authenticated to the authorization server that responds to the verification challenge and approves or denies the client's registration request. The authentication strength and trust model applied to the approving party are determined by authorization server policy. The approving party is deliberately distinct from the {{RFC6749}} "resource owner" and "end-user": this protocol approves the registration of a client, not access to a protected resource, and the approving party is not required to own or control any resource. The approving party's authenticated identity is bound to the resulting registration (see {{approver-association}}).

Verification Code:
: A short, transcribable code that the approving party presents at the verification URI to identify a specific pending registration.

Verification URI:
: A URL at which an approving party completes the approval interaction.

Registration Code:
: A long, opaque, high-entropy code that the client uses to poll the client registration endpoint for the outcome of a deferred registration.

# Protocol {#protocol}

This extension operates at the client registration endpoint defined in {{RFC7591}} Section 3. All requests defined by this specification (registration, polling, and cancellation) MUST be sent to that endpoint.

A client signals that it is willing to accept a deferred, approval-based registration through the `registration_mode` member of its registration request. When authorization server policy requires approval, the server defers the registration and returns a 202 (Accepted) response; the client then polls the same endpoint for the outcome.

As required by {{RFC7591}} Section 3, the registration endpoint MUST be protected by a transport-layer security mechanism, and the use of TLS MUST follow the recommendations in {{RFC9325}}. The authorization server MUST NOT redirect requests for the registration endpoint to a URI whose scheme is not HTTPS. Clients MUST NOT follow redirects from the registration endpoint to non-HTTPS URIs.

## Registration Request

The client sends a client registration request as defined in {{RFC7591}} Section 3.1. To request the approval-based flow defined in this specification, the client additionally includes the following member:

registration_mode:
: REQUIRED to request the approval-based flow defined in this specification; otherwise OPTIONAL. A JSON array of registration mode values the client is willing to accept, drawn from the "OAuth Client Registration Modes" registry ({{modes-registry}}). Including the value `approval` signals that the client is prepared to accept a deferred, approval-based registration and to poll for its outcome. The value `immediate`, denoting synchronous completion as in {{RFC7591}}, is accepted by every client and need not be listed. Values the authorization server does not recognize MUST be ignored.

The authorization server MUST NOT defer a registration for a request whose `registration_mode` does not include `approval`. Including `approval` does not entitle the client to a deferred registration; the authorization server completes the request synchronously when its policy does not require approval. A registration request that omits `registration_mode`, or whose `registration_mode` does not include `approval`, is an ordinary {{RFC7591}} request: the authorization server either completes it synchronously or rejects it with a registration error, and never defers it. This preserves compatibility with clients that expect a synchronous response. An authorization server that does not support this extension ignores the `registration_mode` member and processes the request per {{RFC7591}}; a client detects the absence of support from the resulting synchronous response, or in advance from authorization server metadata ({{discovery}}).

For example, the client makes the following request:

~~~ http-message
POST /register HTTP/1.1
Host: as.example.com
Content-Type: application/json

{
  "client_name": "Example Client",
  "redirect_uris": ["http://127.0.0.1:41337/callback"],
  "token_endpoint_auth_method": "client_secret_basic",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "registration_mode": ["approval"]
}
~~~

## Registration Response

The authorization server applies its policy to the registration request. It returns one of two outcomes.

### Immediate Registration

If authorization server policy does not require approval for this request (for example, open registration or a request accompanied by an Initial Access Token per {{RFC7591}}), the authorization server completes the registration synchronously and responds as defined in {{RFC7591}} Section 3.2.1, with an HTTP 201 (Created) status code and the Client Information Response. No polling occurs. This is the ordinary {{RFC7591}} behavior and is unchanged by this specification.

### Deferred Registration {#deferred-registration}

If authorization server policy requires approval and the client's `registration_mode` includes `approval`, the authorization server defers the registration. A deferred registration has been accepted for processing but not yet completed. The authorization server creates a pending registration record and responds with an HTTP 202 (Accepted) status code ({{RFC9110}} Section 15.3.3), `Content-Type: application/json`, `Cache-Control: no-store`, and a JSON object ({{RFC8259}}) containing the following members:

registration_code:
: REQUIRED. The long, opaque code the client uses to poll the registration endpoint for the outcome.

verification_code:
: REQUIRED. The short, transcribable code presented to the approving party at the verification URI to identify a specific pending registration.

verification_uri:
: REQUIRED. URL at which the approving party completes approval.

verification_uri_complete:
: OPTIONAL. A verification URI that includes the `verification_code` (or other information with the same function as the `verification_code`), which is designed for non-textual transmission.

expires_in:
: REQUIRED. Lifetime in seconds of the `registration_code` and the `verification_code`.

interval:
: OPTIONAL. Minimum number of seconds the client MUST wait between polling requests. Defaults to 5. The client MUST retain this value for the lifetime of the `registration_code`, because subsequent 202 (Accepted) responses need not repeat it.

For example, the authorization server responds:

~~~ http-message
HTTP/1.1 202 Accepted
Content-Type: application/json
Cache-Control: no-store

{
  "registration_code": "ZDY2NTFiYTg1N2I4NGFjYWY5N2ZmZGZhYmJiMTE0NjIyMTBjMzYyMwo",
  "verification_code": "FBWS-TRGI",
  "verification_uri": "https://as.example.com/register-verification",
  "verification_uri_complete":
    "https://as.example.com/register-verification?verification_code=FBWS-TRGI",
  "expires_in": 600,
  "interval": 5
}
~~~

### Registration Request Error Responses

When a registration request is malformed or otherwise rejected before any pending registration is created, the authorization server returns an error response as defined in {{RFC7591}} Section 3.2.2: an HTTP 400 (Bad Request) status code, a `Content-Type` of `application/json`, a `Cache-Control: no-store` response header, and a JSON object with `error` and OPTIONAL `error_description` members. The error codes `invalid_redirect_uri`, `invalid_client_metadata`, `invalid_software_statement`, and `unapproved_software_statement` are used as defined in {{RFC7591}} Section 3.2.2.

If authorization server policy requires approval but the client's `registration_mode` does not include `approval`, the authorization server cannot complete the registration synchronously and MUST NOT defer it (see {{protocol}}); it rejects the request with a registration error as described here, using `invalid_client_metadata` unless a more specific code applies.

## Registration Polling

After receiving a 202 (Accepted) response, the client polls for the outcome by sending further requests to the same client registration endpoint. The polling pattern is adopted from {{RFC8628}} Section 3.4.

Because this extension operates at the registration endpoint, whose success status is 201 (Created), a pending registration is conveyed with 202 (Accepted), a requirement to poll less frequently with 429 (Too Many Requests), and a failed registration with 400 (Bad Request).

### Polling Request

The client sends an HTTP POST to the registration endpoint. The request body MUST be a JSON object ({{RFC8259}}) and the `Content-Type` header MUST be `application/json`. The presence of a `registration_code` member distinguishes a polling request from a registration request.

registration_code:
: REQUIRED. The value received in the deferred registration response.

The `registration_code` is a bearer credential: presenting it is sufficient to poll for and complete the pending registration it identifies. It MUST be protected accordingly, both in transit (all requests use TLS, see {{security}}) and by the authorization server (see {{registration-code-compromise}}).

For example, the client makes the following request:

~~~ http-message
POST /register HTTP/1.1
Host: as.example.com
Content-Type: application/json

{
  "registration_code": "ZDY2NTFiYTg1N2I4NGFjYWY5N2ZmZGZhYmJiMTE0NjIyMTBjMzYyMwo"
}
~~~

### Polling Response While Pending {#polling-pending}

While the approving party has not yet completed the approval action, the authorization server responds to each poll with an HTTP 202 (Accepted) status code, indicating that the registration remains pending and the client SHOULD continue polling. Before each new polling request, the client MUST wait at least the number of seconds specified by the `interval` member of the deferred registration response, or 5 seconds if none was provided, and MUST respect any increase in the polling interval required by a 429 (Too Many Requests) response (see {{polling-rate-limit}}).

The 202 (Accepted) response body MAY be empty. The client retains the `registration_code`, `interval`, and `expires_in` values from the deferred registration response; a 202 response need not repeat them.

The following is a non-normative example of a 202 (Accepted) response while the client polls before approval has completed:

~~~ http-message
HTTP/1.1 202 Accepted
Cache-Control: no-store
~~~

### Polling Rate Limiting {#polling-rate-limit}

When the authorization server requires the client to poll less frequently, it responds with an HTTP 429 (Too Many Requests) status code ({{RFC6585}} Section 4) and a `Retry-After` header ({{RFC9110}} Section 10.2.3) giving the minimum number of seconds the client MUST wait before its next poll. The client MUST increase its polling interval to at least the `Retry-After` value for this and all subsequent requests. A 429 response indicates that the registration remains pending; the client SHOULD continue polling. This response plays the role of the `slow_down` response of {{RFC8628}} Section 3.5.

The following is a non-normative example of a 429 (Too Many Requests) response:

~~~ http-message
HTTP/1.1 429 Too Many Requests
Retry-After: 10
Cache-Control: no-store
~~~

### Polling Response on Success

Once the approving party has approved the registration, the next poll completes the registration. The authorization server responds with an HTTP 201 (Created) status code and a body identical in structure to the Client Information Response of {{RFC7591}} Section 3.2.1. Where the authorization server supports the registration management protocol of {{RFC7592}}, the response additionally includes the `registration_access_token` and `registration_client_uri` fields defined in {{RFC7592}} Section 3, as shown in the example below. The `registration_code` is consumed and MUST NOT be usable again.

The following is a non-normative example response body:

~~~ http-message
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store

{
  "client_id": "ZmRkYTk0ZT",
  "client_secret": "ZmRhOTk3Y2VjZjM0NDI1YTM1OTE1N2E4ZmQ5OWJiNDVhZmMxY2EyOAo",
  "client_id_issued_at": 1779050000,
  "client_secret_expires_at": 1810586000,
  "registration_access_token": "reg-OGE4NGQwY2Q0YWJjOTY...",
  "registration_client_uri":
    "https://as.example.com/register/ZmRkYTk0ZT",
  "client_name": "Example Client",
  "redirect_uris": ["http://127.0.0.1:41337/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "token_endpoint_auth_method": "client_secret_basic",
  "response_types": ["code"]
}
~~~

Subsequent management of the registered client, including retrieval, update, and deletion, is performed as defined in {{RFC7592}} using the `registration_access_token` and `registration_client_uri` returned above.

### Polling Error Responses {#errors}

When the registration cannot be completed, the authorization server returns an error response with an HTTP 400 (Bad Request) status code, a `Content-Type` of `application/json`, a `Cache-Control: no-store` response header, and a JSON object using the `error` and OPTIONAL `error_description` members of the error response defined in {{RFC6749}} Section 5.2, as also used by the registration error response of {{RFC7591}} Section 3.2.2. A 400 (Bad Request) response is terminal: the client MUST stop polling.

The error codes `access_denied` and `expired_token` are adopted from {{RFC8628}} Section 3.5. The following error codes are defined for use when polling the registration endpoint under this specification:

access_denied:
: The approving party denied the registration. The client MUST stop polling.

expired_token:
: The `registration_code` has expired and the pending registration is no longer valid. The client MUST stop polling. The client MAY make a new registration request, but SHOULD wait for user interaction before doing so to avoid unnecessary polling.

invalid_registration:
: The pending registration cannot be completed. The `registration_code` is unknown to the authorization server or has already been consumed. These conditions are deliberately not distinguished, to avoid leaking pending-registration state to unauthenticated callers. The client MUST stop polling. The client MAY make a new registration request, but SHOULD do so only in response to user action to avoid unnecessary polling.

On encountering a connection timeout or other transient transport error, clients MUST reduce their polling frequency before retrying, following the same guidance as documented in {{RFC8628}} Section 3.5. The use of an exponential backoff algorithm, such as doubling the polling interval on each connection timeout, is RECOMMENDED.

The following is a non-normative example of an `invalid_registration` response:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_registration",
  "error_description": "The pending registration cannot be completed."
}
~~~

## Cancellation {#cancellation}

A client that has initiated a deferred registration MAY cancel it before it is approved, for example when the user abandons the client. The client sends an HTTP POST to the registration endpoint with a JSON body containing the `registration_code` and a `cancel_registration` member set to `true`.

cancel_registration:
: REQUIRED to request cancellation. Boolean. When `true`, the authorization server discards the pending registration identified by `registration_code`. A polling request omits this member (or sets it to `false`).

Presenting the `registration_code` authorizes cancellation, just as it authorizes polling. On success, the authorization server discards the pending registration and responds with an HTTP 204 (No Content) status code. For a `registration_code` that is unknown or already consumed, including one consumed by a prior cancellation or completion, the authorization server responds with `invalid_registration` as defined in {{errors}} and does not disclose whether a matching pending registration exists.

Cancellation by the client is distinct from denial by the approving party; either results in the pending registration no longer being completable.

For example, the client cancels a pending registration:

~~~ http-message
POST /register HTTP/1.1
Host: as.example.com
Content-Type: application/json

{
  "registration_code": "ZDY2NTFiYTg1N2I4NGFjYWY5N2ZmZGZhYmJiMTE0NjIyMTBjMzYyMwo",
  "cancel_registration": true
}
~~~

# Approval Interaction {#approval}

The approving party interaction described in Section 3.3 of {{RFC8628}} is applicable to the approval interaction of this protocol. The approving party is presented with the `verification_uri` and `verification_code` and asked to access the URI in a user agent of their choosing, such as a web browser or a native application capable of handling the URI. This can be on the same device, or on a different device. The authorization server MUST authenticate the approving party and MUST use a secure TLS-protected session. The approving party MUST provide the `verification_code` before the process can continue. The authorization server SHOULD present the approving party with a clear description of the action the client is trying to complete and request their approval or denial. The authorization server MUST present the approving party with the effective client metadata to be registered.

In case `verification_uri_complete` is provided, clients MAY present this in a non-textual manner, including using a QR (Quick Response) code or NFC (Near Field Communication). The client SHOULD present the `verification_uri` and MUST present the `verification_code`. The authorization server SHOULD display the `verification_code` so the approving party is able to verify with the displayed code from the client. This flow does not change the authentication and approval requirements, it provides an approving party an optimization to not enter the code manually.

This specification does not specify the mechanism by which the authorization server authenticates the approving party or the user agent through which approval is performed. The mechanism, authentication strength, and any additional policy applied to the approving party are determined by authorization server policy.

## Approver Association {#approver-association}

A registration produced by this flow is associated with the specific authenticated approver who approved it. Authorization server implementations SHOULD retain this association so that the resulting client can later be managed or revoked through an approver-driven workflow. This specification does not define such a workflow; what is recorded, how the association is surfaced to the approver, and how revocation is exposed are determined by authorization server policy. Protocol-level management and revocation of a completed client registration remain as defined in {{RFC7592}}, using the `registration_access_token` and `registration_client_uri` returned in the polling response.

# Authorization Server Metadata {#discovery}

This extension defines the following addition to the OAuth 2.0 Authorization Server Metadata ({{RFC8414}}) registry.

registration_modes_supported:
: OPTIONAL. JSON array listing the client registration modes supported at the authorization server's `registration_endpoint`. Values are registered in the "OAuth Client Registration Modes" registry established by this document ({{modes-registry}}). This document defines two values: `immediate`, indicating that a registration request may complete synchronously in the registration response as defined in {{RFC7591}}; and `approval`, indicating that the authorization server supports the deferred, approval-based flow defined in this specification. An authorization server supporting this specification MUST include `approval` in this array.

A client discovers support for the approval-based flow by the presence of `approval` in `registration_modes_supported`. A client MAY also include `registration_mode` with the value `approval` in its registration request without prior discovery; if the authorization server does not support this extension it ignores the member and the registration is processed per {{RFC7591}}.

# Security Considerations {#security}

The Security Considerations of {{RFC7591}} Section 5 are applicable to this protocol, in particular the requirement that the registration endpoint be protected by a transport-layer security mechanism ({{RFC7591}} Section 3 and Section 5, with {{RFC9325}} for TLS configuration). Because this extension operates at the client registration endpoint, the general OAuth security recommendations of {{RFC9700}} also apply.

## Registration Endpoint Abuse

The approval-based flow can be initiated without authentication. An attacker can submit a high volume of registration requests to exhaust the authorization server's pending-registration state or to mint many verification codes in the hope of a social engineering attack. The authorization server SHOULD apply rate-limiting to the registration endpoint.

## Verification Code Phishing

An attacker can initiate a registration, obtain a verification code, and trick a victim into approving the attacker's registration under the victim's account. The attacker's client then receives client credentials associated with the victim's approval at the authorization server.

This is the registration equivalent of the device phishing attack in {{RFC8628}} Section 5.4 and is not eliminated by this extension. To mitigate the risk, authorization servers MUST inform the approving party that they are registering a new client, including the client name, timestamp, and registration details. When the `verification_uri_complete` optimization is used, the authorization server MUST display the verification code for the approving party to verify the same code is shown by the client.

The verification code SHOULD have a short lifetime and SHOULD be single-use to make it harder for an attacker to phish targets.

## Verification Code Brute Forcing

The verification code is short to allow human transcription. An attacker who can submit many verification codes against the verification URI could locate a pending registration not their own and attempt to trick another approving party into approving it. The authorization server MUST rate-limit verification code submission per source IP and per authenticated approver, MUST reject expired or already-consumed verification codes, and MUST require an explicit approval action (not auto-confirm on code entry). Guidance in {{RFC8628}} Section 5.1 applies directly.

## Registration Code Compromise {#registration-code-compromise}

The `registration_code` is a bearer credential: any party that possesses it can poll for and complete, or cancel, the pending registration it identifies. Unlike the `device_code` of {{RFC8628}}, which the client presents together with its `client_id` and, where applicable, client authentication, the `registration_code` is presented on its own, because the client is not yet registered and has no credentials; the `registration_code` is therefore the sole protection for the pending registration. An attacker who guesses, intercepts, or otherwise obtains a `registration_code` (for example through a TLS-terminating intermediary, a server log, or a brute-force attempt) can therefore complete the registration in place of the legitimate client, receiving the issued `client_id` and, where applicable, `client_secret`. Protection of the `registration_code` is essential.

Accordingly, the authorization server MUST protect the registration endpoint with a transport-layer security mechanism ({{RFC7591}} Section 3, with {{RFC9325}} for TLS configuration). The `registration_code` MUST contain at least 128 bits of entropy (160 bits RECOMMENDED), drawn from a cryptographically secure random source per {{RFC6749}} Section 10.10, and MUST NOT be guessable. It SHOULD have a short lifetime, bounded by the `expires_in` returned in the deferred registration response, and MUST be single-use, becoming unusable once the registration is completed, denied, cancelled, or expired. The authorization server SHOULD store the `registration_code` in hashed form, so that read access to its pending-registration store does not yield a usable code, and SHOULD apply the brute-force mitigations recommended in {{RFC8628}} Section 5.2 to polling requests.

Authorization servers MUST NOT treat possession of a `registration_code` as an authorization grant. It authorizes nothing beyond completing or cancelling the single pending registration it identifies.

## Registration Cancellation

Cancellation of a pending registration ({{cancellation}}) is authorized by presenting the `registration_code`, exactly as polling is. It therefore inherits the same protection requirements as the `registration_code` itself ({{registration-code-compromise}}). A cancellation request only discards pending registration state; it returns no credentials and discloses no information about whether a matching pending registration existed, returning `invalid_registration` for an unknown or already-consumed code exactly as a poll does.

# Privacy Considerations

The privacy considerations of {{RFC7591}} Section 6 apply to client metadata exchanged through this protocol. In addition, the verification URI approval interaction surfaces client metadata to the approving party, and client authors SHOULD assume all metadata in the registration request may be visible to the approving party and to the authorization server operator.

## Approver Identity Linkability

The authorization server learns, for every approved registration, which approver authenticated to approve it. Authorization server operators are therefore able to link a given approver to the set of clients that approver has registered. This linkage may be retained for audit, revocation, and management purposes (see {{approver-association}}). Authorization servers SHOULD apply the same retention and access-control policies to this linkage as they do to other authorization decisions made by the same approver.

## Denied and Expired Registrations

Authorization servers MAY retain records of denied or expired registration requests for security monitoring (for example, to detect repeated phishing attempts as in {{RFC8628}} Section 5.4). Such records may include the submitted client metadata, the approver who denied the request, and metadata about the approval session. Authorization servers SHOULD bound the retention of these records and SHOULD NOT use them for purposes beyond security operations.

## Verification URI Access Logs

Access to the verification URI typically generates server-side logs containing the approver's IP address, user-agent, authentication context, and timestamp. These logs may be more sensitive than logs for ordinary authorization endpoints because they tie a specific human approver to a specific newly registered client. Authorization server operators SHOULD treat these logs accordingly.

# IANA Considerations

## OAuth Authorization Server Metadata

IANA is requested to register the following entries in the "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

### Registration Modes Supported

Metadata Name:
: registration_modes_supported

Metadata Description:
: JSON array containing a list of the client registration modes supported at the authorization server's registration endpoint.

Change Controller:
: IESG

Specification Document(s):
: This document

## OAuth Dynamic Client Registration Metadata

IANA is requested to register the following entry in the "OAuth Dynamic Client Registration Metadata" registry established by {{RFC7591}}.

### Registration Mode

Client Metadata Name:
: registration_mode

Client Metadata Description:
: A JSON array of client registration mode values the client is willing to accept for this registration request.

Change Controller:
: IESG

Specification Document(s):
: This document

The `registration_code`, `verification_code`, and `cancel_registration` members defined by this document are exchanged only within the flow defined here and are not registered as OAuth parameters, following the approach taken for the `user_code` and `verification_uri` fields of {{RFC8628}}.

## OAuth Client Registration Modes {#modes-registry}

This document establishes the IANA "OAuth Client Registration Modes" registry. Client registration mode values, used in the `registration_modes_supported` authorization server metadata field, are registered with a Specification Required ({{RFC8126}}) after a two-week review period on the oauth-ext-review@ietf.org mailing list, on the advice of one or more Designated Experts. However, to allow for the allocation of values prior to publication, the Designated Experts may approve registration once they are satisfied that such a specification will be published, per {{RFC7120}}.

Registration requests sent to the mailing list for review should use an appropriate subject (e.g., "Request to register client registration mode: example").

Within the review period, the Designated Experts will either approve or deny the registration request, communicating this decision to the review list and IANA. Denials should include an explanation and, if applicable, suggestions as to how to make the request successful.

IANA must only accept registry updates from the Designated Experts and should direct all requests for registration to the review mailing list.

### Registration Template

Registration Mode Name:
: The name requested (e.g., `approval`). This name is case sensitive. Names that match other registered names in a case-insensitive manner SHOULD NOT be accepted.

Registration Mode Description:
: Brief description of the registration mode's behavior, including how and when the registration outcome is returned.

Change Controller:
: For Standards Track RFCs, list "IESG". For others, give the name of the responsible party.

Specification Document(s):
: Reference to the document or documents that specify the registration mode, preferably including a URI that can be used to retrieve a copy.

### Initial Registry Contents

#### Immediate {#mode-immediate}

Registration Mode Name:
: immediate

Registration Mode Description:
: The registration request may complete synchronously, with the Client Information Response returned directly in the registration response as defined in {{RFC7591}}.

Change Controller:
: IESG

Specification Document(s):
: This document

#### Approval {#mode-approval}

Registration Mode Name:
: approval

Registration Mode Description:
: The authorization server may defer the registration for approval by an approving party, returning a 202 (Accepted) response that the client polls to obtain the outcome, as defined in this document.

Change Controller:
: IESG

Specification Document(s):
: This document

## OAuth Extensions Error Registry

IANA is requested to register the following entries in the "OAuth Extensions Error Registry" established by {{RFC6749}}.

### Invalid Registration

Name:
: invalid_registration

Usage Location:
: client registration polling response, client registration cancellation response

Protocol Extension:
: OAuth 2.0 Approval-Based Dynamic Client Registration

Change Controller:
: IESG

Reference:
: This document

The `access_denied` and `expired_token` error codes used when polling are defined by {{RFC8628}} and are not re-registered by this document.

--- back

# Example Use Cases {#use-cases}
{:numbered="false"}

## Minimal Registration with Human User Approval
{:numbered="false"}

A client registers itself against an authorization server. The approving party is a human user who visits the verification URI in a browser tab already authenticated to the authorization server.

1. The client POSTs its metadata to the registration endpoint with `registration_mode: ["approval"]`.
2. Because approval is required, the authorization server returns a 202 (Accepted) response with a `registration_code`, `verification_code`, and `verification_uri`.
3. The client displays: "Visit https://as.example.com/register-verification and enter code FBWS-TRGI".
4. The user visits the URI, enters the code, sees "Example Client wants to register. Approve?", and clicks Approve.
5. The client's next poll to the registration endpoint returns `client_id` and `client_secret`.

## Registration Producing a URL Client Identifier
{:numbered="false"}

A client registers against an authorization server whose policy issues an HTTPS URL as the `client_id`, which the authorization server later resolves as a client identifier metadata document ({{I-D.ietf-oauth-client-id-metadata-document}}). The approval-based flow is unchanged; only the form of the issued `client_id` differs.

1. The client POSTs its metadata to the registration endpoint with `registration_mode: ["approval"]`.
2. The authorization server returns a 202 (Accepted) response with a `registration_code` and verification data. The client encodes `verification_uri_complete` as a QR code.
3. The approving party scans the QR code, authenticates, reviews the client metadata, and approves.
4. The client's next poll returns a Client Information Response whose `client_id` is an HTTPS URL.

## High-Assurance Approval via a Native Application
{:numbered="false"}

A command-line client on a shared workstation registers against an authorization server whose policy requires a high level of assurance for the approving party. Approval is completed in the authorization server operator's own mobile application rather than a web browser, allowing the operator to combine account authentication, application attestation, and a biometric check in the approval step.

1. The client POSTs its metadata to the registration endpoint with `registration_mode: ["approval"]`.
2. The authorization server returns a 202 (Accepted) response with a `registration_code` and a `verification_uri_complete`. The client renders the `verification_uri_complete` as a QR code and displays the `verification_code` alongside it.
3. The approving party opens the `verification_uri_complete` in the operator's mobile application, which handles the URI directly. The application verifies the approving party against their existing account, attests its own integrity to the authorization server, and requires a biometric confirmation.
4. The application displays the client name and the metadata to be registered together with the `verification_code`, which the approving party checks against the code shown by the client, and the approving party approves.
5. The client's next poll returns `client_id` and `client_secret`.

# Acknowledgments
{:numbered="false"}

This document draws structural inspiration and significant protocol mechanics from the OAuth 2.0 Device Authorization Grant ({{RFC8628}}) by William Denniss, John Bradley, Michael B. Jones, and Hannes Tschofenig. The polling pattern and verification code/URI presentation in this specification are direct adaptations of that work.

The approach of deferring completion at an existing endpoint, rather than defining a new endpoint, follows the deferred token response described in {{I-D.gerber-oauth-deferred-token-response}}.

This document extends the OAuth 2.0 Dynamic Client Registration Protocol ({{RFC7591}}) by Justin Richer (Editor), Michael B. Jones, John Bradley, Maciej Machulak, and Phil Hunt, and is intended to interoperate with the OAuth 2.0 Dynamic Client Registration Management Protocol ({{RFC7592}}) by Justin Richer (Editor), Michael B. Jones, John Bradley, and Maciej Machulak.

The author thanks Jeff Lombardo and Ashay Raut for their review and feedback.
