# Q3: What is Code Signing in Java?

[Home](../../README.md) > [Java Security](./README.md) > Code Signing

---

## Overview

Code signing in Java is the process of digitally signing JAR files to verify their **integrity** (the code hasn't been tampered with) and **authenticity** (the code comes from a trusted source). It uses public-key cryptography with certificates stored in a **KeyStore**.

## Code Signing Workflow

```
Developer                          User/Runtime
    |                                   |
    |  1. Generate key pair             |
    |     (keytool -genkeypair)         |
    |                                   |
    |  2. Sign JAR                      |
    |     (jarsigner myapp.jar alias)   |
    |                                   |
    |  3. Distribute signed JAR ------> |
    |                                   |  4. Verify signature
    |                                   |     (jarsigner -verify)
    |                                   |
    |                                   |  5. Check certificate trust
    |                                   |     (cacerts truststore)
```

## KeyStore and keytool

```bash
# Generate a new key pair in a keystore
keytool -genkeypair \
    -alias myapp \
    -keyalg RSA \
    -keysize 2048 \
    -validity 365 \
    -keystore myapp-keystore.jks \
    -storepass changeit \
    -dname "CN=My App, OU=Dev, O=MyCompany, L=City, ST=State, C=US"

# List keystore contents
keytool -list -keystore myapp-keystore.jks -storepass changeit

# Export the public certificate
keytool -exportcert \
    -alias myapp \
    -keystore myapp-keystore.jks \
    -file myapp-cert.cer \
    -storepass changeit

# Import a trusted certificate into a truststore
keytool -importcert \
    -alias trustedvendor \
    -file vendor-cert.cer \
    -keystore myapp-truststore.jks \
    -storepass changeit

# Generate a Certificate Signing Request (CSR)
keytool -certreq \
    -alias myapp \
    -keystore myapp-keystore.jks \
    -file myapp.csr \
    -storepass changeit

# Modern format: PKCS12 (preferred over JKS)
keytool -genkeypair \
    -alias myapp \
    -keyalg RSA \
    -keysize 4096 \
    -storetype PKCS12 \
    -keystore myapp-keystore.p12 \
    -validity 730 \
    -storepass securePassword123
```

## Signing JARs with jarsigner

```bash
# Sign a JAR file
jarsigner -keystore myapp-keystore.jks \
    -storepass changeit \
    myapp.jar myapp

# Sign with timestamping (critical for long-term validity)
jarsigner -keystore myapp-keystore.jks \
    -storepass changeit \
    -tsa http://timestamp.digicert.com \
    myapp.jar myapp

# Verify a signed JAR
jarsigner -verify myapp.jar

# Verify with verbose output and certificate details
jarsigner -verify -verbose -certs myapp.jar
```

## Certificate Chain

```
Root CA Certificate (self-signed, in cacerts)
    |
    +-- Intermediate CA Certificate
            |
            +-- Code Signing Certificate (your key)
                    |
                    +-- Signed JAR file
```

```java
import java.security.KeyStore;
import java.security.cert.*;
import java.io.FileInputStream;

public class CertificateChainDemo {

    public static void inspectCertificateChain(String keystorePath,
                                                String alias) throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, "changeit".toCharArray());
        }

        // Get the certificate chain
        Certificate[] chain = keyStore.getCertificateChain(alias);
        if (chain == null) {
            System.out.println("No certificate chain found for: " + alias);
            return;
        }

        System.out.println("Certificate chain length: " + chain.length);
        for (int i = 0; i < chain.length; i++) {
            X509Certificate cert = (X509Certificate) chain[i];
            System.out.println("\n--- Certificate [" + i + "] ---");
            System.out.println("Subject : " + cert.getSubjectX500Principal());
            System.out.println("Issuer  : " + cert.getIssuerX500Principal());
            System.out.println("Valid   : " + cert.getNotBefore() + " to " + cert.getNotAfter());
            System.out.println("Serial  : " + cert.getSerialNumber());
            System.out.println("Algo    : " + cert.getSigAlgName());
        }
    }
}
```

## Programmatic JAR Verification

```java
import java.util.jar.*;
import java.security.cert.Certificate;
import java.io.*;

public class JarVerifier {

    public static boolean verifyJar(String jarPath) throws IOException {
        try (JarFile jarFile = new JarFile(jarPath, true)) {  // true = verify

            byte[] buffer = new byte[8192];
            java.util.Enumeration<JarEntry> entries = jarFile.entries();

            while (entries.hasMoreElements()) {
                JarEntry entry = entries.nextElement();

                // Skip directories and manifest files
                if (entry.isDirectory() || entry.getName().startsWith("META-INF/")) {
                    continue;
                }

                // Reading the entry triggers signature verification
                try (InputStream is = jarFile.getInputStream(entry)) {
                    while (is.read(buffer) != -1) { /* consume */ }
                }

                // Check that entry was signed
                Certificate[] certs = entry.getCertificates();
                if (certs == null || certs.length == 0) {
                    System.err.println("Unsigned entry: " + entry.getName());
                    return false;
                }
            }

            // Verify manifest exists
            Manifest manifest = jarFile.getManifest();
            if (manifest == null) {
                System.err.println("No manifest found");
                return false;
            }

            System.out.println("JAR verified successfully: " + jarPath);
            return true;
        } catch (SecurityException e) {
            System.err.println("Signature verification failed: " + e.getMessage());
            return false;
        }
    }
}
```

## Timestamping

```
Without Timestamping:
    Certificate expires Dec 2025
    Signature created Nov 2024
    After Dec 2025 --> Signature INVALID (even though it was valid when signed)

With Timestamping:
    Certificate expires Dec 2025
    Signature created Nov 2024, timestamped by TSA
    After Dec 2025 --> Signature STILL VALID (TSA proves it was signed before expiry)
```

```bash
# Always use timestamping for production releases
jarsigner -keystore release.p12 \
    -storetype PKCS12 \
    -tsa http://timestamp.digicert.com \
    -tsadigestalg SHA-256 \
    release-app.jar releaseKey
```

## When Code Signing Matters

| Scenario | Importance |
|----------|------------|
| **Applets (legacy)** | Required - browser needed signed code for elevated access |
| **Java Web Start (legacy)** | Required for system access outside sandbox |
| **Maven Central publishing** | Required - artifacts must be GPG signed |
| **Enterprise deployment** | Best practice - verifies internal code integrity |
| **Plugin/extension systems** | Important - validates third-party code |
| **Regulatory compliance** | Often required (HIPAA, PCI-DSS, SOX) |

## Anti-Patterns and Best Practices

```java
public class CodeSigningPractices {

    // ❌ Anti-pattern: Using weak key algorithms
    // keytool -genkeypair -keyalg DSA -keysize 1024 ...
    // DSA 1024-bit keys are considered insecure

    // ✅ Best practice: Use strong algorithms
    // keytool -genkeypair -keyalg RSA -keysize 4096 -sigalg SHA384withRSA ...

    // ❌ Anti-pattern: Storing keystore passwords in scripts
    // jarsigner -storepass myP@ssw0rd -keystore key.p12 app.jar alias

    // ✅ Best practice: Use environment variables or prompt
    // jarsigner -keystore key.p12 app.jar alias   (prompts for password)

    // ❌ Anti-pattern: Signing without timestamping
    // jarsigner -keystore key.p12 app.jar alias

    // ✅ Best practice: Always timestamp
    // jarsigner -keystore key.p12 -tsa http://timestamp.digicert.com app.jar alias

    // ❌ Anti-pattern: Using self-signed certs in production
    // Self-signed certificates cannot be verified by standard trust chains

    // ✅ Best practice: Use CA-issued certificates for production
    // 1. Generate CSR with keytool -certreq
    // 2. Submit to a trusted Certificate Authority
    // 3. Import the signed certificate back into the keystore
}
```

## Working with KeyStore Programmatically

```java
import java.security.KeyStore;
import java.security.PrivateKey;
import java.security.cert.X509Certificate;
import java.io.FileInputStream;

public class KeyStoreOperations {

    // ✅ Load and inspect a keystore
    public static KeyStore loadKeyStore(String path, char[] password,
                                         String type) throws Exception {
        KeyStore ks = KeyStore.getInstance(type);  // "PKCS12" or "JKS"
        try (FileInputStream fis = new FileInputStream(path)) {
            ks.load(fis, password);
        }
        return ks;
    }

    // ✅ Retrieve a private key for signing
    public static PrivateKey getPrivateKey(KeyStore ks, String alias,
                                            char[] keyPassword) throws Exception {
        KeyStore.PrivateKeyEntry entry = (KeyStore.PrivateKeyEntry)
            ks.getEntry(alias, new KeyStore.PasswordProtection(keyPassword));
        return entry.getPrivateKey();
    }

    // ✅ Always clear sensitive data from memory
    public static void secureCleanup(char[] password) {
        java.util.Arrays.fill(password, '\0');
    }
}
```

## Interview Tips

| Point | Detail |
|-------|--------|
| **Purpose** | Verify integrity (not tampered) and authenticity (trusted source) |
| **Tools** | `keytool` for key/cert management, `jarsigner` for signing JARs |
| **KeyStore types** | PKCS12 (modern default) vs JKS (legacy) |
| **Timestamping** | Ensures signatures remain valid after certificate expiry |
| **Certificate chain** | Root CA > Intermediate CA > Code Signing Cert |
| **Best practice** | RSA 4096-bit, SHA-256+, CA-issued certs, always timestamp |

---

[<< Previous: Q2 - Permissions & Policies](./q02-permissions-policies.md) | [Next: Q4 - JAAS >>](./q04-jaas.md)
