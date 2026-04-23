# SDK Configuration

## Overview

The SDK Configuration module provides a type-safe way to configure AnchorKit SDK client connections with network settings, anchor domains, timeouts, and custom HTTP headers.

## Data Structures

### SdkConfig

Main configuration structure for SDK clients.

```rust
pub struct SdkConfig {
    pub network: NetworkType,
    pub anchor_domain: String,
    pub timeout_seconds: u64,
    pub custom_headers: Vec<HttpHeader>,
}
```

### NetworkType

Enum for Stellar network selection.

```rust
pub enum NetworkType {
    Testnet = 1,
    Mainnet = 2,
}
```

### HttpHeader

Custom HTTP header for API requests.

```rust
pub struct HttpHeader {
    pub key: String,
    pub value: String,
}
```

## Validation Rules

The `SdkConfig::validate()` method enforces the following constraints:

### Anchor Domain
- **Minimum length**: 3 characters
- **Maximum length**: 253 characters
- **Format**: Valid domain name

### Timeout
- **Minimum**: 1 second
- **Maximum**: 300 seconds (5 minutes)
- **Default**: 30 seconds (recommended)

### Custom Headers
- **Maximum count**: 20 headers
- **Header key length**: 1-64 characters
- **Header value length**: 0-1024 characters

## Usage Example

### Rust (Contract)

```rust
use soroban_sdk::{Env, String, Vec};

let env = Env::default();

// Create headers
let mut headers = Vec::new(&env);
headers.push_back(HttpHeader {
    key: String::from_str(&env, "Authorization"),
    value: String::from_str(&env, "Bearer token123"),
});

// Create config
let config = SdkConfig {
    network: NetworkType::Testnet,
    anchor_domain: String::from_str(&env, "anchor.example.com"),
    timeout_seconds: 30,
    custom_headers: headers,
};

// Validate
if config.validate() {
    // Use config
}
```

### JavaScript (Client SDK)

```javascript
const config = {
    network: 'Testnet',
    anchor_domain: 'anchor.example.com',
    timeout_seconds: 30,
    custom_headers: [
        {
            key: 'Authorization',
            value: 'Bearer token123'
        },
        {
            key: 'X-Custom-Header',
            value: 'custom-value'
        }
    ]
};
```

## Configuration Form

An HTML form is provided in `sdk_config_form.html` for easy configuration generation. The form includes:

- Network selection (Testnet/Mainnet)
- Anchor domain input with validation
- Timeout configuration
- Dynamic custom header management
- JSON output generation

### Using the Form

1. Open `sdk_config_form.html` in a web browser
2. Select your network (Testnet or Mainnet)
3. Enter the anchor domain
4. Set the timeout (default: 30 seconds)
5. Add custom headers as needed
6. Click "Generate Configuration" to get JSON output

## Security Considerations

### Header Security
- Never include sensitive credentials directly in headers
- Use secure credential management (see `SECURE_CREDENTIALS.md`)
- Rotate tokens regularly
- Use HTTPS for all anchor communications

### Domain Validation
- Validate anchor domains against a whitelist
- Use DNS verification for production
- Implement certificate pinning for critical operations

### Timeout Configuration
- Set appropriate timeouts based on network conditions
- Consider retry logic for transient failures
- Monitor timeout rates for performance tuning

## Best Practices

### Network Selection
- Use **Testnet** for development and testing
- Use **Mainnet** only for production deployments
- Never mix testnet and mainnet configurations

### Timeout Settings
- **Development**: 60-120 seconds (for debugging)
- **Production**: 30 seconds (recommended)
- **High-latency networks**: 60-90 seconds
- **Low-latency networks**: 15-30 seconds

### Custom Headers
- Use headers for:
  - Authentication tokens
  - API versioning
  - Request tracing
  - Custom metadata
- Avoid headers for:
  - Large payloads (use request body)
  - Sensitive data without encryption
  - Unnecessary metadata

## Integration with AnchorKit

The SDK configuration integrates with:

- **Session Management**: Timeout settings affect session duration
- **Credential Management**: Headers can include auth tokens
- **Health Monitoring**: Timeout affects health check intervals
- **Rate Comparison**: Network selection determines available anchors

## Feature Flags

### `mock-only`

The `mock-only` feature flag is defined in `Cargo.toml` and is intended exclusively for test environments. It signals that the SDK is running in a fully mocked context where real network calls, SEP-10 JWT verification, and on-chain authentication are not required.

#### Purpose

In normal operation, functions such as `register_attestor`, `verify_sep10_token`, and `set_sep10_jwt_verifying_key` require a valid SEP-10 JWT signed by a trusted anchor key. This is the correct behaviour for production and integration testing.

When writing unit tests that focus on contract logic rather than authentication flows, setting up a real SEP-10 token chain adds friction. The `mock-only` flag marks a build as test-only so that mock helpers (e.g. `env.mock_all_auths()`) can be used freely and any future mock-specific code paths can be compiled in without affecting production builds.

#### Affected Functions

The following contract functions are most relevant when working under `mock-only`:

| Function | Normal behaviour | Under `mock-only` |
|---|---|---|
| `register_attestor` | Requires a valid SEP-10 JWT and a registered verifying key | Call directly on `AnchorKitContract` (not via client) with `mock_all_auths()`; the JWT check is bypassed in the test harness |
| `verify_sep10_token` | Validates JWT signature and expiry against stored keys | Not called in mock mode; omit from test setup |
| `set_sep10_jwt_verifying_key` | Admin-only; stores Ed25519 public key on-chain | Not required in mock mode; skip entirely |
| `submit_attestation` | Requires `issuer.require_auth()` and a registered attestor | `require_auth()` satisfied by `mock_all_auths()`; register attestor first |
| `submit_with_request_id` | Same as `submit_attestation` | Same as above |

> **Important:** `mock_all_auths()` only bypasses `require_auth()` calls. It does **not** bypass the SEP-10 JWT signature verification inside `register_attestor`. To skip JWT verification in tests, call `AnchorKitContract::register_attestor` directly (struct method, not via the generated client) — the Soroban test harness does not enforce JWT checks on direct struct calls. For tests that must validate real SEP-10 flows, use `sep10_test_util::register_attestor_with_sep10` instead.

#### Enabling `mock-only` in Tests

Add the feature flag to your `Cargo.toml` and pass it on the command line:

```toml
# Cargo.toml
[features]
mock-only = []
```

Run tests with the flag enabled:

```bash
cargo test --features mock-only
```

#### Example: Unit Test Using Mock Mode

```rust
#[cfg(test)]
mod tests {
    use soroban_sdk::{testutils::Address as _, Address, Env, String};
    use anchorkit::contract::AnchorKitContract;

    fn make_env() -> Env {
        let env = Env::default();
        // mock_all_auths() satisfies every require_auth() call in the contract.
        env.mock_all_auths();
        env
    }

    #[test]
    #[cfg(feature = "mock-only")]
    fn test_register_attestor_mock() {
        let env = make_env();

        let admin = Address::generate(&env);
        let attestor = Address::generate(&env);
        let sep10_issuer = Address::generate(&env);

        // Initialize contract state directly via struct method.
        AnchorKitContract::initialize(env.clone(), admin);

        // Call register_attestor directly on the struct — the Soroban test
        // harness skips JWT verification on direct struct calls, so a
        // placeholder token is sufficient in mock-only mode.
        let placeholder_token = String::from_str(&env, "mock.jwt.token");
        AnchorKitContract::register_attestor(
            env.clone(),
            attestor.clone(),
            placeholder_token,
            sep10_issuer,
        );

        assert!(AnchorKitContract::is_attestor(env, attestor));
    }
}
```

> **Note:** Never enable `mock-only` in production builds. It is strictly a test-time convenience. For integration tests that must validate real SEP-10 flows, use the `sep10_test_util` helpers (`build_sep10_jwt` / `register_attestor_with_sep10`) instead.

## Testing

Run the SDK configuration tests:

```bash
cargo test sdk_config_tests --lib
```

Run tests with mock mode enabled:

```bash
cargo test --features mock-only
```

Test coverage includes:
- Valid configuration validation
- Domain length constraints
- Timeout boundary conditions
- Header count limits
- Header size constraints
- Network type enum values

## Error Handling

Configuration validation returns a boolean. For detailed error handling, check specific constraints:

```rust
if !config.validate() {
    // Check individual constraints
    if config.anchor_domain.len() < 3 {
        // Handle domain too short
    }
    if config.timeout_seconds < 1 || config.timeout_seconds > 300 {
        // Handle invalid timeout
    }
    if config.custom_headers.len() > 20 {
        // Handle too many headers
    }
}
```

## Future Enhancements

Potential improvements:
- Add retry configuration
- Support for connection pooling settings
- Circuit breaker configuration
- Rate limiting settings
- Custom DNS resolver configuration
- Proxy support

## Related Documentation

- [SECURE_CREDENTIALS.md](./SECURE_CREDENTIALS.md) - Credential management
- [HEALTH_MONITORING.md](./HEALTH_MONITORING.md) - Health check configuration
- [API_SPEC.md](./API_SPEC.md) - API specifications
- [QUICK_START.md](./QUICK_START.md) - Getting started guide
