# Q7: How Does SSL/TLS Work in Java?

[Home](../../README.md) > [Java Security](./README.md) > SSL/TLS

---

## Overview

Java provides built-in support for SSL/TLS through the **Java Secure Socket Extension (JSSE)**. It enables encrypted communication between clients and servers using `SSLContext`, `SSLSocketFactory`, `TrustManager`, and `KeyManager`.

## TLS Handshake Flow

```
Client                                Server
  |                                     |
  |  1. ClientHello (TLS version,       |
  |     cipher suites, random)  ------> |
  |                                     |
  |  <------ 2. ServerHello (chosen     |
  |     cipher, certificate, random)    |
  |                                     |
  |  3. Verify server certificate       |
  |     (TrustManager)                  |
  |                                     |
  |  4. Key exchange  ---------------->  |
  |  (Pre-master secret encrypted       |
  |   with server's public key)         |
  |                                     |
  |  <======= 5. Encrypted session ====>|
  |     (Symmetric encryption with      |
  |      derived session keys)          |
```

## SSLContext Setup

```java
import javax.net.ssl.*;
import java.security.*;
import java.io.FileInputStream;

public class SSLContextSetup {

    // ✅ Proper SSLContext configuration
    public static SSLContext createSSLContext(String keystorePath, char[] keystorePass,
                                              String truststorePath, char[] truststorePass)
            throws Exception {

        // KeyManager: provides OUR certificate to remote parties
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(
            KeyManagerFactory.getDefaultAlgorithm());
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(keystorePath)) {
            keyStore.load(fis, keystorePass);
        }
        kmf.init(keyStore, keystorePass);

        // TrustManager: validates THEIR certificates
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        try (FileInputStream fis = new FileInputStream(truststorePath)) {
            trustStore.load(fis, truststorePass);
        }
        tmf.init(trustStore);

        // Build SSLContext
        SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
        sslContext.init(kmf.getKeyManagers(), tmf.getTrustManagers(), new SecureRandom());

        return sslContext;
    }
}
```

## TrustManager and KeyManager

```java
import javax.net.ssl.*;
import java.security.cert.X509Certificate;

public class TrustManagerDemo {

    // ❌ Anti-pattern: Trust-all TrustManager (NEVER use in production!)
    public static SSLContext createInsecureContext() throws Exception {
        TrustManager[] trustAll = new TrustManager[]{
            new X509TrustManager() {
                public X509Certificate[] getAcceptedIssuers() { return null; }
                public void checkClientTrusted(X509Certificate[] certs, String type) { }
                public void checkServerTrusted(X509Certificate[] certs, String type) { }
            }
        };

        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(null, trustAll, new java.security.SecureRandom());
        return ctx;
        // This disables ALL certificate validation - vulnerable to MITM attacks!
    }

    // ✅ Best practice: Custom TrustManager with logging
    public static X509TrustManager createLoggingTrustManager(
            TrustManagerFactory tmf) {
        X509TrustManager defaultTm = null;
        for (TrustManager tm : tmf.getTrustManagers()) {
            if (tm instanceof X509TrustManager) {
                defaultTm = (X509TrustManager) tm;
                break;
            }
        }

        final X509TrustManager delegate = defaultTm;

        return new X509TrustManager() {
            @Override
            public void checkServerTrusted(X509Certificate[] chain,
                                            String authType)
                    throws java.security.cert.CertificateException {
                System.out.println("Verifying server: " +
                    chain[0].getSubjectX500Principal());
                delegate.checkServerTrusted(chain, authType);
                System.out.println("Server certificate verified successfully");
            }

            @Override
            public void checkClientTrusted(X509Certificate[] chain,
                                            String authType)
                    throws java.security.cert.CertificateException {
                delegate.checkClientTrusted(chain, authType);
            }

            @Override
            public X509Certificate[] getAcceptedIssuers() {
                return delegate.getAcceptedIssuers();
            }
        };
    }
}
```

## Self-Signed Certificate Handling

```java
import javax.net.ssl.*;
import java.security.KeyStore;
import java.security.cert.X509Certificate;
import java.io.FileInputStream;

public class SelfSignedCertHandling {

    // ✅ Proper approach: Add self-signed cert to a custom truststore
    public static SSLContext trustSelfSigned(String certPath) throws Exception {
        // Load the self-signed certificate into a truststore
        KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
        trustStore.load(null, null);  // Empty truststore

        java.security.cert.CertificateFactory cf =
            java.security.cert.CertificateFactory.getInstance("X.509");

        try (FileInputStream fis = new FileInputStream(certPath)) {
            X509Certificate cert = (X509Certificate) cf.generateCertificate(fis);
            trustStore.setCertificateEntry("self-signed", cert);
        }

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(trustStore);

        SSLContext sslContext = SSLContext.getInstance("TLSv1.3");
        sslContext.init(null, tmf.getTrustManagers(), null);
        return sslContext;
    }
}
```

## Certificate Pinning

```java
import javax.net.ssl.*;
import java.security.MessageDigest;
import java.security.cert.X509Certificate;
import java.util.Base64;
import java.util.Set;

public class CertificatePinning {

    // ✅ Pin to specific certificate public key hashes
    private static final Set<String> PINNED_HASHES = Set.of(
        "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=",
        "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
    );

    public static X509TrustManager createPinningTrustManager(
            X509TrustManager delegate) {

        return new X509TrustManager() {
            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType)
                    throws java.security.cert.CertificateException {
                // First do normal validation
                delegate.checkServerTrusted(chain, authType);

                // Then verify the pin
                boolean pinMatch = false;
                for (X509Certificate cert : chain) {
                    String hash = getPublicKeyHash(cert);
                    if (PINNED_HASHES.contains("sha256/" + hash)) {
                        pinMatch = true;
                        break;
                    }
                }

                if (!pinMatch) {
                    throw new java.security.cert.CertificateException(
                        "Certificate pin verification failed");
                }
            }

            @Override
            public void checkClientTrusted(X509Certificate[] chain,
                    String authType) throws java.security.cert.CertificateException {
                delegate.checkClientTrusted(chain, authType);
            }

            @Override
            public X509Certificate[] getAcceptedIssuers() {
                return delegate.getAcceptedIssuers();
            }
        };
    }

    private static String getPublicKeyHash(X509Certificate cert) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(cert.getPublicKey().getEncoded());
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to compute certificate hash", e);
        }
    }
}
```

## TLS Versions and Cipher Suites

```java
import javax.net.ssl.*;

public class TLSConfiguration {

    // ✅ Best practice: Enforce modern TLS versions
    public static SSLSocket configureSocket(SSLSocket socket) {
        // Enable only TLS 1.2 and 1.3
        socket.setEnabledProtocols(new String[]{"TLSv1.3", "TLSv1.2"});

        // Restrict to strong cipher suites
        socket.setEnabledCipherSuites(new String[]{
            "TLS_AES_256_GCM_SHA384",             // TLS 1.3
            "TLS_AES_128_GCM_SHA256",             // TLS 1.3
            "TLS_CHACHA20_POLY1305_SHA256",        // TLS 1.3
            "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",   // TLS 1.2
            "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"    // TLS 1.2
        });

        return socket;
    }

    // ❌ Anti-pattern: Allowing deprecated protocols
    public static void insecureConfig(SSLSocket socket) {
        socket.setEnabledProtocols(new String[]{
            "SSLv3",     // POODLE vulnerability
            "TLSv1",     // Deprecated (RFC 8996)
            "TLSv1.1"    // Deprecated (RFC 8996)
        });
    }
}
```

## HttpsURLConnection

```java
import javax.net.ssl.HttpsURLConnection;
import javax.net.ssl.SSLContext;
import java.net.URL;
import java.io.*;

public class HttpsDemo {

    // ✅ HTTPS connection with proper SSL configuration
    public static String secureGet(String urlStr, SSLContext sslContext)
            throws Exception {
        URL url = new URL(urlStr);
        HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();

        conn.setSSLSocketFactory(sslContext.getSocketFactory());
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);

        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(conn.getInputStream()))) {
            StringBuilder response = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            return response.toString();
        } finally {
            conn.disconnect();
        }
    }
}
```

## Modern Java HttpClient with SSL

```java
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;
import javax.net.ssl.SSLContext;

public class ModernHttpClientSSL {

    // ✅ Java 11+ HttpClient with custom SSLContext
    public static String secureRequest(SSLContext sslContext) throws Exception {
        HttpClient client = HttpClient.newBuilder()
            .sslContext(sslContext)
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(java.time.Duration.ofSeconds(10))
            .build();

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/data"))
            .header("Accept", "application/json")
            .GET()
            .build();

        HttpResponse<String> response = client.send(
            request, HttpResponse.BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());
        System.out.println("TLS: " + response.sslSession()
            .map(s -> s.getProtocol()).orElse("unknown"));

        return response.body();
    }
}
```

## Quick Reference

| Component | Purpose |
|-----------|---------|
| `SSLContext` | Factory for SSL socket factories and engines |
| `KeyManager` | Manages OUR keys and certificates (client/server identity) |
| `TrustManager` | Decides which remote certificates to trust |
| `SSLSocketFactory` | Creates SSL-enabled sockets |
| `SSLParameters` | Configures protocols, cipher suites, SNI |
| `KeyStore` | Storage for keys and trusted certificates |

## Interview Tips

| Point | Detail |
|-------|--------|
| **SSLContext** | Central factory; configured with KeyManagers + TrustManagers |
| **TrustManager** | Validates remote certificates; NEVER disable in production |
| **TLS versions** | Use TLS 1.2/1.3 only; SSLv3, TLS 1.0/1.1 are deprecated |
| **Certificate pinning** | Compares server cert hash against known-good values |
| **Self-signed certs** | Add to custom truststore; don't disable validation |
| **HttpClient** | Java 11+ supports SSLContext natively with HTTP/2 |

---

[<< Previous: Q6 - SecureRandom](./q06-secure-random.md) | [Next: Q8 - Preventing Injection >>](./q08-preventing-injection.md)
