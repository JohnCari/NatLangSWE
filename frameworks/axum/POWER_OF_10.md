# The Power of 10 Rules for Safety-Critical Axum Code

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** Axum uses `#![forbid(unsafe_code)]` — the entire framework is implemented in 100% safe Rust.

---

## The Original 10 Rules (C Language)

| # | Rule |
|---|------|
| 1 | Restrict all code to very simple control flow constructs—no `goto`, `setjmp`, `longjmp`, or recursion |
| 2 | Give all loops a fixed upper bound provable by static analysis |
| 3 | Do not use dynamic memory allocation after initialization |
| 4 | No function longer than one printed page (~60 lines) |
| 5 | Assertion density: minimum 2 assertions per function |
| 6 | Declare all data objects at the smallest possible scope |
| 7 | Check all return values and validate all function parameters |
| 8 | Limit preprocessor to header includes and simple macros |
| 9 | Restrict pointers: single dereference only, no function pointers |
| 10 | Compile with all warnings enabled; use static analyzers daily |

---

## The Power of 10 Rules — Axum Edition

### Rule 1: Simple Control Flow — Early Returns with `?`

**Original Intent:** Eliminate complex control flow that impedes static analysis.

**Axum Adaptation:**

```rust
// BAD: Deeply nested handler
async fn create_user(
    State(state): State<AppState>,
    Json(input): Json<CreateUserInput>,
) -> impl IntoResponse {
    if let Ok(validated) = validate_input(&input) {
        if let Ok(user) = state.db.create_user(validated).await {
            if let Ok(token) = create_token(&user) {
                return (StatusCode::CREATED, Json(UserResponse { user, token }));
            }
        }
    }
    StatusCode::INTERNAL_SERVER_ERROR
}

// GOOD: Guard clauses with early returns using ?
async fn create_user(
    State(state): State<AppState>,
    Json(input): Json<CreateUserInput>,
) -> Result<impl IntoResponse, AppError> {
    let validated = validate_input(&input)?;
    let user = state.db.create_user(validated).await?;
    let token = create_token(&user)?;

    Ok((StatusCode::CREATED, Json(UserResponse { user, token })))
}

// GOOD: Guard clause for authorization
async fn get_user(
    State(state): State<AppState>,
    auth: AuthUser,
    Path(user_id): Path<Uuid>,
) -> Result<Json<User>, AppError> {
    if !auth.can_view_user(user_id) {
        return Err(AppError::Forbidden);
    }

    let user = state.db.get_user(user_id).await?;
    Ok(Json(user))
}
```

**Guidelines:**
- Use `?` operator for early returns
- Use guard clauses at the start of handlers
- Maximum 3-4 levels of nesting
- No recursion in handlers or services

---

### Rule 2: Bounded Loops — Paginate and Limit

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**Axum Adaptation:**

```rust
// BAD: Unbounded database query
async fn list_users(
    State(state): State<AppState>,
) -> Result<Json<Vec<User>>, AppError> {
    let users = state.db.get_all_users().await?;  // Could be millions
    Ok(Json(users))
}

// BAD: Unbounded request body
async fn upload(
    body: Bytes,  // No size limit!
) -> impl IntoResponse {
    // Process unlimited data
}

// GOOD: Paginated query with limits
const MAX_PAGE_SIZE: i64 = 100;
const DEFAULT_PAGE_SIZE: i64 = 20;

#[derive(Deserialize)]
struct Pagination {
    #[serde(default)]
    page: i64,
    #[serde(default = "default_page_size")]
    page_size: i64,
}

fn default_page_size() -> i64 { DEFAULT_PAGE_SIZE }

async fn list_users(
    State(state): State<AppState>,
    Query(pagination): Query<Pagination>,
) -> Result<Json<PaginatedResponse<User>>, AppError> {
    let page_size = pagination.page_size.min(MAX_PAGE_SIZE).max(1);
    let offset = pagination.page.max(0) * page_size;

    let users = state.db
        .get_users_paginated(page_size, offset)
        .await?;

    Ok(Json(PaginatedResponse {
        data: users,
        page: pagination.page,
        page_size,
    }))
}

// GOOD: Limited request body size
use axum::extract::DefaultBodyLimit;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::max(10 * 1024 * 1024));  // 10MB limit

// GOOD: Bounded iteration
const MAX_BATCH_SIZE: usize = 1000;

async fn process_batch(items: Vec<Item>) -> Result<(), AppError> {
    debug_assert!(items.len() <= MAX_BATCH_SIZE);

    for item in items.into_iter().take(MAX_BATCH_SIZE) {
        process_item(item).await?;
    }
    Ok(())
}
```

**Guidelines:**
- Always paginate database queries
- Set `DefaultBodyLimit` on all routes accepting bodies
- Use `.take(N)` on iterators
- Define bounds as constants
- Assert collection sizes

---

### Rule 3: Controlled Memory — Arc State, Stream Responses

**Original Intent:** Prevent unbounded memory growth.

**Axum Adaptation:**

```rust
// BAD: Cloning large state on every request
#[derive(Clone)]
struct AppState {
    large_cache: HashMap<String, LargeData>,  // Cloned per request!
}

// BAD: Loading entire file into memory
async fn download(Path(id): Path<String>) -> impl IntoResponse {
    let contents = tokio::fs::read(format!("files/{id}")).await?;
    contents  // Entire file in memory
}

// GOOD: Shared state with Arc
use std::sync::Arc;

struct AppState {
    db: PgPool,
    cache: Arc<DashMap<String, CachedData>>,
}

// State is wrapped in Arc automatically by Axum
async fn handler(State(state): State<Arc<AppState>>) -> impl IntoResponse {
    // state is a reference, no cloning of inner data
}

// GOOD: Stream large responses
use axum::body::Body;
use tokio_util::io::ReaderStream;

async fn download(Path(id): Path<String>) -> Result<Body, AppError> {
    let file = tokio::fs::File::open(format!("files/{id}")).await?;
    let stream = ReaderStream::new(file);
    Ok(Body::from_stream(stream))
}

// GOOD: Avoid cloning large data in extractors
async fn handler(
    State(state): State<Arc<AppState>>,
    // Use references where possible
) -> impl IntoResponse {
    let data = state.cache.get(&key);  // Returns reference
    // Process without cloning
}
```

**Guidelines:**
- Wrap `AppState` in `Arc` (Axum does this automatically)
- Use `Arc<DashMap>` or similar for shared caches
- Stream large file responses
- Avoid `.clone()` on large data structures
- Use connection pools (`PgPool`, `RedisPool`)

---

### Rule 4: Short Handlers — Extract to Services

**Original Intent:** Ensure functions are small enough to understand and verify.

**Axum Adaptation:**

```rust
// BAD: Monolithic handler
async fn create_order(
    State(state): State<AppState>,
    auth: AuthUser,
    Json(input): Json<CreateOrderInput>,
) -> Result<Json<Order>, AppError> {
    // 200 lines of validation, business logic, database calls...
}

// GOOD: Handler delegates to service layer
async fn create_order(
    State(state): State<AppState>,
    auth: AuthUser,
    Json(input): Json<CreateOrderInput>,
) -> Result<Json<Order>, AppError> {
    let order = state.order_service
        .create_order(auth.user_id, input)
        .await?;

    Ok(Json(order))
}

// Service layer (separate module)
impl OrderService {
    pub async fn create_order(
        &self,
        user_id: Uuid,
        input: CreateOrderInput,
    ) -> Result<Order, AppError> {
        self.validate_order(&input)?;
        let order = self.repository.insert_order(user_id, input).await?;
        self.notify_order_created(&order).await?;
        Ok(order)
    }

    fn validate_order(&self, input: &CreateOrderInput) -> Result<(), AppError> {
        // ≤60 lines
    }
}

// GOOD: Use extractors to simplify handlers
// Custom extractor handles validation
struct ValidatedOrder(CreateOrderInput);

#[async_trait]
impl<S> FromRequest<S> for ValidatedOrder
where
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(input) = Json::<CreateOrderInput>::from_request(req, state)
            .await
            .map_err(|_| AppError::InvalidInput)?;

        validate_order_input(&input)?;
        Ok(ValidatedOrder(input))
    }
}

// Now handler is simple
async fn create_order(
    State(state): State<AppState>,
    auth: AuthUser,
    ValidatedOrder(input): ValidatedOrder,
) -> Result<Json<Order>, AppError> {
    let order = state.order_service.create_order(auth.user_id, input).await?;
    Ok(Json(order))
}
```

**Guidelines:**
- Maximum 60 lines per handler
- Extract business logic to service layer
- Use custom extractors for validation
- One file per handler group (users, orders, etc.)
- Keep handlers focused on HTTP concerns

---

### Rule 5: Validation — Typed Extractors, Assert Invariants

**Original Intent:** Defensive programming catches bugs early.

**Axum Adaptation:**

```rust
use validator::Validate;

// Define validated input types
#[derive(Deserialize, Validate)]
struct CreateUserInput {
    #[validate(email)]
    email: String,
    #[validate(length(min = 8, max = 100))]
    password: String,
    #[validate(length(min = 1, max = 50))]
    name: String,
}

// GOOD: Validate in extractor
struct ValidatedJson<T>(T);

#[async_trait]
impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    S: Send + Sync,
    T: DeserializeOwned + Validate,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|e| AppError::InvalidJson(e.to_string()))?;

        value.validate()
            .map_err(|e| AppError::ValidationError(e.to_string()))?;

        Ok(ValidatedJson(value))
    }
}

// GOOD: Validate path parameters
async fn get_user(
    Path(user_id): Path<Uuid>,  // Uuid parsing validates format
) -> Result<Json<User>, AppError> {
    debug_assert!(!user_id.is_nil(), "User ID should not be nil");
    // ...
}

// GOOD: Assert invariants in services
impl UserService {
    pub async fn transfer_funds(
        &self,
        from: Uuid,
        to: Uuid,
        amount: Decimal,
    ) -> Result<(), AppError> {
        // Preconditions
        debug_assert!(amount > Decimal::ZERO);
        debug_assert!(from != to, "Cannot transfer to same account");

        let from_balance = self.get_balance(from).await?;
        if from_balance < amount {
            return Err(AppError::InsufficientFunds);
        }

        self.repository.transfer(from, to, amount).await?;

        // Postcondition
        let new_balance = self.get_balance(from).await?;
        debug_assert!(new_balance >= Decimal::ZERO);

        Ok(())
    }
}
```

**Guidelines:**
- Use `validator` crate for input validation
- Create `ValidatedJson<T>` extractor
- Use strong types (Uuid, Email, etc.) in paths
- Assert preconditions and postconditions
- Validate at trust boundaries (HTTP layer)

---

### Rule 6: Minimal Scope — Narrow Middleware, Sub-States

**Original Intent:** Reduce state complexity and potential for misuse.

**Axum Adaptation:**

```rust
// BAD: Global middleware for specific routes
let app = Router::new()
    .route("/public", get(public_handler))
    .route("/admin", get(admin_handler))
    .layer(RequireAuth);  // Auth required everywhere!

// BAD: Monolithic state
struct AppState {
    db: PgPool,
    redis: RedisPool,
    s3: S3Client,
    stripe: StripeClient,
    sendgrid: SendGridClient,
    // Every handler gets everything
}

// GOOD: Scoped middleware
let public_routes = Router::new()
    .route("/health", get(health))
    .route("/login", post(login));

let protected_routes = Router::new()
    .route("/users", get(list_users))
    .route("/users/:id", get(get_user))
    .layer(RequireAuth);

let admin_routes = Router::new()
    .route("/admin/users", get(admin_list_users))
    .layer(RequireAdmin);

let app = Router::new()
    .merge(public_routes)
    .merge(protected_routes)
    .merge(admin_routes);

// GOOD: Sub-states with FromRef
#[derive(Clone)]
struct AppState {
    db: PgPool,
    auth: AuthState,
    payments: PaymentState,
}

#[derive(Clone)]
struct AuthState {
    jwt_secret: String,
    session_store: Arc<SessionStore>,
}

impl FromRef<AppState> for AuthState {
    fn from_ref(state: &AppState) -> Self {
        state.auth.clone()
    }
}

// Handler only receives what it needs
async fn login(
    State(auth): State<AuthState>,  // Only auth state
    Json(creds): Json<Credentials>,
) -> Result<Json<Token>, AppError> {
    // ...
}
```

**Guidelines:**
- Apply middleware only where needed
- Use `Router::merge` to compose route groups
- Use `FromRef` for sub-states
- Handlers should only access required state
- Avoid global mutable state (use `Arc<RwLock>` sparingly)

---

### Rule 7: Check Returns — Typed Error Responses

**Original Intent:** Never ignore errors; verify at trust boundaries.

**Axum Adaptation:**

```rust
// BAD: Using unwrap
async fn get_user(Path(id): Path<String>) -> Json<User> {
    let uuid = Uuid::parse_str(&id).unwrap();  // Panic!
    let user = db.get_user(uuid).await.unwrap();  // Panic!
    Json(user)
}

// BAD: Ignoring errors
async fn save_log(Json(log): Json<LogEntry>) -> StatusCode {
    let _ = db.insert_log(log).await;  // Error ignored!
    StatusCode::OK
}

// GOOD: Comprehensive error handling with typed errors
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("Not found")]
    NotFound,
    #[error("Unauthorized")]
    Unauthorized,
    #[error("Forbidden")]
    Forbidden,
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    #[error("Database error")]
    Database(#[from] sqlx::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "Unauthorized"),
            AppError::Forbidden => (StatusCode::FORBIDDEN, "Forbidden"),
            AppError::InvalidInput(msg) => (StatusCode::BAD_REQUEST, msg.as_str()),
            AppError::Database(_) => {
                tracing::error!("Database error: {:?}", self);
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal error")
            }
        };

        (status, Json(json!({ "error": message }))).into_response()
    }
}

// GOOD: All paths return Result
async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, AppError> {
    let user = state.db
        .get_user(id)
        .await?
        .ok_or(AppError::NotFound)?;

    Ok(Json(user))
}
```

**Guidelines:**
- Never use `.unwrap()` in handlers
- Define custom `AppError` implementing `IntoResponse`
- Use `?` operator throughout
- Log internal errors, return safe messages to clients
- Handle `Option` with `.ok_or(AppError::NotFound)?`

---

### Rule 8: Limit Metaprogramming — Prefer `from_fn`

**Original Intent:** Avoid constructs that create unanalyzable code.

**Axum Adaptation:**

```rust
// BAD: Complex custom Tower service
struct MyMiddleware<S> {
    inner: S,
    config: Config,
}

impl<S, ReqBody, ResBody> Service<Request<ReqBody>> for MyMiddleware<S>
where
    S: Service<Request<ReqBody>, Response = Response<ResBody>>,
    // ... many trait bounds ...
{
    // Complex implementation
}

// BAD: Excessive macro usage
macro_rules! define_routes {
    ($($method:ident $path:literal => $handler:ident),*) => {
        Router::new()
            $(.route($path, $method($handler)))*
    };
}

// GOOD: Simple middleware with from_fn
use axum::middleware::{self, Next};

async fn auth_middleware(
    State(state): State<AppState>,
    request: Request,
    next: Next,
) -> Result<Response, AppError> {
    let token = request
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(AppError::Unauthorized)?;

    let user = validate_token(&state, token).await?;

    let mut request = request;
    request.extensions_mut().insert(user);

    Ok(next.run(request).await)
}

let app = Router::new()
    .route("/protected", get(handler))
    .layer(middleware::from_fn_with_state(state.clone(), auth_middleware));

// GOOD: Use from_extractor for extractor-based middleware
use axum::middleware::from_extractor;

let app = Router::new()
    .route("/protected", get(handler))
    .layer(from_extractor::<AuthUser>());

// GOOD: Explicit route definitions
let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/:id", get(get_user).put(update_user).delete(delete_user));
```

**Guidelines:**
- Use `middleware::from_fn` for simple middleware
- Use `middleware::from_extractor` when you have an extractor
- Write custom `Service` only for reusable/published middleware
- Keep macro usage simple and transparent
- Prefer explicit route definitions over macros

---

### Rule 9: Type Safety — No `unwrap()`, Typed Extractors

**Original Intent:** (C: Restrict pointer usage for safety)

**Axum Adaptation:**

```rust
// BAD: Stringly-typed IDs
async fn get_user(Path(id): Path<String>) -> Result<Json<User>, AppError> {
    let uuid = Uuid::parse_str(&id)?;  // Could fail
    // ...
}

// BAD: Any-typed JSON
async fn handler(Json(data): Json<serde_json::Value>) -> impl IntoResponse {
    let name = data["name"].as_str().unwrap();  // Runtime error
}

// GOOD: Strongly-typed path parameters
async fn get_user(Path(id): Path<Uuid>) -> Result<Json<User>, AppError> {
    // id is already a valid Uuid
}

// GOOD: Typed request bodies
#[derive(Deserialize)]
struct CreateUserRequest {
    email: String,
    name: String,
}

async fn create_user(
    Json(request): Json<CreateUserRequest>,
) -> Result<Json<User>, AppError> {
    // request.email and request.name are guaranteed to exist
}

// GOOD: Newtype wrappers for domain types
#[derive(Debug, Clone, Deserialize, Serialize)]
struct UserId(Uuid);

#[derive(Debug, Clone, Deserialize, Serialize)]
struct Email(String);

impl Email {
    pub fn new(value: String) -> Result<Self, AppError> {
        if value.contains('@') {
            Ok(Self(value))
        } else {
            Err(AppError::InvalidInput("Invalid email".into()))
        }
    }
}

// GOOD: Typed extractors with explicit rejections
async fn handler(
    result: Result<Json<Input>, JsonRejection>,
) -> Result<Json<Output>, AppError> {
    let Json(input) = result.map_err(|e| AppError::InvalidJson(e.to_string()))?;
    // ...
}
```

**Guidelines:**
- Use `Uuid` in paths, not `String`
- Define typed request/response structs
- Use newtype wrappers for domain types
- Handle extractor rejections explicitly
- Never use `serde_json::Value` for known structures

---

### Rule 10: Static Analysis — Clippy Pedantic, Zero Warnings

**Original Intent:** Catch issues at development time.

**Axum Adaptation:**

```rust
// Crate-level lints
#![forbid(unsafe_code)]
#![deny(warnings)]
#![deny(clippy::all)]
#![deny(clippy::pedantic)]
#![deny(clippy::nursery)]

// Specific denials for web services
#![deny(clippy::unwrap_used)]
#![deny(clippy::expect_used)]
#![deny(clippy::panic)]
#![deny(clippy::todo)]
#![deny(clippy::unimplemented)]
```

**Cargo.toml:**
```toml
[lints.rust]
unsafe_code = "forbid"

[lints.clippy]
all = "deny"
pedantic = "deny"
nursery = "warn"
unwrap_used = "deny"
expect_used = "deny"
```

**CI Pipeline:**
```bash
# Required checks
cargo fmt --check
cargo clippy -- -D warnings
cargo test
cargo audit                  # Security vulnerabilities
cargo deny check             # License & dependency audit
```

**Security Headers Middleware:**
```rust
use tower_http::set_header::SetResponseHeaderLayer;
use http::header::{CONTENT_TYPE, X_CONTENT_TYPE_OPTIONS, X_FRAME_OPTIONS};

let app = Router::new()
    .route("/", get(handler))
    .layer(SetResponseHeaderLayer::if_not_present(
        X_CONTENT_TYPE_OPTIONS,
        HeaderValue::from_static("nosniff"),
    ))
    .layer(SetResponseHeaderLayer::if_not_present(
        X_FRAME_OPTIONS,
        HeaderValue::from_static("DENY"),
    ));
```

**Guidelines:**
- Use `#![forbid(unsafe_code)]`
- Enable Clippy pedantic lints
- Zero warnings policy
- Run `cargo audit` regularly
- Configure security headers
- Use `tower-http` for common security middleware

---

## Summary: Axum Adaptation

| # | Original Rule | Axum Guideline |
|---|---------------|----------------|
| 1 | No goto/recursion | Use `?` for early returns, guard clauses |
| 2 | Fixed loop bounds | Paginate queries, limit body size |
| 3 | No dynamic allocation | `Arc<State>`, stream responses |
| 4 | 60-line functions | 60-line handlers, extract to services |
| 5 | 2+ assertions/function | Validate extractors, assert invariants |
| 6 | Minimize scope | Scoped middleware, sub-states |
| 7 | Check returns | Typed `AppError`, no `unwrap()` |
| 8 | Limit preprocessor | Use `from_fn`, limit macros |
| 9 | Restrict pointers | Typed extractors, newtype wrappers |
| 10 | All warnings enabled | Clippy pedantic, `#![forbid(unsafe_code)]` |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Axum Documentation](https://docs.rs/axum/latest/axum/)
- [Axum GitHub](https://github.com/tokio-rs/axum)
- [Shuttle Axum Guide](https://www.shuttle.dev/blog/2023/12/06/using-axum-rust)
- [Tower Middleware](https://docs.rs/tower/latest/tower/)
