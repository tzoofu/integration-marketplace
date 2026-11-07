# Secrets at Rest with AES-256-GCM (row-bound AAD)

- **category**: other
- **provider**: internal (custom), Go standard library `crypto/aes` + `crypto/cipher` (GCM)
- **reusable**: yes — a ~150-line package with no dependencies; works for any app that stores third-party API keys, OAuth/access tokens, webhook secrets or session blobs in its own database.
- **docs**: https://pkg.go.dev/crypto/cipher#NewGCM · https://csrc.nist.gov/pubs/sp/800/38/d/final

## Overview
Third-party credentials that users hand the app (API keys, bot or app tokens, webhook signing secrets, serialized client sessions) are sealed with AES-256-GCM before they reach the database and opened only in memory when needed. Each ciphertext is bound to its row through GCM's *additional authenticated data* (e.g. `"provider.access_token:42"`), so a value copied into another row or column fails to decrypt. Clients only ever see a hint (`…a1b2`), never the plaintext. Reach for it when you must store reversible secrets (not hashable passwords) in an app-managed database.

## Playbook

### Prerequisites
- Go (standard library only).
- A 32-byte key from an env var, or a key file generated on first start inside the data dir.

### Setup steps
1. Add a `secrets` package with `Box{Seal, Open}`, `LoadKey`, `ParseKey`, `Hint` and a key check (Core pattern).
2. At startup, load the key (env first, then file, else generate the file with mode `0600`), build a `Box`, and verify it against a stored check value.
3. Define **one AAD helper per sealed column**, e.g. `func AccessTokenAAD(id int64) string { return "provider.access_token:" + strconv.FormatInt(id, 10) }`. Always build AADs through these helpers, never inline strings.
4. Store `<column>_enc TEXT` (the sealed value) and `<column>_hint TEXT` (last 4 characters) side by side. API responses return only the hint plus a `has_<x>` boolean.
5. When the AAD needs an id that only exists after insert, seal inside the insert transaction through a callback that receives the new id (pattern below).
6. Map the "unreadable" error to a distinct API error (e.g. `409 secret_unreadable`) so the UI can prompt the user to re-enter the credential.
7. Back up the key file together with the database (see the SQLite backup playbook).

### Core pattern
```go
package secrets

import (
	"context"
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"encoding/hex"
	"errors"
	"fmt"
	"io/fs"
	"os"
	"path/filepath"
	"strings"
)

var ErrUnreadable = errors.New("secret is unreadable (the key changed or the value was altered)")

const version = "v1:" // lets you change the format or algorithm later

type Box struct{ aead cipher.AEAD }

func New(key []byte) (*Box, error) {
	if len(key) != 32 {
		return nil, fmt.Errorf("secret key must be 32 bytes, got %d", len(key))
	}
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	aead, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	return &Box{aead: aead}, nil
}

// Seal encrypts plain for the row named by aad, e.g. "provider.access_token:42".
func (b *Box) Seal(plain, aad string) (string, error) {
	nonce := make([]byte, b.aead.NonceSize()) // 12 random bytes per value
	if _, err := rand.Read(nonce); err != nil {
		return "", err
	}
	out := b.aead.Seal(nonce, nonce, []byte(plain), []byte(aad)) // nonce || ciphertext || tag
	return version + base64.RawStdEncoding.EncodeToString(out), nil
}

func (b *Box) Open(sealed, aad string) (string, error) {
	raw, ok := strings.CutPrefix(sealed, version)
	if !ok {
		return "", ErrUnreadable
	}
	data, err := base64.RawStdEncoding.DecodeString(raw)
	n := b.aead.NonceSize()
	if err != nil || len(data) < n {
		return "", ErrUnreadable
	}
	plain, err := b.aead.Open(nil, data[:n], data[n:], []byte(aad))
	if err != nil {
		return "", ErrUnreadable // wrong key, wrong row, or tampered: callers can't tell, by design
	}
	return string(plain), nil
}

// LoadKey: env value if set (64 hex chars or base64 of 32 bytes), else the key file,
// generated with 0600 on first use.
func LoadKey(envValue, path string) ([]byte, error) {
	if v := strings.TrimSpace(envValue); v != "" {
		return ParseKey(v)
	}
	raw, err := os.ReadFile(path)
	if err == nil {
		return ParseKey(strings.TrimSpace(string(raw)))
	}
	if !errors.Is(err, fs.ErrNotExist) {
		return nil, fmt.Errorf("read %s: %w", path, err)
	}
	key := make([]byte, 32)
	if _, err := rand.Read(key); err != nil {
		return nil, err
	}
	if err := os.MkdirAll(filepath.Dir(path), 0o700); err != nil {
		return nil, err
	}
	f, err := os.OpenFile(path, os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o600) // EXCL: never clobber
	if err != nil {
		return nil, fmt.Errorf("create %s: %w", path, err)
	}
	if _, err := f.WriteString(hex.EncodeToString(key) + "\n"); err != nil {
		f.Close()
		return nil, err
	}
	return key, f.Close()
}

func ParseKey(v string) ([]byte, error) {
	if len(v) == 64 {
		if k, err := hex.DecodeString(v); err == nil {
			return k, nil
		}
	}
	for _, enc := range []*base64.Encoding{base64.StdEncoding, base64.RawStdEncoding, base64.URLEncoding, base64.RawURLEncoding} {
		if k, err := enc.DecodeString(v); err == nil && len(k) == 32 {
			return k, nil
		}
	}
	return nil, errors.New("secret key must be 32 bytes as 64 hex characters or base64")
}

// Hint shows the last four characters, enough to recognise a credential without revealing it.
func Hint(secret string) string {
	r := []rune(secret)
	if len(r) <= 4 {
		return "…"
	}
	return "…" + string(r[len(r)-4:])
}

// Key check: a known plaintext sealed under the current key, stored once in a settings row.
const checkAAD, checkPlain = "server.secret_check", "secret check v1"

type CheckStore interface {
	SecretCheck(ctx context.Context) (string, error)
	SetSecretCheck(ctx context.Context, sealed string) error
}

// Verify reports whether b is the key that sealed the stored check value (writing one if absent).
func Verify(ctx context.Context, b *Box, cs CheckStore) (bool, error) {
	sealed, err := cs.SecretCheck(ctx)
	if err != nil {
		return false, err
	}
	if sealed == "" {
		s, err := b.Seal(checkPlain, checkAAD)
		if err != nil {
			return false, err
		}
		return true, cs.SetSecretCheck(ctx, s)
	}
	plain, err := b.Open(sealed, checkAAD)
	return err == nil && plain == checkPlain, nil
}
```

**Startup wiring**
```go
key, err := secrets.LoadKey(os.Getenv("APP_SECRET_KEY"), filepath.Join(dataDir, "secret.key"))
if err != nil { return err }
box, err := secrets.New(key)
if err != nil { return err }
if ok, err := secrets.Verify(ctx, box, store); err != nil {
	return err
} else if !ok {
	log.Warn("secret key changed: stored credentials are unreadable until users re-enter them")
}
```

**Seal with an id that only exists after INSERT (callback inside the tx)**
```go
acc, err := store.CreateIntegration(ctx, userID, row, func(id int64) (tokenEnc, secretEnc string, err error) {
	if tokenEnc, err = box.Seal(in.AccessToken, AccessTokenAAD(id)); err != nil {
		return
	}
	secretEnc, err = box.Seal(in.AppSecret, AppSecretAAD(id))
	return
})
// store.CreateIntegration: INSERT row → LastInsertId → call fn(id) → UPDATE row SET *_enc = ? → COMMIT
```

**Read path and API shape**
```go
tok, err := box.Open(row.AccessTokenEnc, AccessTokenAAD(row.ID))
switch {
case errors.Is(err, secrets.ErrUnreadable):
	// mark the integration unhealthy; API answers 409 {"error":"secret_unreadable"}
case err != nil:
	return err
}
// JSON never carries tok: {"has_access_token": true, "access_token_hint": "…a1b2"}
```

### Env vars
- `APP_SECRET_KEY`: optional; 64 hex chars or base64 of 32 bytes. When unset, `<data dir>/secret.key` is used, and generated on first run.

### Gotchas
- **AAD is what stops row swapping.** Without it, anyone with write access to the database can move a sealed token from row A into row B and the app happily decrypts it for B. Include both the *purpose* (column) and the *row id* in the AAD, and build it through one helper per column so seal and open can't drift apart.
- **Fresh random nonce per Seal.** GCM breaks badly (it leaks the auth key and XOR of plaintexts) if a nonce repeats under one key. 96-bit random nonces are fine up to roughly 2^32 encryptions per key, far beyond typical credential volumes.
- **Version-prefix the ciphertext** (`v1:`). It lets you introduce key ids, a new algorithm, or envelope encryption later without guessing formats.
- **Opaque failure is intentional.** Wrong key, wrong row and tampering all return `ErrUnreadable`. Surface it as "please re-enter this credential", not as a 500.
- **A changed key shouldn't crash the server.** Verify against a stored check value at startup, log a warning, and let affected features report unreadable. Refusing to boot would lock out unrelated functionality.
- **Key file safety**: create with `O_EXCL` and `0600` in a `0700` dir. Never regenerate over an existing file. Back it up with the database. A restored database without its key is useless, and a key next to the database in the same public bucket is no protection.
- **Hints only, everywhere.** API responses, logs, error messages, audit entries and SSE payloads carry at most the hint. Echo the full value once, at creation time, only when the app itself generated it (e.g. a webhook signing secret the user must copy).
- **Rotation**: support `APP_SECRET_KEY_OLD` (or a key id in the prefix). On startup, open with the old key, re-seal with the new one in a transaction per row, re-write the check value, then drop the old key. Without a rotation tool, a key change means users re-enter every credential.
- **What not to use this for**: user passwords and your own API tokens should be *hashed* (Argon2/bcrypt, or SHA-256 for high-entropy random tokens), not encrypted. Encrypt only what the app must send back out to a third party.
- Seal serialized session blobs (e.g. an MTProto or device session) the same way, base64-encoding binary first, with an AAD like `"provider.session:<account id>"`.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
