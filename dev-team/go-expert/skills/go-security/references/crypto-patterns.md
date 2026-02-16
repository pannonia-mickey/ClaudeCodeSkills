# Crypto Patterns

## AES-GCM Encryption

```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "io"
)

func encrypt(plaintext, key []byte) ([]byte, error) {
    block, err := aes.NewCipher(key) // key must be 16, 24, or 32 bytes
    if err != nil {
        return nil, err
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, aesGCM.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }

    // Nonce prepended to ciphertext
    return aesGCM.Seal(nonce, nonce, plaintext, nil), nil
}

func decrypt(ciphertext, key []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }

    aesGCM, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonceSize := aesGCM.NonceSize()
    if len(ciphertext) < nonceSize {
        return nil, errors.New("ciphertext too short")
    }

    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    return aesGCM.Open(nil, nonce, ciphertext, nil)
}
```

## Key Derivation (Argon2)

```go
import "golang.org/x/crypto/argon2"

type KeyDerivationParams struct {
    Memory      uint32
    Iterations  uint32
    Parallelism uint8
    SaltLength  uint32
    KeyLength   uint32
}

var DefaultParams = KeyDerivationParams{
    Memory:      64 * 1024, // 64 MB
    Iterations:  3,
    Parallelism: 2,
    SaltLength:  16,
    KeyLength:   32,
}

func deriveKey(password string, params KeyDerivationParams) (key, salt []byte, err error) {
    salt = make([]byte, params.SaltLength)
    if _, err := rand.Read(salt); err != nil {
        return nil, nil, err
    }

    key = argon2.IDKey(
        []byte(password),
        salt,
        params.Iterations,
        params.Memory,
        params.Parallelism,
        params.KeyLength,
    )

    return key, salt, nil
}
```

## Secure Random Generation

```go
import "crypto/rand"

// Generate cryptographically secure random bytes
func generateRandomBytes(n int) ([]byte, error) {
    b := make([]byte, n)
    _, err := rand.Read(b)
    return b, err
}

// Generate URL-safe random string
func generateRandomString(length int) (string, error) {
    b := make([]byte, length)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b)[:length], nil
}

// Generate API key
func generateAPIKey() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return "sk_" + hex.EncodeToString(b), nil
}

// NEVER use math/rand for security-sensitive operations
// BAD: rand.Intn(100) for tokens, passwords, keys
```

## Digital Signatures (Ed25519)

```go
import "crypto/ed25519"

// Generate key pair
func generateKeyPair() (ed25519.PublicKey, ed25519.PrivateKey, error) {
    return ed25519.GenerateKey(rand.Reader)
}

// Sign message
func signMessage(privateKey ed25519.PrivateKey, message []byte) []byte {
    return ed25519.Sign(privateKey, message)
}

// Verify signature
func verifySignature(publicKey ed25519.PublicKey, message, sig []byte) bool {
    return ed25519.Verify(publicKey, message, sig)
}

// Webhook signature verification
func verifyWebhookSignature(payload []byte, signature string, secret []byte) bool {
    mac := hmac.New(sha256.New, secret)
    mac.Write(payload)
    expectedMAC := mac.Sum(nil)
    expectedSig := hex.EncodeToString(expectedMAC)
    return hmac.Equal([]byte(signature), []byte(expectedSig))
}
```

## Secrets Management

```go
// Environment variables (simplest approach)
func loadSecrets() (*Secrets, error) {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return nil, errors.New("DATABASE_URL is required")
    }
    return &Secrets{DatabaseURL: dbURL}, nil
}

// HashiCorp Vault integration
import vault "github.com/hashicorp/vault/api"

func getVaultSecret(path string) (map[string]any, error) {
    config := vault.DefaultConfig()
    client, err := vault.NewClient(config)
    if err != nil {
        return nil, err
    }

    // Token from VAULT_TOKEN env or Kubernetes auth
    secret, err := client.KVv2("secret").Get(context.Background(), path)
    if err != nil {
        return nil, fmt.Errorf("reading secret %s: %w", path, err)
    }

    return secret.Data, nil
}

// AWS Secrets Manager
import "github.com/aws/aws-sdk-go-v2/service/secretsmanager"

func getAWSSecret(ctx context.Context, client *secretsmanager.Client, name string) (string, error) {
    input := &secretsmanager.GetSecretValueInput{SecretId: &name}
    result, err := client.GetSecretValue(ctx, input)
    if err != nil {
        return "", fmt.Errorf("getting secret %s: %w", name, err)
    }
    return *result.SecretString, nil
}
```

## Constant-Time Comparison

```go
import "crypto/subtle"

// Always use constant-time comparison for secrets
func validateAPIKey(provided, stored string) bool {
    return subtle.ConstantTimeCompare([]byte(provided), []byte(stored)) == 1
}

// NEVER use == for secret comparison
// BAD: if token == storedToken { ... }
// Timing attacks can leak information about the secret
```
