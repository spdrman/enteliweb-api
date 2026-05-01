# enteliWEB API Library Specification and Test Plan

**Working package name:** `delta-enteliweb-client`  

**Primary implementation target:** C, Go, Node, Rust, Python, C# 

**Version:** 0.0.1

**Target:** EnteliWEB 4.3.x

\---

## 1\. Executive summary

Build a clean, test-first enteliWEB client library for building automation integrations. The library must prioritize protocol correctness, safe defaults, strong typing where it helps, and raw escape hatches where real-world enteliWEB installations differ by version or configuration.

The client should be organized around two distinct API surfaces:

```text
EnteliWebClient
├── legacy\_rest
│   ├── /enteliweb/api/...
│   ├── HTTP Basic/session-oriented authentication
│   ├── BACnet object browsing and property read/write
│   ├── .multi read/write
│   ├── events, alarms, systems, users, views, permissions
│   └── XML and JSON CSML payloads
│
└── app\_api
    ├── /enteliweb/v1/...
    ├── /enteliweb/index/...
    ├── access/refresh token workflow
    ├── CSRF header workflow
    ├── deployment information and enteliCLOUD detection
    ├── site/admin/log/Secure Connect/eVAULT routes
    └── version-sensitive application API operations
```

The first production milestone should support safe read-only legacy REST calls, basic authentication, object/property reference construction, `.multi` reads, systems/views/users/events/alarm-details reads, and deterministic XML/JSON parsing. Writes must be opt-in, heavily guarded, auditable, and covered by explicit integration-test configuration.

\---

## 2\. Critical design review

### 2.1 Good decisions to preserve

1. **Do not make a shallow endpoint wrapper.**  
A useful library needs typed URL builders, session handling, CSRF support, parsers, pagination helpers, structured errors, and safety controls.
2. **Separate legacy REST from modern application API.**  
The `/enteliweb/api/...` surface and the `/enteliweb/v1/...` surface have different authentication models, route conventions, and stability expectations. They should not share one ambiguous request layer.
3. **Treat enteliCLOUD compatibility as feature-specific, not global.**  
Some application shell and identity behaviour may be shared, but BACnet, tenant routing, permissions, and administrative operations must be validated per deployment.
4. **Explicitly model the `/enteliweb` base path.**  
The library must prevent accidental calls to `/api/...` at the server root when the intended base is `/enteliweb/api/...`.
5. **Keep raw protocol access available.**  
Industrial systems often have version-specific payloads. A raw request layer is necessary, but mutating raw calls must be intentionally enabled.

### 2.2 Design weaknesses to avoid

1. **Do not conflate authentication eras.**  
Legacy session-style authentication and modern token/refresh authentication are separate flows. Sharing tokens, CSRF headers, or refresh logic across both without explicit rules will create subtle failures.
2. **Do not normalize unsafe operations into friendly names.**  
Methods such as `set\_value()` or `write()` are too casual for BAS/BMS control. Mutating methods should be visibly dangerous, gated, and auditable.
3. **Do not hard-code URL strings throughout the codebase.**  
Route construction should flow through typed builders that prevent malformed query strings, double slashes, unescaped path segments, and host-switching pagination URLs.
4. **Do not parse everything into `serde\_json::Value` forever.**  
Raw values are needed at the transport boundary, but the public API should gradually expose typed CSML, BACnet, event, alarm, user, and system models.
5. **Do not over-type the first version.**  
enteliWEB responses can be broad and version-dependent. The right strategy is layered: raw response, semi-typed CSML model, then typed convenience wrappers for stable high-value calls.
6. **Do not let integration tests become destructive by accident.**  
Live write tests must require multiple independent opt-ins: compile-time feature, environment variable, runtime config, and allowlisted object reference.
7. **Do not weaken TLS silently.**  
Some older BAS servers may have certificate issues. The library can support lab-only overrides, but the configuration must be intentionally named and disabled by default.

\---

## 3\. Non-negotiable design principles

1. **Protocol correctness first.** Route builders, encoders, and parsers must be deterministic and test-covered.
2. **Safe reads first.** The initial public API should make read-only operations easy and write operations unavailable or explicitly gated.
3. **Separate API surfaces.** Legacy REST and modern application API should be separate modules with separate sessions.
4. **No credential leakage.** Passwords, access tokens, refresh tokens, CSRF tokens, and session cookies must be redacted from logs, errors, snapshots, and debug output.
5. **Base URL clarity.** A caller should always know whether the resolved legacy base is `https://host/enteliweb/api/`.
6. **Typed builders over string concatenation.** URL paths, query strings, XML, and form payloads must be built by encoders/serializers.
7. **Raw escape hatch with guardrails.** Advanced users can call unwrapped endpoints, but mutating raw calls require explicit unsafe access.
8. **Version tolerance.** Unsupported or unknown response shapes should preserve raw data where possible instead of crashing.
9. **Explicit format selection.** JSON and XML support must be deliberate; adding `alt=json` or XML content type should be visible and testable.
10. **Auditable mutation.** Every mutating request should emit structured audit metadata without secrets.

\---

## 4\. Scope

### 4.1 In scope for v0.1

* Rust async client.
* Base URL configuration and normalization.
* Legacy REST API under `/enteliweb/api`.
* HTTP Basic authentication to `/api/auth/basiclogin?alt=json`.
* Explicit support for deprecated query login only behind a feature flag or unsafe legacy option.
* Cookie store support.
* CSRF token capture and non-GET header injection for the legacy flow.
* Read-only BACnet browsing and read-property operations.
* `.multi` read creation and repeated retrieval.
* Systems, views, users, events, alarm details, and active alarm read calls.
* Raw XML and JSON body support.
* Typed object reference builder.
* Robust URL encoding.
* Structured errors.
* Unit, snapshot, property, parser, mock-server, security, and read-only integration tests.

### 4.2 In scope for v0.2

* BACnet property writes.
* `.multi` writes.
* Event acknowledgement.
* Alarm details update.
* User and permission CRUD.
* Object create/delete.
* Trend log retrieval convenience API.
* Typed parsers for common CSML values.
* Compatibility options for legacy server behaviours.

### 4.3 In scope for v0.3

* Modern application API authentication.
* Token refresh behaviour.
* Deployment information and enteliCLOUD detection.
* Site/admin/log routes.
* Secure Connect routes.
* eVAULT routes.
* Optional CLI.
* Optional Python bindings.

### 4.4 Out of scope initially

* Browser-based OAuth automation for ADFS/AWS/Google/Microsoft Azure.
* Full replacement for the enteliWEB UI.
* Direct BACnet/IP client implementation.
* Undocumented destructive admin operations by default.
* Auto-discovery across internet-facing endpoints.
* Background polling service or daemon.

\---

## 5\. Protocol surface inventory

### 5.1 Base paths

```text
/enteliweb
/enteliweb/api
/enteliweb/v1
/enteliweb/index
/enteliweb/wssession
/enteliweb/wssite
/enteliweb/wslog
/enteliweb/admin
/enteliweb/sysmodule/secureconnect
/enteliweb/sysmodule/entelivault
```

The library should allow the caller to supply either:

```text
https://host.example.edu
```

with `enteliweb\_base\_path("/enteliweb")`, or:

```text
https://host.example.edu/enteliweb
```

with `from\_enteliweb\_url(...)`.

Resolved bases must be inspectable:

```text
legacy\_rest\_base = https://host.example.edu/enteliweb/api/
app\_api\_base     = https://host.example.edu/enteliweb/v1/
```

### 5.2 Legacy REST API classes

|Class/module|Purpose|
|-|-|
|Root API|Summary list of data items available through web services|
|Alarm API|Alarm management functions|
|Device permissions API|Device permission collections|
|BACnet API|BACnet network, device, object, property, log, create/delete operations|
|Data API|Server device configuration and site object information|
|Definitions API|Server definitions|
|Info API|enteliWEB identity/capability information|
|Multi API|Multiple BACnet object values in one operation|
|Sysdata API|Server/system configuration and site object information|
|Sysinfo API|Server/system identity/capability information|
|Event API|Event notification history|
|Object permissions API|Object permission collections|
|Service permissions API|Service permission collections|
|Systems API|enteliWEB systems object properties|
|User API|enteliWEB user properties|
|User group API|User group permissions and membership|
|Views API|Views, systems, dashboard properties|

### 5.3 Key legacy endpoints

```text
GET  /api/auth/basiclogin?alt=json
GET  /api/auth/login?alt=json\&username=...\&password=...   \[deprecated/unsafe]
GET  /api/.info
GET  /api/.sysinfo
GET  /api/.defs
GET  /api/.data
GET  /api/.sysdata
GET  /api/.bacnet
GET  /api/.bacnet/{site}
GET  /api/.bacnet/{site}/{device}
GET  /api/.bacnet/{site}/{device}/{object-type},{instance}
GET  /api/.bacnet/{site}/{device}/{object-type},{instance}/{property}
PUT  /api/.bacnet/{site}/{device}/{object-type},{instance}/{property}?priority={1..16}
POST /api/.multi
GET  /api/.multi/{id}
GET  /api/.multi/{id}/values
GET  /api/systems
GET  /api/systems/{systemname}
GET  /api/views/{view-path...}
GET  /api/user
GET  /api/event?...
GET  /api/event/{eventindex}/AlarmDetails
PUT  /api/event/{eventindex}/AlarmDetails
POST /api/event/ack
GET  /api/alarm/active
```

### 5.4 Legacy authentication and CSRF

Expected basic login request:

```http
GET /enteliweb/api/auth/basiclogin?alt=json
Authorization: Basic <base64(username:password)>
```

The client should support:

* cookie retention;
* CSRF token capture from response body or response headers;
* `X-CSRF-Token` injection on non-GET legacy requests when a token is present;
* mapping `401` to authentication failure;
* mapping CSRF-related `403` responses to a dedicated error.

Deprecated query-string login must be disabled unless explicitly enabled:

```text
/api/auth/login?alt=json\&username=...\&password=...
```

### 5.5 Modern application API

Modern authentication and application operations should live under a separate module.

```text
POST /v1/auth/login/
POST /v1/auth/refreshtoken/
POST /v1/auth/logout/
GET  /index/verify
POST /index/checkuserauthenticationmethod
GET  /wssession/getsessioninfo
GET  /index/getdeploymentinfo
POST /oauth/adfsoauth
POST /oauth/awsoauth
POST /oauth/googleoauth
POST /oauth/microsoftazureoauth
```

Supported authentication method labels should be represented as an enum:

```text
ADFS
App
AWS
enteliWEB
Google
LDAP
Microsoft Azure
```

Token and CSRF state should support:

```text
ACCESS\_TOKEN
REFRESH\_TOKEN
CSRF\_TOKEN
CSRF\_TOKENNAME
CSRF\_TOKEN\_DRF
```

Expected modern non-GET request headers:

```http
Authorization: Bearer <access token>
X-CSRFToken: <csrf token>
```

Refresh flow:

```http
POST /enteliweb/v1/auth/refreshtoken/
Authorization: Bearer <access token>
Content-Type: application/json

{"refresh":"<refresh token>"}
```

### 5.6 Site, log, Secure Connect, and eVAULT operations

These route groups should initially be exposed through raw or lightly typed modules:

```text
/enteliweb/wssite/...
/enteliweb/admin/...
/enteliweb/wslog/...
/enteliweb/v1/log/display
/enteliweb/sysmodule/secureconnect/...
/enteliweb/v1/evault...
/enteliweb/sysmodule/entelivault...
```

The first version should not expose destructive convenience methods for these areas.

\---

## 6\. Package architecture

### 6.1 Repository layout

```text
delta-enteliweb-client/
├── Cargo.toml
├── README.md
├── LICENSE
├── src/
│   ├── lib.rs
│   ├── client.rs
│   ├── config.rs
│   ├── error.rs
│   ├── transport/
│   │   ├── mod.rs
│   │   ├── raw.rs
│   │   ├── legacy\_session.rs
│   │   ├── modern\_session.rs
│   │   ├── csrf.rs
│   │   └── retry.rs
│   ├── legacy\_rest/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   ├── bacnet.rs
│   │   ├── multi.rs
│   │   ├── info.rs
│   │   ├── data.rs
│   │   ├── systems.rs
│   │   ├── views.rs
│   │   ├── events.rs
│   │   ├── alarms.rs
│   │   ├── users.rs
│   │   └── permissions.rs
│   ├── app\_api/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   ├── deployment.rs
│   │   ├── sites.rs
│   │   ├── logs.rs
│   │   ├── settings.rs
│   │   ├── secure\_connect.rs
│   │   └── evault.rs
│   ├── model/
│   │   ├── mod.rs
│   │   ├── object\_ref.rs
│   │   ├── csml.rs
│   │   ├── bacnet.rs
│   │   ├── multi.rs
│   │   ├── event.rs
│   │   ├── alarm.rs
│   │   ├── user.rs
│   │   ├── system.rs
│   │   └── pagination.rs
│   ├── format/
│   │   ├── mod.rs
│   │   ├── json.rs
│   │   ├── xml.rs
│   │   └── escaping.rs
│   └── safety.rs
├── tests/
│   ├── contract\_legacy.rs
│   ├── contract\_modern.rs
│   ├── integration\_live\_readonly.rs
│   ├── integration\_live\_writes.rs
│   ├── fixtures/
│   │   ├── protocol/
│   │   ├── legacy\_rest/
│   │   ├── app\_api/
│   │   ├── xml/
│   │   └── json/
│   └── snapshots/
├── fuzz/
│   ├── fuzz\_targets/
│   │   ├── parse\_csml\_xml.rs
│   │   ├── parse\_csml\_json.rs
│   │   └── parse\_object\_ref.rs
│   └── Cargo.toml
└── examples/
    ├── legacy\_read\_property.rs
    ├── legacy\_multi\_read.rs
    ├── list\_systems.rs
    ├── list\_events.rs
    ├── acknowledge\_event.rs
    └── modern\_deployment\_info.rs
```

### 6.2 Core dependencies

```toml
\[dependencies]
reqwest = { version = "0.12", features = \["json", "cookies", "rustls-tls"] }
tokio = { version = "1", features = \["macros", "rt-multi-thread"] }
serde = { version = "1", features = \["derive"] }
serde\_json = "1"
quick-xml = { version = "0.37", features = \["serialize"] }
url = "2"
base64 = "0.22"
thiserror = "2"
tracing = "0.1"
chrono = { version = "0.4", features = \["serde"] }
secrecy = "0.8"
zeroize = "1"

\[dev-dependencies]
wiremock = "0.6"
insta = { version = "1", features = \["json", "redactions"] }
proptest = "1"
pretty\_assertions = "1"
serde\_path\_to\_error = "0.1"
```

### 6.3 Feature flags

```toml
\[features]
default = \["legacy-rest", "rustls"]
legacy-rest = \[]
modern-app-api = \[]
xml = \[]
json = \[]
unsafe-writes = \[]
unsafe-deprecated-query-login = \[]
native-tls = \["reqwest/native-tls"]
rustls = \["reqwest/rustls-tls"]
python-bindings = \[]
```

Rules:

* `unsafe-writes` enables compile-time access to mutating convenience APIs.
* Raw mutating requests remain possible only through an explicitly named escape hatch such as `client.unsafe\_raw()`.
* `unsafe-deprecated-query-login` is required for `/api/auth/login?username=...\&password=...`.
* `modern-app-api` enables `/v1` and application API routes.

\---

## 7\. Public API design

### 7.1 Client construction

```rust
use delta\_enteliweb\_client::{EnteliWebClient, EnteliWebConfig, ResponseFormat};

let config = EnteliWebConfig::builder()
    .origin("https://bms.example.edu")
    .enteliweb\_base\_path("/enteliweb")
    .default\_format(ResponseFormat::Json)
    .timeout\_secs(30)
    .build()?;

let client = EnteliWebClient::new(config)?;
```

The library should also support callers who already provide `/enteliweb`:

```rust
let client = EnteliWebClient::from\_enteliweb\_url("https://bms.example.edu/enteliweb")?;
```

Resolved URLs should be inspectable:

```rust
assert\_eq!(client.legacy\_rest\_base().as\_str(), "https://bms.example.edu/enteliweb/api/");
assert\_eq!(client.app\_base().as\_str(), "https://bms.example.edu/enteliweb/v1/");
```

### 7.2 Legacy auth

```rust
client
    .legacy\_rest()
    .auth()
    .basic\_login("operator", secret\_password)
    .await?;
```

Expected request:

```http
GET /enteliweb/api/auth/basiclogin?alt=json
Authorization: Basic <base64(username:password)>
```

Deprecated query login should look intentionally risky:

```rust
client
    .legacy\_rest()
    .auth()
    .unsafe\_deprecated\_query\_login("operator", secret\_password)
    .await?;
```

It requires the `unsafe-deprecated-query-login` feature and should emit a warning log event.

### 7.3 Modern app auth

```rust
let auth\_method = client
    .app\_api()
    .auth()
    .check\_auth\_method("operator")
    .await?;

let session = client
    .app\_api()
    .auth()
    .login\_native("operator", secret\_password)
    .await?;
```

Expected modern login behaviour:

* Username/password fields use the server-required encoding convention.
* Response contains access and refresh tokens.
* Response header may include `x-csrftoken`.
* Non-GET app API requests include:

```http
Authorization: Bearer <access>
X-CSRFToken: <csrf>
```

Refresh behaviour:

```http
POST /enteliweb/v1/auth/refreshtoken/
Authorization: Bearer <access>
Content-Type: application/json

{"refresh":"<refresh>"}
```

### 7.4 Object reference builder

```rust
let r = ObjectRef::builder()
    .site("MainSite")
    .device(5600)
    .object\_type("analog-value")
    .instance(1)
    .property("present-value")
    .priority(10)
    .build()?;

assert\_eq!(r.api\_path(), "/.bacnet/MainSite/5600/analog-value,1/present-value?priority=10");
```

Rules:

* Path segments must be URL-encoded safely.
* Object type and instance are joined with a comma, and the comma is preserved.
* Query strings must be built with a real query builder.
* The builder must never create malformed strings such as `?priority=10?alt=json`.
* The reference must support sub-property paths, for example:

```text
/.bacnet/MainSite/10100/multi-state-value,2/state-text/2
```

### 7.5 Legacy BACnet reads

```rust
let value = client
    .legacy\_rest()
    .bacnet()
    .read\_property\_json(\&r)
    .await?;
```

Expected URL:

```text
GET /enteliweb/api/.bacnet/MainSite/5600/analog-value,1/present-value?priority=10\&alt=json
```

### 7.6 Legacy BACnet writes

Writes must be explicit:

```rust
let write = WriteProperty::new(r, CsmlValue::Real(72.0))
    .with\_priority(10)
    .require\_acknowledged\_risk();

client
    .legacy\_rest()
    .bacnet()
    .unsafe\_write\_property\_json(write)
    .await?;
```

Do **not** expose a casual `set\_value()` method in v0.1.

### 7.7 `.multi` reads

Read once without creating a long-lived multi:

```rust
let refs = vec!\[r1, r2, r3];
let values = client
    .legacy\_rest()
    .multi()
    .read\_once\_json(refs)
    .await?;
```

Create a multi with lifetime:

```rust
let multi = client
    .legacy\_rest()
    .multi()
    .create\_json(MultiCreateRequest::new(refs).lifetime\_seconds(60))
    .await?;

let values = client
    .legacy\_rest()
    .multi()
    .get\_values\_json(multi.id())
    .await?;
```

### 7.8 Events and alarms

```rust
let events = client
    .legacy\_rest()
    .events()
    .list(EventQuery::new().sequence\_gt(120).format(ResponseFormat::Json))
    .await?;

let details = client
    .legacy\_rest()
    .events()
    .alarm\_details(event.index())
    .await?;
```

Acknowledge event requires `unsafe-writes`:

```rust
client
    .legacy\_rest()
    .events()
    .unsafe\_acknowledge\_event(event.index(), AckText::new("Acknowledged by integration")?)
    .await?;
```

### 7.9 Systems, views, users

```rust
let systems = client.legacy\_rest().systems().list().await?;
let system = client.legacy\_rest().systems().get("Admin Building Setpoints").await?;
let users = client.legacy\_rest().users().list().await?;
let view = client.legacy\_rest().views().get\_path(\&\["Virtual Stat", "SAT\_SP"]).await?;
```

### 7.10 Raw escape hatch

```rust
let response = client
    .legacy\_rest()
    .raw()
    .get("/api/.info/software-version")
    .format(ResponseFormat::Json)
    .send()
    .await?;
```

Mutating raw calls require an explicit unsafe handle:

```rust
let response = client
    .legacy\_rest()
    .unsafe\_raw()
    .put("/api/.bacnet/MainSite/100/analog-output,1/present-value")
    .xml\_body(xml)
    .send()
    .await?;
```

\---

## 8\. Models

### 8.1 `ObjectRef`

```rust
pub struct ObjectRef {
    pub site: SiteName,
    pub device: DeviceNumber,
    pub object\_type: ObjectType,
    pub instance: ObjectInstance,
    pub property: Option<PropertyPath>,
    pub priority: Option<CommandPriority>,
}
```

Validation:

* `site` non-empty.
* `device` numeric or server-supported string form.
* `object\_type` non-empty; optionally enum for common BACnet types plus custom string.
* `instance` numeric or supported wildcard form.
* `priority` must be 1 through 16.

### 8.2 `CsmlValue`

```rust
pub enum CsmlValue {
    Any,
    Null,
    Boolean(bool),
    Real(f64),
    Double(f64),
    Signed(i64),
    Unsigned(u64),
    Enumerated(String),
    CharacterString(String),
    ObjectIdentifier { object\_type: String, instance: u32 },
    List(Vec<CsmlValue>),
    Struct(BTreeMap<String, CsmlValue>),
    Choice(Box<CsmlValue>),
    Sequence(BTreeMap<String, CsmlValue>),
    Unknown { base: String, raw: serde\_json::Value },
}
```

Rules:

* Preserve unknown `$base` values.
* Preserve metadata such as `via`, `error`, `priority`, `units`, `truncated`, and `next`.
* Do not collapse all values to strings.

### 8.3 `MultiValue`

```rust
pub struct MultiValue {
    pub index: String,
    pub via: ObjectRefLike,
    pub value: CsmlValue,
    pub error: Option<ApiItemError>,
}
```

### 8.4 `ApiError`

```rust
pub enum EnteliWebError {
    InvalidBaseUrl { input: String, reason: String },
    UrlBuild { reason: String },
    Http(reqwest::Error),
    Status { status: StatusCode, body: String, request\_id: Option<String> },
    Unauthorized,
    ForbiddenCsrf,
    AuthFailed { code: Option<String>, message: String },
    UnsupportedAuthMethod { method: String },
    TokenRefreshFailed,
    ParseJson { error: serde\_json::Error, body\_excerpt: String },
    ParseXml { reason: String, body\_excerpt: String },
    InvalidObjectRef { reason: String },
    InvalidPriority { value: u8 },
    UnsafeWritesDisabled,
    DeprecatedQueryLoginDisabled,
    UnsupportedFeature { feature: String, server\_version: Option<String> },
    BmsServerError { code: Option<String>, message: String, raw: String },
}
```

Security rule: body excerpts must be redacted for known secret fields.

\---

## 9\. Endpoint specification

### 9.1 Legacy auth

|Operation|Method|Path|Status|
|-|-:|-|-|
|Basic login|GET|`/api/auth/basiclogin?alt=json`|v0.1|
|Deprecated login|GET|`/api/auth/login?alt=json\&username=...\&password=...`|feature-gated|

### 9.2 Legacy info and definitions

|Operation|Method|Path|Status|
|-|-:|-|-|
|Get all summary|GET|`/api` or `/api/`|v0.2|
|Instance info|GET|`/api/.info`|v0.1|
|Instance info property|GET|`/api/.info/{property}`|v0.1|
|System info|GET|`/api/.sysinfo`|v0.2|
|System info property|GET|`/api/.sysinfo/{property}`|v0.2|
|Definitions|GET|`/api/.defs`|v0.2|
|Definitions alias|GET|`/api/.definitions`|v0.2|

### 9.3 Legacy BACnet

|Operation|Method|Path|Status|
|-|-:|-|-|
|List sites|GET|`/api/.bacnet?skip=\&max-results=`|v0.1|
|List devices in site|GET|`/api/.bacnet/{site}?skip=\&max-results=`|v0.1|
|List objects in device|GET|`/api/.bacnet/{site}/{device}?skip=\&max-results=`|v0.1|
|Get object|GET|`/api/.bacnet/{site}/{device}/{object-type},{instance}`|v0.1|
|Get property|GET|`/api/.bacnet/{site}/{device}/{object-type},{instance}/{property}`|v0.1|
|Get sub-property|GET|`/api/.bacnet/{site}/{device}/{object-type},{instance}/{property}/{path...}`|v0.2|
|Get trend log records|GET|`/api/.bacnet/{site}/{device}/trend-log,{instance}/log-buffer?...`|v0.2|
|Write property|PUT|`/api/.bacnet/{site}/{device}/{object-type},{instance}/{property}?priority=`|v0.2 gated|
|Create object|POST|`/api/.bacnet/{site}/{device}`|v0.3 gated|
|Delete object|DELETE|`/api/.bacnet/{site}/{device}/{object-type},{instance}`|v0.3 gated|

### 9.4 Legacy `.multi`

|Operation|Method|Path|Status|
|-|-:|-|-|
|Create multi read resource|POST|`/api/.multi` or `/api/.multi?alt=json`|v0.1|
|Write multiple values|POST|`/api/.multi` or `/api/.multi?alt=json`|v0.2 gated|
|Create multi and write values|POST|`/api/.multi` or `/api/.multi?alt=json`|v0.2 gated|
|Get all multis|GET|`/api/.multi`|v0.2|
|Get one multi|GET|`/api/.multi/{id}`|v0.1|
|Get one multi values|GET|`/api/.multi/{id}/values`|v0.1|

### 9.5 Legacy data/sysdata

|Operation|Method|Path|Status|
|-|-:|-|-|
|Data root|GET|`/api/.data`|v0.2|
|Database revision|GET|`/api/.data/database-revision`|v0.2|
|Nodes|GET|`/api/.data/nodes`|v0.2|
|Devices nodes|GET|`/api/.data/nodes/devices`|v0.2|
|Networks nodes|GET|`/api/.data/nodes/networks`|v0.2|
|Objects nodes|GET|`/api/.data/nodes/objects`|v0.2|
|Protocol nodes|GET|`/api/.data/nodes/protocols`|v0.2|
|Events|GET|`/api/.data/events?skip=\&max-results=`|v0.2|
|Histories|GET|`/api/.data/histories?skip=\&max-results=`|v0.2|
|Sysdata equivalents|GET|`/api/.sysdata/...`|v0.3|

### 9.6 Legacy events and alarms

|Operation|Method|Path|Status|
|-|-:|-|-|
|List events|GET|`/api/event?sequence-gt=\&sequence-le=\&alt=json`|v0.1|
|User notification index|GET|`/api/event/extensions/usernotificationindex`|v0.2|
|Get alarm details|GET|`/api/event/{eventindex}/AlarmDetails`|v0.1|
|Get alarm details property|GET|`/api/event/{eventindex}/AlarmDetails/{property}`|v0.2|
|Set alarm details|PUT|`/api/event/{eventindex}/AlarmDetails`|v0.2 gated|
|Acknowledge event|POST|`/api/event/ack`|v0.2 gated|
|Active alarms|GET|`/api/alarm/active?limit=\&start=\&dbquery...`|v0.1|
|Alarm category|GET|`/api/alarm/category`|v0.2|
|Alarm detail|GET|`/api/alarm/detail`|v0.2|
|Alarm event|GET|`/api/alarm/event`|v0.2|

### 9.7 Legacy users and permissions

|Operation|Method|Path|Status|
|-|-:|-|-|
|List users|GET|`/api/user`|v0.1|
|Get user|GET|`/api/user/{username}`|v0.2|
|Get user property|GET|`/api/user/{username}/{property}`|v0.2|
|Create user|POST|`/api/user`|v0.3 gated|
|Update user property|PUT|`/api/user/{username}/{property}`|v0.3 gated|
|Delete user|DELETE|`/api/user/{username}`|v0.3 gated|
|User groups|GET|`/api/usergroup`|v0.2|
|User group details|GET|`/api/usergroup/{groupname}`|v0.2|
|Permission sets|GET|`/api/usergroup/{groupname}/permission-sets`|v0.2|
|Device permissions|GET/POST/DELETE|`/api/devicepermissions...`|v0.3|
|Object permissions|GET/POST/DELETE|`/api/objectpermissions...`|v0.3|
|Service permissions|GET/POST/DELETE|`/api/servicepermissions...`|v0.3|

### 9.8 Legacy systems and views

|Operation|Method|Path|Status|
|-|-:|-|-|
|List systems|GET|`/api/systems`|v0.1|
|Get system|GET|`/api/systems/{systemname}`|v0.1|
|Get system object|GET|`/api/systems/{systemname}/{objectname}`|v0.2|
|Get view|GET|`/api/views/{view path...}`|v0.1|
|Virtual stat|GET|`/api/views/Virtual+Stat/{stat}`|v0.1 compatibility|

### 9.9 Modern app API

Treat as version-sensitive.

|Operation|Method|Path|Status|
|-|-:|-|-|
|Native login|POST|`/v1/auth/login/`|v0.3|
|Refresh token|POST|`/v1/auth/refreshtoken/`|v0.3|
|Logout|POST|`/v1/auth/logout/`|v0.3|
|Verify|GET/POST TBD|`/index/verify`|v0.3|
|Check auth method|POST form|`/index/checkuserauthenticationmethod`|v0.3|
|Session info|GET|`/wssession/getsessioninfo`|v0.3|
|Deployment info|GET|`/index/getdeploymentinfo`|v0.3|
|OAuth redirects|POST|`/oauth/{provider}oauth`|raw only|
|Site operations|mixed|`/wssite/...` and `/admin/...`|raw only initially|
|Logs|mixed|`/wslog/...` and `/v1/log/display`|raw only initially|
|Secure Connect|mixed|`/sysmodule/secureconnect/...`|raw only initially|
|eVAULT|mixed|`/v1/evault...` and `/sysmodule/entelivault...`|raw only initially|

\---

## 10\. Payload format requirements

### 10.1 JSON CSML values

Example `.multi` create request body pattern:

```json
{
  "$base": "Struct",
  "lifetime": {
    "$base": "Unsigned",
    "value": 60
  },
  "values": {
    "$base": "List",
    "1": {
      "$base": "Any",
      "via": "/.bacnet/MainSite/5600/analog-value,1/present-value"
    }
  }
}
```

The library should have a builder that produces stable JSON field order for snapshot testing.

### 10.2 XML CSML values

Example `.multi` create request body pattern:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Struct xmlns="http://bacnet.org/csml/1.2">
  <Unsigned name="lifetime" value="60"/>
  <List name="values">
    <Any name="1" via="/.bacnet/MainSite/5600/analog-value,1/present-value"/>
  </List>
</Struct>
```

XML writers must escape attribute values:

|Character|Escape|
|-:|-|
|`\&`|`\&amp;`|
|`<`|`\&lt;`|
|`>`|`\&gt;` where needed|
|`"`|`\&quot;`|
|`'`|`\&apos;` where needed|

Do not use string concatenation for XML payloads in production implementation except inside low-level tested serializers.

### 10.3 Form payloads

The modern check-auth-method endpoint uses form-encoded content:

```http
Content-Type: application/x-www-form-urlencoded

username=<encoded username>
```

The library must use a form encoder, not hand-built strings.

\---

## 11\. Security and operational safety

### 11.1 Credential handling

* Use `secrecy::SecretString` for passwords and tokens.
* Redact credentials in `Debug`, logs, errors, and test snapshots.
* Do not put credentials in URLs except in the feature-gated deprecated login method.
* Warn loudly when deprecated query login is used.

### 11.2 Write safety

Mutating APIs that can affect real equipment must require:

1. Compile-time feature: `unsafe-writes`.
2. Runtime config: `allow\_writes(true)`.
3. Call-site naming: method must include `unsafe\_` prefix or a similarly explicit risk acknowledgment.
4. Optional allowlist of writable object references.
5. Audit callback hook.

Example:

```rust
let client = EnteliWebConfig::builder()
    .origin("https://bms.example.edu")
    .allow\_writes(false)
    .build()?;
```

Default is always `allow\_writes(false)`.

### 11.3 Audit events

Emit structured tracing events for:

* login attempt/success/failure without password;
* token refresh attempt/success/failure;
* every mutating request;
* CSRF failure;
* response status >= 400;
* retries;
* base URL normalization.

### 11.4 TLS

Default to `rustls`. If legacy servers require native TLS or invalid certificates, make it explicit:

```rust
.danger\_accept\_invalid\_certs\_for\_lab\_only(true)
```

Never enable invalid certificates by default.

\---

## 12\. Test suite overview

The test suite must be treated as part of the product. The goal is to prevent an integration library from silently writing wrong values, building wrong paths, swallowing CSRF failures, or mis-parsing building-system state.

Test layers:

1. Unit tests.
2. Snapshot tests.
3. Property-based tests.
4. Mock-server contract tests.
5. Parser fixture tests from protocol examples.
6. Fuzz tests.
7. Security tests.
8. Live read-only integration tests.
9. Live write integration tests, disabled unless explicitly enabled.
10. Compatibility tests across enteliWEB versions when available.

\---

## 13\. Detailed test matrix

### 13.1 Configuration and URL building

|ID|Test|Expected|
|-|-|-|
|CFG-001|`origin("https://host") + base\_path("/enteliweb")`|legacy base = `https://host/enteliweb/api/`|
|CFG-002|`from\_enteliweb\_url("https://host/enteliweb")`|same as above|
|CFG-003|trailing slash in origin|no double slash|
|CFG-004|trailing slash in base path|no double slash|
|CFG-005|missing scheme|invalid base URL error|
|CFG-006|path contains `/api` already|reject or normalize with warning; do not produce `/api/api`|
|CFG-007|base contains query string|reject|
|CFG-008|base contains fragment|reject|
|CFG-009|app base path generation|`/enteliweb/v1/`|
|CFG-010|secure-connect base path generation|`/enteliweb/sysmodule/secureconnect/`|

### 13.2 Object reference tests

|ID|Test|Expected|
|-|-|-|
|REF-001|Build standard analog value present-value ref|`/.bacnet/MainSite/5600/analog-value,1/present-value`|
|REF-002|Add priority 10|query `priority=10`|
|REF-003|Add `alt=json` to prioritized ref|query `priority=10\&alt=json`, never `?priority=10?alt=json`|
|REF-004|Site with spaces|path encodes as required by server; snapshot expected|
|REF-005|Object property subpath `state-text/2`|preserves subpath segments|
|REF-006|Empty site|error|
|REF-007|Priority 0|error|
|REF-008|Priority 17|error|
|REF-009|Custom object type|accepted as string|
|REF-010|Object type with illegal slash|rejected unless escaped as a segment|
|REF-011|Parse contract example `/MainSite/10100/binary-output,1/present-value`|correct fields|
|REF-012|Parse trend-log log-buffer path|object type `trend-log`, property `log-buffer`|
|REF-013|Round-trip parse/build for generated refs|stable|
|REF-014|Property path with numeric segment|stable|
|REF-015|Query builder preserves existing priority while adding format|stable|

### 13.3 Legacy auth tests

|ID|Test|Expected|
|-|-|-|
|AUTH-L-001|Basic login sends GET to `/api/auth/basiclogin?alt=json`|correct path|
|AUTH-L-002|Basic login sends `Authorization: Basic ...`|correct base64|
|AUTH-L-003|Password redacted in debug logs|no secret appears|
|AUTH-L-004|Login captures `\_csrfToken` from JSON body if present|stored in session|
|AUTH-L-005|Login captures CSRF header if present|stored in session|
|AUTH-L-006|POST after login sends `X-CSRF-Token`|present|
|AUTH-L-007|GET after login does not require CSRF|no CSRF header unless configured|
|AUTH-L-008|401 maps to `Unauthorized`|correct error|
|AUTH-L-009|403 maps to `ForbiddenCsrf` when body/status indicates token failure|correct error|
|AUTH-L-010|Deprecated query login without feature|compile/runtime disabled|
|AUTH-L-011|Deprecated query login with feature|emits warning and encodes user/pass|
|AUTH-L-012|Cookie jar retains session cookies|subsequent request includes cookie|

### 13.4 Modern app auth tests

|ID|Test|Expected|
|-|-|-|
|AUTH-M-001|Check auth method posts form body|`Content-Type: application/x-www-form-urlencoded`|
|AUTH-M-002|Check auth method encodes username correctly|contract-compatible value|
|AUTH-M-003|Native login posts to `/v1/auth/login/`|correct path|
|AUTH-M-004|Native login stores access token|available for request interceptor|
|AUTH-M-005|Native login stores refresh token|available for refresh|
|AUTH-M-006|Login captures `x-csrftoken` header|stored as modern CSRF|
|AUTH-M-007|Non-GET app request sends `Authorization: Bearer`|present|
|AUTH-M-008|Non-GET app request sends `X-CSRFToken`|present|
|AUTH-M-009|GET app request sends bearer only|no CSRF unless configured|
|AUTH-M-010|401 triggers refresh once|second request succeeds|
|AUTH-M-011|Refresh failure clears tokens|session invalidated|
|AUTH-M-012|Refresh loop prevented|no infinite retry|
|AUTH-M-013|Logout sends access/refresh body|correct request|
|AUTH-M-014|External OAuth method returns unsupported/browser-required error|no fake automation|
|AUTH-M-015|Deployment info parses `ecloud` boolean|correct|

### 13.5 Raw transport tests

|ID|Test|Expected|
|-|-|-|
|HTTP-001|GET JSON success|parsed body + status available|
|HTTP-002|GET XML success|raw body + parsed body available|
|HTTP-003|POST JSON sets content type|`application/json`|
|HTTP-004|POST XML sets content type|`application/xml`|
|HTTP-005|Form request sets content type|`application/x-www-form-urlencoded`|
|HTTP-006|Timeout maps to transport error|no panic|
|HTTP-007|Invalid TLS maps to transport error|no secret leakage|
|HTTP-008|500 returns status error with redacted excerpt|correct|
|HTTP-009|HTML login page response detected as auth/session failure|correct|
|HTTP-010|Next URL absolute path followed safely|same host unless explicitly allowed|
|HTTP-011|Next URL from different host rejected by default|SSRF guard|
|HTTP-012|Query parameters encode arrays where needed|stable|

### 13.6 BACnet read contract tests

|ID|Test|Mock setup|Expected|
|-|-|-|-|
|BAC-001|List sites XML|returns collection|parsed collection|
|BAC-002|List sites JSON|returns JSON collection|parsed collection|
|BAC-003|List devices with pagination|response has `next`|exposes next page token/url|
|BAC-004|List objects in device|protocol fixture|parsed object list|
|BAC-005|Get object|analog-input fixture|parsed object properties|
|BAC-006|Get property real value|present-value fixture|`CsmlValue::Real`|
|BAC-007|Get property enum value|mode/status fixture|`CsmlValue::Enumerated` or unknown enum preserved|
|BAC-008|Missing object|server item error|typed item error|
|BAC-009|Permission denied|403|`Forbidden`/permission error|
|BAC-010|Add `alt=json`|query appended correctly|success|
|BAC-011|XML default read|no `alt=json`|XML parser path used|
|BAC-012|Non-ASCII site name|encoded path|success in mock|

### 13.7 BACnet write safety tests

|ID|Test|Expected|
|-|-|-|
|WRITE-001|Build without `unsafe-writes`|write methods unavailable or return disabled|
|WRITE-002|`unsafe-writes` but `allow\_writes(false)`|`UnsafeWritesDisabled`|
|WRITE-003|`allow\_writes(true)` but ref not allowlisted|denied|
|WRITE-004|Allowlisted ref writes priority 10|correct URL and body|
|WRITE-005|Priority 0 rejected|error before request|
|WRITE-006|Priority 17 rejected|error before request|
|WRITE-007|XML write escapes text|correct XML|
|WRITE-008|JSON write serializes numeric value|correct JSON|
|WRITE-009|Audit callback invoked before request|present|
|WRITE-010|Failed write audit includes status but no secret|present and redacted|

### 13.8 `.multi` tests

|ID|Test|Expected|
|-|-|-|
|MUL-001|Create JSON read multi|contract-compatible body|
|MUL-002|Create XML read multi|contract-compatible body|
|MUL-003|Lifetime 60 seconds|lifetime included|
|MUL-004|Read once without lifetime|no persistent multi if one-shot mode is used|
|MUL-005|Get values|parses list|
|MUL-006|One value error in list|item error preserved|
|MUL-007|Mixed numeric/string values|all preserved|
|MUL-008|Multi write disabled by default|denied|
|MUL-009|Multi write allowlisted refs only|denied for non-allowlisted|
|MUL-010|Multi ID path encoding|safe|

### 13.9 Events and alarms tests

|ID|Test|Expected|
|-|-|-|
|EVT-001|List events with sequence greater-than|query correct|
|EVT-002|List events with two filters|both query params encoded|
|EVT-003|Pagination next URL same-origin|followed safely|
|EVT-004|Pagination next URL different-origin|rejected|
|EVT-005|Parse event index|stable|
|EVT-006|Parse event state|stable/unknown preserved|
|EVT-007|Get alarm details|parsed details|
|EVT-008|Set alarm details disabled by default|denied|
|EVT-009|Set alarm details escaped XML|correct|
|EVT-010|Acknowledge event disabled by default|denied|
|EVT-011|Acknowledge event enabled|correct path/body|
|EVT-012|Active alarms query|correct params|

### 13.10 Systems, views, users, permissions tests

|ID|Test|Expected|
|-|-|-|
|SYS-001|List systems|parsed list|
|SYS-002|Get system with spaces|encoded path|
|SYS-003|Get system object|encoded nested path|
|VIEW-001|Get view by path|encoded path segments|
|VIEW-002|Get `Virtual Stat` path|stable path|
|USER-001|List users|parsed list|
|USER-002|Get user with special chars|encoded path|
|PERM-001|List user groups|parsed list|
|PERM-002|Permission denied|typed permission error|
|PERM-003|Permission mutation disabled by default|denied|

### 13.11 Modern application API tests

|ID|Test|Expected|
|-|-|-|
|APP-001|Deployment info|parses version/site/ecloud fields|
|APP-002|Session info|parsed or raw JSON preserved|
|APP-003|Check auth method ADFS|enum parsed|
|APP-004|Check auth method LDAP|enum parsed|
|APP-005|Unknown auth method|unknown variant preserved|
|APP-006|OAuth method|returns browser redirect required|
|APP-007|Refresh after 401|request retried once|
|APP-008|Secure Connect raw GET|correct base path|
|APP-009|Site admin raw POST requires CSRF|header present|
|APP-010|eVAULT raw route|correct base path|

### 13.12 Parser tests

|ID|Test|Expected|
|-|-|-|
|PARSE-001|JSON `$base: Real`|`CsmlValue::Real`|
|PARSE-002|JSON `$base: Unsigned`|`CsmlValue::Unsigned`|
|PARSE-003|JSON `$base: List`|list/map preserved|
|PARSE-004|JSON `$base: Struct`|struct map preserved|
|PARSE-005|JSON `$base: Choice`|choice preserved|
|PARSE-006|XML `<Real value="50"/>`|real|
|PARSE-007|XML `<Null value="NULL"/>`|null|
|PARSE-008|XML nested sequence|nested values|
|PARSE-009|Unknown XML element|unknown value preserved|
|PARSE-010|Truncated metadata|preserved|
|PARSE-011|`next` attribute|parsed pagination|
|PARSE-012|Malformed XML|parse error, no panic|
|PARSE-013|Malformed JSON|parse error, no panic|
|PARSE-014|Huge response within configured limit|succeeds or returns size error|
|PARSE-015|Entity expansion attack|rejected/no external entity resolution|

### 13.13 Property-based tests

|ID|Test|Strategy|
|-|-|-|
|PROP-001|URL query builder never creates two `?`|generate arbitrary params|
|PROP-002|URL path builder round-trips legal segments|generate legal site/object/property strings|
|PROP-003|XML escaping is reversible for attribute-safe values|generate strings with special chars|
|PROP-004|JSON serialize/deserialize stable for `CsmlValue`|generate CsmlValue tree|
|PROP-005|Priority accepts only 1..16|generate u8|
|PROP-006|Secrets never appear in `Debug` output|generate credentials|
|PROP-007|Pagination next URLs cannot change host by default|generate URLs|
|PROP-008|Redaction catches username/password/token keys|generate maps|

### 13.14 Fuzz tests

|ID|Target|Success condition|
|-|-|-|
|FUZZ-001|`parse\_csml\_json`|no panic, bounded memory|
|FUZZ-002|`parse\_csml\_xml`|no panic, no entity expansion|
|FUZZ-003|`parse\_object\_ref`|no panic; invalid refs return errors|
|FUZZ-004|`redact\_error\_body`|no panic; preserves UTF-8 replacement|
|FUZZ-005|`event\_query\_parser`|no panic|

### 13.15 Security tests

|ID|Test|Expected|
|-|-|-|
|SEC-001|Password not in logs|absent|
|SEC-002|Access token not in logs|absent|
|SEC-003|Refresh token not in logs|absent|
|SEC-004|Deprecated query login warns|warning event|
|SEC-005|Raw absolute URL to different host rejected|SSRF guard|
|SEC-006|Invalid certificate not accepted by default|fail|
|SEC-007|`danger\_accept\_invalid\_certs` name required|explicit config only|
|SEC-008|XML external entity disabled|safe|
|SEC-009|Path traversal-like segment encoded|no traversal|
|SEC-010|Write calls disabled by default|denied|

### 13.16 Live integration tests

Environment variables:

```bash
ENTELIWEB\_BASE\_URL=https://bms.example.edu/enteliweb
ENTELIWEB\_USERNAME=...
ENTELIWEB\_PASSWORD=...
ENTELIWEB\_SITE=MainSite
ENTELIWEB\_DEVICE=5600
ENTELIWEB\_SAFE\_READ\_REF=/.bacnet/MainSite/5600/analog-value,1/present-value
ENTELIWEB\_SAFE\_WRITE\_REF=/.bacnet/MainSite/5600/analog-value,999/present-value
ENTELIWEB\_ENABLE\_WRITES=false
ENTELIWEB\_ALLOW\_INVALID\_CERTS=false
```

Read-only live tests:

|ID|Test|Expected|
|-|-|-|
|LIVE-R-001|Basic login|succeeds|
|LIVE-R-002|Get `.info/software-version`|non-empty|
|LIVE-R-003|List sites|contains configured site or non-empty|
|LIVE-R-004|List devices|succeeds|
|LIVE-R-005|Read safe property|returns value or typed BACnet error|
|LIVE-R-006|Create `.multi` read for safe ref|succeeds|
|LIVE-R-007|Get systems|succeeds|
|LIVE-R-008|Get users|succeeds if permission allows; otherwise typed forbidden|
|LIVE-R-009|Get events|succeeds|
|LIVE-R-010|Get deployment info via app API if enabled|succeeds or unsupported|

Write live tests, skipped unless `ENTELIWEB\_ENABLE\_WRITES=true`:

|ID|Test|Expected|
|-|-|-|
|LIVE-W-001|Write safe value at priority 16|succeeds|
|LIVE-W-002|Read back written value|matches or server-formatted equivalent|
|LIVE-W-003|Relinquish priority|succeeds|
|LIVE-W-004|Audit log emitted|present|
|LIVE-W-005|Attempt non-allowlisted ref|denied client-side|

No live write test may run unless:

1. `unsafe-writes` feature is enabled.
2. `ENTELIWEB\_ENABLE\_WRITES=true`.
3. `ENTELIWEB\_SAFE\_WRITE\_REF` is set.
4. Runtime config `allow\_writes(true)` is set.

\---

## 14\. Acceptance criteria

### 14.1 v0.1 acceptance

* Compiles on stable Rust.
* `cargo test` passes without network access.
* Mock contract tests cover auth, `.info`, BACnet reads, `.multi` reads, systems, users, views, events, and alarm details.
* URL tests prove `/enteliweb/api` is built correctly.
* Query builder test proves `?priority=10\&alt=json`, never `?priority=10?alt=json`.
* Deprecated query login is unavailable unless explicitly feature-gated.
* Writes are unavailable or disabled by default.
* Secrets are redacted in logs/errors/snapshots.
* Live read-only integration tests can run with env vars.
* README includes safety warning for BAS/BMS writes.

### 14.2 v0.2 acceptance

* Unsafe write APIs implemented behind feature + runtime guard.
* `.multi` writes implemented.
* Event acknowledge implemented and verified against mock + live optional test.
* Alarm details update implemented.
* Trend log retrieval implemented.
* Parser supports common nested CSML structures.
* Fuzz targets run in CI on a time budget.

### 14.3 v0.3 acceptance

* Modern app API auth implemented.
* Token refresh implemented and tested.
* Deployment info/ecloud detection implemented.
* Raw app API route clients exist for site/log/Secure Connect/eVAULT areas.
* Browser OAuth paths are represented as redirect-required, not fake-authenticated.

\---

## 15\. Implementation sequence

### Phase 0 — Contract lock-down

1. Create endpoint inventory Markdown in the repository.
2. Add snapshot tests that represent all known URL templates.
3. Decide exact crate name and feature gates.
4. Build sanitized fixtures for protocol examples.

### Phase 1 — Transport and URL foundation

1. Implement `EnteliWebConfig`.
2. Implement base URL normalization.
3. Implement raw request layer.
4. Implement structured errors.
5. Implement redaction.
6. Implement mock-server test harness.

### Phase 2 — Legacy auth and safe reads

1. Implement legacy basic login.
2. Implement cookie jar and CSRF storage.
3. Implement `.info` and `.sysinfo` reads.
4. Implement object reference builder.
5. Implement BACnet read-property.
6. Implement list sites/devices/objects.
7. Implement systems/users/views/events/alarm details read APIs.

### Phase 3 — `.multi` and CSML parser

1. Implement JSON CSML model.
2. Implement XML CSML model.
3. Implement `.multi` read-create and get-values.
4. Add parser fixtures from protocol examples.
5. Add fuzz tests.

### Phase 4 — Controlled writes

1. Add `unsafe-writes` feature.
2. Add runtime guard and allowlist.
3. Implement BACnet write property.
4. Implement `.multi` write.
5. Implement event acknowledgement.
6. Implement alarm details update.
7. Add live write tests, skipped by default.

### Phase 5 — Modern app API

1. Implement `/index/checkuserauthenticationmethod`.
2. Implement `/v1/auth/login/`.
3. Implement token storage and refresh.
4. Implement modern CSRF handling.
5. Implement deployment info.
6. Add raw clients for site/admin/log/Secure Connect/eVAULT.

### Phase 6 — Hardening and release

1. Add CI matrix.
2. Add fuzz job.
3. Add README and examples.
4. Add semver policy.
5. Publish internal alpha.
6. Validate against a real enteliWEB instance.

\---

## 16\. Example README warning

```markdown
## Safety warning

This library can read and, when explicitly enabled, write values through an enteliWEB server.
Writing BACnet object properties may affect real building equipment.
All write APIs are disabled by default and require both a compile-time feature and runtime opt-in.
Do not enable write APIs against a production BAS/BMS without an approved test point, operator authorization, and rollback procedure.
```

\---

## 17\. Open questions requiring live-server validation

1. Does `/api/event/ack` require POST, PUT, or a server-specific method convention?
2. Does legacy `/api/auth/basiclogin` return `\_csrfToken` in body, header, cookie, or version-dependent combinations?
3. Which enteliWEB versions require CSRF on PUT/POST to `/api/...`?
4. Does `alt=xml` work consistently, or is XML simply the default when `alt=json` is absent?
5. How does enteliWEB encode site names and system names containing `/`, `+`, `#`, `%`, or non-ASCII characters?
6. Are absolute `next` URLs always same-origin?
7. Does enteliCLOUD expose the same legacy `/api/.bacnet` endpoints for all tenants?
8. Are `/v1/auth/login/` username/password fields always encoded the same way across supported versions?
9. Does modern app auth authorize legacy `/api/...` calls, or are these separate session realms?
10. What are the exact response bodies for permission-denied versus CSRF-denied versus BACnet object errors?

\---

## 18\. Final recommendation

Build the library as a careful Rust client for **enteliWEB legacy REST first**. The modern application API should become a separate module for `/v1` authentication and app/admin operations only after the legacy REST read path is correct.

The first serious milestone is not merely “make an API call.” It is:

```text
Login safely → build correct /enteliweb/api URLs → read BACnet values → parse CSML → prove no accidental writes → test every branch with mock contracts.
```

Only after that should write support, event acknowledgement, alarm updates, modern token auth, Secure Connect, logs, or eVAULT be added.

