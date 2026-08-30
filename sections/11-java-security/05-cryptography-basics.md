# Q5: How Does Java Cryptography Work?

[Home](../../README.md) > [Java Security](./README.md) > Cryptography Basics

---

## Overview

The **Java Cryptography Architecture (JCA)** provides a framework and implementations for encryption, hashing, digital signatures, and key management. It uses a **provider-based** architecture where cryptographic algorithms are supplied by pluggable security providers.

## JCA Provider Model

```
Application Code
    |
    v
JCA API (javax.crypto, java.security)
    |
    v
+----------+    +----------+    +----------+
| SunJCE   |    | Bouncy   |    | Custom   |
| Provider |    | Castle   |    | Provider |
+----------+    +----------+    +----------+
```

```java
import java.security.*;

public class ProviderDemo {

    public static void main(String[] args) {
        // List all installed providers
        for (Provider provider : Security.getProviders()) {
            System.out.println(provider.getName() + " v" + provider.getVersionStr());
        }

        // Request a specific provider
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256", "SUN");
            System.out.println("Provider: " + md.getProvider().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## Message Digests (Hashing)

```java
import java.security.MessageDigest;
import java.nio.charset.StandardCharsets;
import java.util.HexFormat;

public class HashingDemo {

    // ✅ Using SHA-256 for secure hashing
    public static String sha256(String input) throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
        return HexFormat.of().formatHex(hash);
    }

    // ✅ Hashing large files incrementally
    public static String hashFile(java.nio.file.Path filePath) throws Exception {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");

        try (var is = java.nio.file.Files.newInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                digest.update(buffer, 0, bytesRead);
            }
        }

        return HexFormat.of().formatHex(digest.digest());
    }

    // ❌ Anti-pattern: Using MD5 for security purposes
    public static String insecureMd5(String input) throws Exception {
        MessageDigest md = MessageDigest.getInstance("MD5");  // WEAK - collision attacks
        return HexFormat.of().formatHex(md.digest(input.getBytes()));
    }

    public static void main(String[] args) throws Exception {
        System.out.println("SHA-256: " + sha256("Hello, World!"));
    }
}
```

## Symmetric Encryption (AES-GCM)

```java
import javax.crypto.*;
import javax.crypto.spec.*;
import java.security.SecureRandom;
import java.util.Base64;

public class AESEncryption {

    private static final int GCM_IV_LENGTH = 12;   // 96 bits recommended for GCM
    private static final int GCM_TAG_LENGTH = 128;  // 128-bit auth tag

    // ✅ Best practice: AES-GCM (authenticated encryption)
    public static byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
        byte[] iv = new byte[GCM_IV_LENGTH];
        new SecureRandom().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);

        byte[] ciphertext = cipher.doFinal(plaintext);

        // Prepend IV to ciphertext for storage
        byte[] result = new byte[iv.length + ciphertext.length];
        System.arraycopy(iv, 0, result, 0, iv.length);
        System.arraycopy(ciphertext, 0, result, iv.length, ciphertext.length);

        return result;
    }

    public static byte[] decrypt(byte[] encryptedData, SecretKey key) throws Exception {
        // Extract IV from the beginning
        byte[] iv = new byte[GCM_IV_LENGTH];
        System.arraycopy(encryptedData, 0, iv, 0, iv.length);

        byte[] ciphertext = new byte[encryptedData.length - iv.length];
        System.arraycopy(encryptedData, iv.length, ciphertext, 0, ciphertext.length);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
        cipher.init(Cipher.DECRYPT_MODE, key, spec);

        return cipher.doFinal(ciphertext);
    }

    // ✅ Generate a secure AES key
    public static SecretKey generateKey(int keySize) throws Exception {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(keySize, new SecureRandom());  // 128, 192, or 256 bits
        return keyGen.generateKey();
    }

    // ❌ Anti-pattern: Using ECB mode
    public static byte[] insecureEcb(byte[] data, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");  // ECB is insecure!
        cipher.init(Cipher.ENCRYPT_MODE, key);
        return cipher.doFinal(data);
        // ECB encrypts identical blocks to identical ciphertext - leaks patterns
    }

    public static void main(String[] args) throws Exception {
        SecretKey key = generateKey(256);
        String message = "Sensitive data: credit card 4111-1111-1111-1111";

        byte[] encrypted = encrypt(message.getBytes(), key);
        System.out.println("Encrypted: " + Base64.getEncoder().encodeToString(encrypted));

        byte[] decrypted = decrypt(encrypted, key);
        System.out.println("Decrypted: " + new String(decrypted));
    }
}
```

## Asymmetric Encryption (RSA)

```java
import java.security.*;
import javax.crypto.Cipher;
import java.util.Base64;

public class RSAEncryption {

    // ✅ Generate RSA key pair
    public static KeyPair generateKeyPair() throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048, new SecureRandom());  // Minimum 2048 bits
        return keyGen.generateKeyPair();
    }

    // ✅ Encrypt with public key (only private key can decrypt)
    public static byte[] encrypt(byte[] data, PublicKey publicKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return cipher.doFinal(data);
    }

    // ✅ Decrypt with private key
    public static byte[] decrypt(byte[] data, PrivateKey privateKey) throws Exception {
        Cipher cipher = Cipher.getInstance("RSA/ECB/OAEPWithSHA-256AndMGF1Padding");
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return cipher.doFinal(data);
    }

    // ❌ Anti-pattern: Using textbook RSA without padding
    // Cipher.getInstance("RSA/ECB/NoPadding");  // Vulnerable to attacks!

    public static void main(String[] args) throws Exception {
        KeyPair keyPair = generateKeyPair();

        String secret = "API_KEY=abc123xyz";
        byte[] encrypted = encrypt(secret.getBytes(), keyPair.getPublic());
        byte[] decrypted = decrypt(encrypted, keyPair.getPrivate());

        System.out.println("Original:  " + secret);
        System.out.println("Encrypted: " + Base64.getEncoder().encodeToString(encrypted));
        System.out.println("Decrypted: " + new String(decrypted));
    }
}
```

## Digital Signatures

```java
import java.security.*;
import java.util.Base64;

public class DigitalSignatureDemo {

    // ✅ Sign data with a private key
    public static byte[] sign(byte[] data, PrivateKey privateKey) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initSign(privateKey, new SecureRandom());
        signature.update(data);
        return signature.sign();
    }

    // ✅ Verify signature with a public key
    public static boolean verify(byte[] data, byte[] signatureBytes,
                                  PublicKey publicKey) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initVerify(publicKey);
        signature.update(data);
        return signature.verify(signatureBytes);
    }

    public static void main(String[] args) throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);
        KeyPair keyPair = keyGen.generateKeyPair();

        String message = "Transfer $1000 to account 12345";
        byte[] data = message.getBytes();

        // Sign the message
        byte[] sig = sign(data, keyPair.getPrivate());
        System.out.println("Signature: " + Base64.getEncoder().encodeToString(sig));

        // Verify the signature
        boolean valid = verify(data, sig, keyPair.getPublic());
        System.out.println("Valid signature: " + valid);  // true

        // Tampered message fails verification
        byte[] tampered = "Transfer $9999 to account 12345".getBytes();
        boolean tamperedValid = verify(tampered, sig, keyPair.getPublic());
        System.out.println("Tampered valid: " + tamperedValid);  // false
    }
}
```

## KeyStore for Key Management

```java
import java.security.KeyStore;
import javax.crypto.SecretKey;
import java.io.*;

public class KeyStoreDemo {

    // ✅ Store and retrieve keys securely
    public static void storeSecretKey(SecretKey key, String alias,
                                       String keystorePath, char[] password) throws Exception {
        KeyStore ks = KeyStore.getInstance("PKCS12");

        // Load existing or create new
        File file = new File(keystorePath);
        if (file.exists()) {
            try (FileInputStream fis = new FileInputStream(file)) {
                ks.load(fis, password);
            }
        } else {
            ks.load(null, password);
        }

        // Store the key
        KeyStore.SecretKeyEntry entry = new KeyStore.SecretKeyEntry(key);
        ks.setEntry(alias, entry, new KeyStore.PasswordProtection(password));

        // Save to file
        try (FileOutputStream fos = new FileOutputStream(keystorePath)) {
            ks.store(fos, password);
        }
    }

    public static SecretKey loadSecretKey(String alias, String keystorePath,
                                           char[] password) throws Exception {
        KeyStore ks = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            ks.load(fis, password);
        }
        KeyStore.SecretKeyEntry entry = (KeyStore.SecretKeyEntry)
            ks.getEntry(alias, new KeyStore.PasswordProtection(password));
        return entry.getSecretKey();
    }
}
```

## Algorithm Quick Reference

| Category | Recommended | Avoid |
|----------|------------|-------|
| **Hashing** | SHA-256, SHA-3 | MD5, SHA-1 |
| **Symmetric** | AES-256-GCM | DES, 3DES, AES-ECB |
| **Asymmetric** | RSA-2048+, EC P-256 | RSA-1024 |
| **Signatures** | SHA256withRSA, SHA256withECDSA | MD5withRSA, SHA1withDSA |
| **Key derivation** | PBKDF2, Argon2 (via library) | Plain hash of password |
| **MAC** | HMAC-SHA256 | HMAC-MD5 |

## Interview Tips

| Point | Detail |
|-------|--------|
| **JCA** | Provider-based architecture; algorithms are pluggable |
| **Hashing** | One-way; use SHA-256+; avoid MD5/SHA-1 |
| **Symmetric** | AES-GCM preferred; always use authenticated encryption |
| **Asymmetric** | RSA with OAEP padding; minimum 2048-bit keys |
| **Signatures** | Private key signs, public key verifies |
| **KeyStore** | PKCS12 format stores keys and certificates securely |
| **IV/Nonce** | Never reuse IVs; generate with SecureRandom |

---

[<< Previous: Q4 - JAAS](./q04-jaas.md) | [Next: Q6 - SecureRandom >>](./q06-secure-random.md)
