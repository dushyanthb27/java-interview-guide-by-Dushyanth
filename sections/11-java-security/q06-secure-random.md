# Q6: What is SecureRandom and Why is It Important?

[Home](../../README.md) > [Java Security](./README.md) > SecureRandom

---

## Overview

`SecureRandom` is Java's **cryptographically strong random number generator (CSPRNG)**. Unlike `java.util.Random`, it produces unpredictable output suitable for security-sensitive operations like key generation, tokens, nonces, and password salts.

## SecureRandom vs Random

```java
import java.security.SecureRandom;
import java.util.Random;

public class RandomComparison {

    public static void main(String[] args) {
        // ❌ java.util.Random - PREDICTABLE! Uses a linear congruential generator
        Random insecure = new Random(12345);  // Same seed = same sequence every time
        System.out.println("Random: " + insecure.nextInt());  // Always the same output

        // ❌ Even without an explicit seed, Random uses System.nanoTime()
        // An attacker who knows the approximate creation time can predict the sequence
        Random stillBad = new Random();

        // ✅ SecureRandom - UNPREDICTABLE! Uses OS entropy sources
        SecureRandom secure = new SecureRandom();
        System.out.println("SecureRandom: " + secure.nextInt());  // Unpredictable
    }
}
```

### Why Random is Dangerous for Security

```java
public class RandomVulnerability {

    // ❌ Anti-pattern: Using Random for session tokens
    public static String insecureToken() {
        Random random = new Random();  // Predictable!
        StringBuilder token = new StringBuilder();
        for (int i = 0; i < 32; i++) {
            token.append(Integer.toHexString(random.nextInt(16)));
        }
        return token.toString();
        // An attacker who observes a few tokens can predict future ones
    }

    // ✅ Best practice: Using SecureRandom for session tokens
    public static String secureToken() {
        SecureRandom random = new SecureRandom();
        byte[] bytes = new byte[32];
        random.nextBytes(bytes);
        return java.util.HexFormat.of().formatHex(bytes);
    }
}
```

## Entropy Sources

```
+-------------------+     +-------------------+
|  Hardware Events   |     |  OS Entropy Pool  |
|  - Disk I/O timing |---->|  /dev/random      |---> SecureRandom
|  - Network packets |     |  /dev/urandom     |     (Java)
|  - Key strokes     |     |  CryptGenRandom   |
|  - Mouse movements |     |  (Windows)        |
+-------------------+     +-------------------+
```

## Algorithm Selection

```java
import java.security.SecureRandom;
import java.security.Provider;

public class SecureRandomAlgorithms {

    public static void main(String[] args) throws Exception {
        // Default (platform-dependent, usually best choice)
        SecureRandom defaultRng = new SecureRandom();
        System.out.println("Default algo: " + defaultRng.getAlgorithm());
        System.out.println("Provider: " + defaultRng.getProvider().getName());

        // Specific algorithms
        SecureRandom nativePrng = SecureRandom.getInstance("NativePRNG");   // Linux/Mac
        SecureRandom drbg = SecureRandom.getInstance("DRBG");              // Java 9+
        // SecureRandom windows = SecureRandom.getInstance("Windows-PRNG"); // Windows only

        // ✅ Best practice: Use getInstanceStrong() for long-term key generation
        SecureRandom strongest = SecureRandom.getInstanceStrong();
        System.out.println("Strongest: " + strongest.getAlgorithm());

        // List all available SecureRandom algorithms
        for (Provider provider : java.security.Security.getProviders()) {
            provider.getServices().stream()
                .filter(s -> "SecureRandom".equals(s.getType()))
                .forEach(s -> System.out.println(
                    s.getAlgorithm() + " [" + provider.getName() + "]"));
        }
    }
}
```

## NativePRNG vs DRBG

| Feature | NativePRNG | DRBG (Java 9+) |
|---------|-----------|-----------------|
| **Source** | OS entropy (`/dev/urandom`) | Deterministic algorithm seeded with entropy |
| **Standard** | OS-dependent | NIST SP 800-90Ar1 |
| **Speed** | Moderate | Configurable |
| **Blocking** | Non-blocking (urandom) | Non-blocking |
| **Reseeding** | Continuous from OS | Configurable |
| **Best for** | General use on Linux/Mac | FIPS compliance, configurable security |

```java
import java.security.DrbgParameters;
import java.security.SecureRandom;

public class DrbgDemo {

    public static void main(String[] args) throws Exception {
        // ✅ DRBG with explicit security strength (Java 9+)
        SecureRandom drbg = SecureRandom.getInstance("DRBG",
            DrbgParameters.instantiation(
                256,                                    // Security strength in bits
                DrbgParameters.Capability.PR_AND_RESEED, // Prediction resistance
                "MyApp-personalization".getBytes()       // Personalization string
            )
        );

        byte[] randomBytes = new byte[32];
        drbg.nextBytes(randomBytes);
    }
}
```

## Performance Considerations

```java
import java.security.SecureRandom;

public class SecureRandomPerformance {

    // ❌ Anti-pattern: Creating new instance for every call
    public byte[] slowApproach() {
        SecureRandom sr = new SecureRandom();  // Expensive construction each time
        byte[] bytes = new byte[16];
        sr.nextBytes(bytes);
        return bytes;
    }

    // ✅ Best practice: Reuse a single instance (SecureRandom is thread-safe)
    private static final SecureRandom SECURE_RANDOM = new SecureRandom();

    public byte[] fastApproach() {
        byte[] bytes = new byte[16];
        SECURE_RANDOM.nextBytes(bytes);
        return bytes;
    }

    // ✅ For high-throughput: use ThreadLocal to avoid contention
    private static final ThreadLocal<SecureRandom> THREAD_LOCAL_RANDOM =
        ThreadLocal.withInitial(SecureRandom::new);

    public byte[] highThroughputApproach() {
        byte[] bytes = new byte[16];
        THREAD_LOCAL_RANDOM.get().nextBytes(bytes);
        return bytes;
    }

    // ❌ Anti-pattern: Using getInstanceStrong() for high-volume operations
    // getInstanceStrong() may block waiting for entropy
    public byte[] potentiallyBlocking() throws Exception {
        SecureRandom strong = SecureRandom.getInstanceStrong();  // May block!
        byte[] bytes = new byte[16];
        strong.nextBytes(bytes);
        return bytes;
    }
}
```

## Seeding

```java
import java.security.SecureRandom;

public class SeedingDemo {

    public static void main(String[] args) {
        // ✅ Auto-seeded (recommended - uses OS entropy)
        SecureRandom autoSeeded = new SecureRandom();

        // ✅ Generate seed bytes for other uses
        byte[] seed = SecureRandom.getSeed(32);  // 32 bytes of entropy

        // ❌ Anti-pattern: Using a fixed seed (makes output predictable!)
        SecureRandom fixedSeed = new SecureRandom(new byte[]{1, 2, 3, 4});
        // This is only acceptable for reproducible testing, NEVER in production

        // ✅ Reseed periodically for long-running processes
        SecureRandom longRunning = new SecureRandom();
        // After generating a large amount of random data:
        longRunning.reseed();  // Java 9+ : forces reseeding from entropy source

        // ✅ Add supplemental entropy (does not replace internal seed)
        longRunning.setSeed(System.nanoTime());  // Mixes in additional entropy
    }
}
```

## Thread Safety

```java
import java.security.SecureRandom;
import java.util.concurrent.*;

public class ThreadSafetyDemo {

    // ✅ SecureRandom IS thread-safe - safe to share across threads
    private static final SecureRandom SHARED = new SecureRandom();

    public static void main(String[] args) throws Exception {
        ExecutorService pool = Executors.newFixedThreadPool(10);

        // Multiple threads can safely use the same instance
        var futures = new java.util.ArrayList<Future<String>>();
        for (int i = 0; i < 100; i++) {
            futures.add(pool.submit(() -> {
                byte[] token = new byte[16];
                SHARED.nextBytes(token);  // Thread-safe
                return java.util.HexFormat.of().formatHex(token);
            }));
        }

        for (var future : futures) {
            System.out.println("Token: " + future.get());
        }

        pool.shutdown();
    }
}
```

## Common Use Cases

```java
import java.security.SecureRandom;
import java.util.Base64;

public class SecureRandomUseCases {

    private static final SecureRandom RANDOM = new SecureRandom();

    // ✅ Generate a password salt
    public static byte[] generateSalt() {
        byte[] salt = new byte[16];
        RANDOM.nextBytes(salt);
        return salt;
    }

    // ✅ Generate an API key
    public static String generateApiKey() {
        byte[] bytes = new byte[32];
        RANDOM.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    // ✅ Generate an AES initialization vector
    public static byte[] generateIV() {
        byte[] iv = new byte[12];  // 96-bit IV for AES-GCM
        RANDOM.nextBytes(iv);
        return iv;
    }

    // ✅ Generate a CSRF token
    public static String generateCsrfToken() {
        byte[] bytes = new byte[32];
        RANDOM.nextBytes(bytes);
        return java.util.HexFormat.of().formatHex(bytes);
    }

    // ✅ Secure random integer in a range
    public static int secureRandomInt(int bound) {
        return RANDOM.nextInt(bound);
    }
}
```

## Configuration

```bash
# java.security configuration file settings

# Set the default SecureRandom algorithm
securerandom.strongAlgorithms=NativePRNGBlocking:SUN,DRBG:SUN

# Set entropy source (Linux)
securerandom.source=file:/dev/urandom

# For Docker/cloud: ensure /dev/urandom is available
# Some minimal containers may not have adequate entropy sources
```

## Interview Tips

| Point | Detail |
|-------|--------|
| **Why not Random** | `Random` uses LCG - predictable; `SecureRandom` uses OS entropy |
| **CSPRNG** | Cryptographically Secure Pseudo-Random Number Generator |
| **Thread safety** | `SecureRandom` is thread-safe; reuse instances |
| **getInstanceStrong()** | Best for key generation; may block on low-entropy systems |
| **DRBG** | NIST-standard algorithm available since Java 9 |
| **Seeding** | Auto-seeded by default; never use fixed seeds in production |
| **Performance** | Reuse instances; use ThreadLocal for high-throughput |

---

[<< Previous: Q5 - Cryptography Basics](./q05-cryptography-basics.md) | [Next: Q7 - SSL/TLS >>](./q07-ssl-tls.md)
