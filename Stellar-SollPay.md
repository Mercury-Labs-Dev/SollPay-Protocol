# Account Abstraction (AA) Wallet on Soroban - Technical Requirements Document

## 1. Project Overview

### 1.1 Purpose
This document outlines the technical requirements for implementing an Account Abstraction (AA) wallet on the Stellar Soroban smart contract platform. The wallet enables users to control their assets without managing private keys, using Zero-Knowledge (ZK) proofs for identity verification.

### 1.2 Scope
- Core wallet functionality (init, update with ZK proof)
- Zero-Knowledge proof verification (Groth16)
- Asset management through vault accounts
- User identity verification via cryptographic proofs

## 2. Core Concepts

### 2.1 Account Abstraction
The wallet implements account abstraction by:
- Separating wallet identity (`hash`) from user identity (`user`)
- Using ZK proofs instead of traditional cryptographic signatures
- Allowing the vault to sign transactions on behalf of the wallet

### 2.2 Wallet Components
- **Hash**: 32-byte unique identifier for the wallet
- **Vault**: Asset storage account derived from the wallet hash
- **User**: Public key of the user verified through ZK proofs

## 3. Functional Requirements

### 3.1 Wallet Initialization
**Function**: `init_wallet(hash: BytesN<32>)`

**Description**: Initialize a new wallet with a unique 32-byte hash identifier.

**Requirements**:
- Create a new wallet instance with the provided hash
- Generate a vault account address deterministically from the hash
- Set initial state:
  - `vault`: Derived vault address
  - `user`: Empty/zero address (to be set via ZK proof)
  - `hash`: Provided hash value
  - `bump`: Bump seed for PDA generation (if applicable)

**Preconditions**:
- Hash must be unique (not already initialized)
- Caller must have authorization to create accounts

**Postconditions**:
- Wallet account is created and persisted
- Vault address is deterministic and can be computed from hash

### 3.2 Wallet Update with ZK Proof
**Function**: `update_wallet(
    hash: BytesN<32>,
    user: Address,
    project_id: u64,
    exp_time: u64,
    proof_a: BytesN<64>,
    proof_b: BytesN<128>,
    proof_c: BytesN<64>
)`

**Description**: Update wallet user identity by providing a valid ZK proof.

**Requirements**:
- Verify ZK proof validity (Groth16)
- Check proof expiration time
- Update wallet `user` field if proof is valid
- Emit event on successful update

**Preconditions**:
- Wallet must exist (initialized)
- Current timestamp must be less than `exp_time`
- ZK proof must be valid for the given parameters

**Validation**:
1. Verify proof hasn't expired: `current_time < exp_time`
2. Verify ZK proof using Groth16 verifier:
   - Public signals: `[hash, user[0:16], user[16:32], exp_time, project_id]`
   - Verify proof components: `proof_a`, `proof_b`, `proof_c`
   - Use appropriate verifying key based on `project_id`
3. Ensure wallet exists and is in valid state

**Postconditions**:
- Wallet `user` field is updated to provided address
- Event is emitted with hash and user address

## 4. Data Structures

### 4.1 Wallet State
```rust
pub struct Wallet {
    /// Bump seed for PDA generation (Soroban equivalent)
    pub bump: u8,
    
    /// Vault account address where assets are stored
    pub vault: Address,
    
    /// User address verified through ZK proof
    pub user: Address,
    
    /// 32-byte unique wallet identifier
    pub hash: BytesN<32>,
    
    /// Nonce for replay protection (optional)
    pub nonce: u64,
}
```

### 4.2 ZK Proof Parameters
```rust
pub struct ZKProof {
    /// Proof component A (64 bytes)
    pub proof_a: BytesN<64>,
    
    /// Proof component B (128 bytes)
    pub proof_b: BytesN<128>,
    
    /// Proof component C (64 bytes)
    pub proof_c: BytesN<64>,
}

pub struct ZKProofContext {
    /// Wallet hash identifier
    pub hash: BytesN<32>,
    
    /// User public key (32 bytes, split into two 16-byte segments)
    pub user: Address,
    
    /// Project identifier (SMS: 10006, Email: 10007)
    pub project_id: u64,
    
    /// Expiration timestamp
    pub exp_time: u64,
}
```

## 5. Zero-Knowledge Proof Integration

### 5.1 Groth16 Verification

**Verifying Keys**: The contract must support at least two verifying keys:
- `PROJECT_ID_SMS = 10006`: SMS-based verification
- `PROJECT_ID_EMAIL = 10007`: Email-based verification

**Public Signals Construction**:
The ZK proof public signals are constructed as follows:
```
public_signals = [
    hash,                    // 32 bytes: wallet identifier
    user_bytes[0..16],      // 16 bytes: first half of user address
    user_bytes[16..32],     // 16 bytes: second half of user address
    exp_time_bytes,         // 8 bytes: expiration timestamp (big-endian)
    project_id_bytes,       // 8 bytes: project ID (big-endian)
]
```

**Verification Process**:
1. Extract `project_id` from proof context
2. Select appropriate verifying key based on `project_id`
3. Construct public signals array (5 signals, each 32 bytes)
4. Initialize Groth16 verifier with proof components and public signals
5. Execute verification and return boolean result

### 5.2 Proof Validity Checks
1. **Expiration Check**: `current_timestamp < exp_time`
2. **Project ID Check**: `project_id` must be supported (SMS or Email)
3. **Proof Format Check**: Verify proof components have correct lengths
4. **Groth16 Verification**: Mathematical proof verification

## 6. Soroban-Specific Implementation Details

### 6.1 Account Storage
- Use Soroban's persistent storage for wallet state
- Wallet accounts keyed by hash (32 bytes)
- Consider using Soroban's data structures for efficient lookups

### 6.2 Vault Address Derivation
Since Soroban doesn't have PDAs like Solana, use one of:
- Deterministic address generation from hash + contract ID
- CREATE2-like mechanism if available
- Pre-computed vault addresses based on hash

**Suggested Approach**:
```
vault_address = sha256(contract_id || "vault_seed" || hash || bump)
```

### 6.3 Events
Emit events for:
- Wallet initialization
- Wallet user update

Event structure:
```rust
pub struct InitWalletEvent {
    pub hash: BytesN<32>,
}

pub struct UpdateWalletEvent {
    pub hash: BytesN<32>,
    pub user: Address,
}
```

## 7. API Specification

### 7.1 Contract Interface

```rust
pub trait WalletContract {
    /// Initialize a new wallet
    fn init_wallet(env: Env, hash: BytesN<32>) -> Result<(), Error>;
    
    /// Update wallet user with ZK proof
    fn update_wallet(
        env: Env,
        hash: BytesN<32>,
        user: Address,
        project_id: u64,
        exp_time: u64,
        proof_a: BytesN<64>,
        proof_b: BytesN<128>,
        proof_c: BytesN<64>,
    ) -> Result<(), Error>;
    
    /// Get wallet information
    fn get_wallet(env: Env, hash: BytesN<32>) -> Result<Wallet, Error>;
    
    /// Get vault address for a wallet hash
    fn get_vault_address(env: Env, hash: BytesN<32>) -> Result<Address, Error>;
}
```

### 7.2 Error Types

```rust
pub enum Error {
    /// Wallet already initialized
    WalletAlreadyExists,
    
    /// Wallet not found
    WalletNotFound,
    
    /// ZK proof verification failed
    InvalidProof,
    
    /// Proof has expired
    ProofExpired,
    
    /// Unsupported project ID
    UnsupportedProjectId,
    
    /// Invalid input parameters
    InvalidInput,
}
```

## 8. Security Considerations

### 8.1 ZK Proof Security
- Verify all proof components before processing
- Use constant-time comparison for cryptographic operations
- Validate all public signals are within expected ranges

### 8.2 Replay Protection
- Use `nonce` or `exp_time` to prevent replay attacks
- Consider adding additional checks for proof uniqueness

### 8.3 Access Control
- Wallet initialization: Should check caller authorization
- Wallet update: Must provide valid ZK proof
- Vault operations: Only wallet owner (proven user) can initiate

### 8.4 Input Validation
- Verify hash is 32 bytes
- Verify proof components have correct lengths
- Validate timestamps are reasonable (not in past, not too far future)
- Check project_id is supported

## 9. Integration Requirements

### 9.1 Groth16 Verifier Library
- Integrate Groth16 verification library compatible with Soroban
- Verify library supports WASM compilation
- Ensure verification keys can be embedded in contract

### 9.2 Verifying Keys
- Embed SMS and Email verifying keys in contract
- Keys should be stored as constants or in contract data
- Consider gas costs of key storage

## 10. Testing Requirements

### 10.1 Unit Tests
- Wallet initialization
- Wallet update with valid proof
- Wallet update with invalid proof
- Wallet update with expired proof
- Error handling for edge cases

### 10.2 Integration Tests
- End-to-end wallet flow
- ZK proof generation and verification
- Multiple wallets with same hash (should fail)
- Vault address derivation consistency

### 10.3 Security Tests
- Attempt to initialize wallet twice
- Attempt to update wallet with invalid proof
- Attempt to update wallet with expired proof
- Fuzzing of proof components

## 11. Performance Considerations

### 11.1 Gas Optimization
- Minimize storage operations
- Cache frequently accessed data
- Optimize ZK proof verification

### 11.2 Scalability
- Design for high transaction throughput
- Consider batch operations if needed
- Optimize storage structure for lookups

## 12. Implementation Phases

### Phase 1: Core Wallet Structure
- Define wallet data structure
- Implement wallet initialization
- Implement basic storage and retrieval

### Phase 2: ZK Proof Integration
- Integrate Groth16 verifier
- Implement proof verification logic
- Add supporting verification keys

### Phase 3: Wallet Update
- Implement `update_wallet` function
- Add proof validation
- Implement event emission

### Phase 4: Testing & Optimization
- Comprehensive testing
- Gas optimization
- Security audit

## 13. Dependencies

### 13.1 External Libraries
- Soroban SDK (latest stable version)
- Groth16 verification library (WASM compatible)
- Cryptographic primitives library

### 13.2 Development Tools
- Soroban CLI
- Soroban test framework
- Rust toolchain (latest stable)

## 14. Success Criteria

1. ✅ Wallet can be initialized with unique hash
2. ✅ Wallet user can be updated with valid ZK proof
3. ✅ Invalid proofs are rejected
4. ✅ Expired proofs are rejected
5. ✅ Vault address is deterministic and consistent
6. ✅ Events are emitted correctly
7. ✅ Contract passes all security tests
8. ✅ Gas costs are acceptable for production use

---

## Appendix A: Verifying Key Structure

The verifying keys should contain:
- `vk_alpha_g1`: 64 bytes
- `vk_beta_g2`: 128 bytes
- `vk_gamma_g2`: 128 bytes
- `vk_delta_g2`: 128 bytes
- `vk_ic`: Array of 6 public inputs (each 64 bytes)

## Appendix B: Reference Implementation

For reference, see the Solana implementation:
- Wallet initialization: `process_init_wallet()`
- Wallet update: `process_update_wallet()`
- ZK proof verification: `is_proof_valid()`

## Appendix C: Soroban Migration Notes

Key differences from Solana implementation:
1. No PDAs - use deterministic address generation
2. Different account model - use Soroban storage
3. Different program structure - WASM instead of BPF
4. Different event system - Soroban events
5. Different crypto primitives - ensure compatibility

