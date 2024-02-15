# RGP

[![ci](https://github.com//seanwatters/rgp/actions/workflows/ci.yml/badge.svg)](https://github.com//seanwatters/rgp/actions/workflows/ci.yml)
[![license](https://img.shields.io/github/license/seanwatters/rgp.svg)](https://github.com/seanwatters/rgp/blob/main/LICENSE)
[![crates.io](https://img.shields.io/crates/v/rgp.svg)](https://crates.io/crates/rgp)
[![docs.rs](https://docs.rs/rgp/badge.svg)](https://docs.rs/rgp/)
[![dependency status](https://deps.rs/repo/github/seanwatters/rgp/status.svg)](https://deps.rs/repo/github/seanwatters/rgp)

Relatively Good Privacy

## Ciphersuite

- Blake2s256 for hashing
- Ed25519 for signatures
- X25519 for shared secrets
- XChaCha20 for content keys
- XChaCha20Poly1305 for content

## Modes

There are currently three supported modes: `Dh` (Diffie-Hellman), `Hash`, and `Session`. All modes provide the ability to sign content and verify the sender. Deniability is preserved by signing the plaintext and encrypting alongside it.

### Diffie-Hellman

`Dh` mode provides forward secrecy as it generates a fresh/random **content key** for each message. The **content key** is then encrypted with each recipients' **shared secrets**. But with the added security of forward secrecy comes the computational and storage overhead of encrypting and attaching the encrypted **content key** for each recipient.

```rust
let (alice_fingerprint, alice_verifying_key) = rgp::generate_fingerprint();

let (alice_priv_key, alice_pub_key) = rgp::generate_dh_keys();
let (bob_priv_key, bob_pub_key) = rgp::generate_dh_keys();

let mut pub_keys = vec![bob_pub_key];

// 8mb
let content = vec![0u8; 8_000_000];

// add another 20,000 recipients
for _ in 0..20_000 {
    let (_, pub_key) = rgp::generate_dh_keys();
    pub_keys.push(pub_key)
}

// encrypt Alice's message for all recipients
let (mut encrypted_content, content_key) = rgp::encrypt(
    alice_fingerprint,
    content.clone(),
    rgp::EncryptMode::Dh(alice_priv_key, &pub_keys),
)
.unwrap();

// extract for Bob
rgp::extract_for_key_position_mut(0, &mut encrypted_content).unwrap();

// decrypt Alice's message for Bob
let (decrypted_content, decrypted_content_key) = rgp::decrypt(
    Some(&alice_verifying_key),
    &encrypted_content,
    rgp::DecryptMode::Dh(alice_pub_key, bob_priv_key),
)
.unwrap();

assert_eq!(decrypted_content, content);
assert_eq!(decrypted_content_key, content_key);
```

**Internal Process:**

1. Generate one-time components
    - **nonce**
    - **content key**
2. Sign plaintext to generate **content signature**
3. Encrypt plaintext and **content signature** with **content key**
4. Encrypt **content key** for all recipients
    - Generate **shared secret** with **recipient public key** and **sender private key**
    - Encrypt **content key** with **shared secret**

### Hash

`Hash` mode, while it doesn't provide the same level of security as random key generation + per-recipient key encryption, it does enable backward secrecy, and when used with a non-constant hash key/value, it can enable forward secrecy when only the **content key** is compromised.

```rust
let (alice_fingerprint, alice_verifying_key) = rgp::generate_fingerprint();

let hash_key = [0u8; 32]; // use an actual key
let hash_value = [1u8; 32]; // use an actual key

let content = vec![0u8; 8_000_000];

// encrypt Alice's message in `Hash` mode
let (mut encrypted_content, content_key) = rgp::encrypt(
    alice_fingerprint,
    content.clone(),
    rgp::EncryptMode::Hash(hash_key, hash_value),
)
.unwrap();

// decrypt Alice's message in `Hash` mode
let (decrypted_content, hashed_content_key) = rgp::decrypt(
    Some(&alice_verifying_key),
    &encrypted_content,
    rgp::DecryptMode::Hash(hash_key, hash_value),
)
.unwrap();

assert_eq!(decrypted_content, content);
assert_eq!(hashed_content_key, content_key);
```

**Internal Process:**

1. Generate **nonce**
2. Hash the content key
3. Sign plaintext to generate **content signature**
4. Encrypt plaintext and **content signature** with the hashed **content key**

### Session

This mode provides no forward or backward secrecy, and uses the provided key "as is" without any modification. From an encryption perspective, this is essentially the same as just running the underlying symmetric cipher.

```rust
let (alice_fingerprint, alice_verifying_key) = rgp::generate_fingerprint();

let session_key = [0u8; 32]; // use an actual key
let content = vec![0u8; 8_000_000];

// encrypt Alice's message with a session key
let (mut encrypted_content, _) = rgp::encrypt(
    alice_fingerprint,
    content.clone(),
    rgp::EncryptMode::Session(session_key),
)
.unwrap();

// decrypt Alice's message with session key
let (decrypted_content, _) = rgp::decrypt(
    Some(&alice_verifying_key),
    &encrypted_content,
    rgp::DecryptMode::Session(session_key),
)
.unwrap();

assert_eq!(decrypted_content, content);
```

**Internal Process:**

1. Generate **nonce**
2. Sign plaintext to generate **content signature**
3. Encrypt plaintext and **content signature** with the provided **content key**, as is

## Encrypted Format

- **nonce** = 24 bytes
- keys count (`Dh` mode only)
    - int size = 2 bits
    - count
        - numbers 0-63 = 6 bits
        - numbers >63 = 1-8 bytes (big endian int)
- encrypted copies of **content key** (`Dh` mode only) = pub_keys.len() * 32 bytes
- encrypted content = content.len()
- **signature** = 64 bytes (encrypted along with the content to preserve deniability)
- Poly1305 MAC = 16 bytes
- mode = 1 byte (0 for `Session` | 1 for `Hash` | 2 for `Dh`)

## Performance

To check performance on your machine, run `cargo bench`. You can also view the latest benches in the GitHub CI [workflow](https://github.com//seanwatters/rgp/actions/workflows/ci.yml) under job/Benchmark.

All benchmarks for multi-recipient payloads are for **20,000** recipients, and all benchmarks encrypt/sign/decrypt **8mb**.

## License

[MIT](https://opensource.org/license/MIT)

## Security

THIS CODE HAS NOT BEEN AUDITED OR REVIEWED. USE AT YOUR OWN RISK.
