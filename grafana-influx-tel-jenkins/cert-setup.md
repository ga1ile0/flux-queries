# SSL Certificate Setup Guide:  App A to App B Communication

## Overview

This guide covers how to configure SSL certificates so that App B can securely communicate with App A over HTTPS.

- **App A** runs at:  `https://qmmetrics.testregion.com:port` (has SSL certificate in keystore)
- **App B** runs at: `https://<servername>. prod.com: port` (needs to trust App A's certificate)
- **Communication**:  App B sends POST requests to App A

## Prerequisites

- Java JDK installed (for `keytool` command)
- App A's keystore file (`.p12` format)
- Access to both App A and App B configuration

---

## Step 1: Create App B's Truststore

### 1.1 List the keystore to find the alias

```bash
keytool -list -keystore appA-keystore.p12 -storetype PKCS12 -storepass <your-keystore-password>
```

**Note the alias name** - you'll need it for the next step.

### 1.2 Export the certificate from App A's keystore

```bash
keytool -exportcert -keystore appA-keystore.p12 -storetype PKCS12 -alias <alias-from-step-1> -file appA-cert.crt -storepass <your-keystore-password>
```

This creates a `appA-cert.crt` file. 

### 1.3 Create App B's truststore and import the certificate

```bash
keytool -importcert -file appA-cert.crt -alias appA -keystore appB-truststore.p12 -storetype PKCS12 -storepass <choose-truststore-password> -noverify
```

When prompted "Trust this certificate?  [no]:", type **yes** and press Enter.

### 1.4 Verify the truststore was created correctly

```bash
keytool -list -keystore appB-truststore.p12 -storetype PKCS12 -storepass <your-truststore-password>
```

You should see an entry for "appA".

---

## Step 2: Configure App B

### 2.1 Place the truststore file

- **Option A**: Place `appB-truststore.p12` in `src/main/resources` folder (for classpath access)
- **Option B**: Place it in a secure location on the server (for file system access)

### 2.2 Update application.properties

Add these lines to App B's `application.properties`:

```properties
# SSL Truststore configuration for App B
server.ssl.trust-store=classpath:appB-truststore. p12
server.ssl.trust-store-password=<your-truststore-password>
server. ssl.trust-store-type=PKCS12

# Alternative if the truststore is NOT in classpath (external file):
# server.ssl.trust-store=file:C:/path/to/appB-truststore.p12
```

---

## Step 3: Configure RestTemplate (Required for App B)

Since App B uses `RestTemplate`, you need to configure it to use the truststore.  Choose one of the following options:

### Option 1: Custom RestTemplate Bean (Recommended)

#### 3.1.1 Add Maven Dependency

Add to `pom.xml`:

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.14</version>
</dependency>
```

Or for Gradle (`build.gradle`):

```gradle
implementation 'org.apache. httpcomponents:httpclient:4.5.14'
```

#### 3.1.2 Create RestTemplate Configuration Class

Create a new file: `RestTemplateConfig.java`

```java
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache. http.impl.client.HttpClients;
import org.apache. http.ssl.SSLContextBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

import javax.net. ssl.SSLContext;

@Configuration
public class RestTemplateConfig {

    @Value("${server.ssl.trust-store}")
    private Resource trustStore;

    @Value("${server.ssl.trust-store-password}")
    private String trustStorePassword;

    @Bean
    public RestTemplate restTemplate() throws Exception {
        SSLContext sslContext = SSLContextBuilder
                .create()
                .loadTrustMaterial(trustStore.getURL(), trustStorePassword.toCharArray())
                .build();

        SSLConnectionSocketFactory socketFactory = new SSLConnectionSocketFactory(sslContext);

        CloseableHttpClient httpClient = HttpClients.custom()
                .setSSLSocketFactory(socketFactory)
                .build();

        HttpComponentsClientHttpRequestFactory factory = 
                new HttpComponentsClientHttpRequestFactory(httpClient);

        return new RestTemplate(factory);
    }
}
```

#### 3.1.3 Use RestTemplate in Your Service

```java
@Service
public class MyService {

    private final RestTemplate restTemplate;

    @Autowired
    public MyService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public String callAppA() {
        String url = "https://qmmetrics.testregion.com:port/api/endpoint";
        
        // Example POST request
        ResponseEntity<String> response = restTemplate.postForEntity(
            url, 
            requestBody, 
            String.class
        );
        
        return response.getBody();
    }
}
```

---

### Option 2: JVM System Properties (Simpler Alternative)

If you prefer a simpler approach without custom configuration:

#### 3.2.1 Programmatic Configuration (Recommended within Option 2)

Modify your main application class:

```java
import org.springframework.boot.SpringApplication;
import org. springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AppBApplication {

    public static void main(String[] args) {
        // Set system properties before Spring Boot starts
        System.setProperty("javax.net. ssl.trustStore", "C:/path/to/appB-truststore.p12");
        System.setProperty("javax.net. ssl.trustStorePassword", "<your-truststore-password>");
        System.setProperty("javax. net.ssl.trustStoreType", "PKCS12");
        
        SpringApplication.run(AppBApplication.class, args);
    }
}
```

#### 3.2.2 Use RestTemplate Normally

```java
@Service
public class MyService {

    private final RestTemplate restTemplate = new RestTemplate();

    public String callAppA() {
        String url = "https://qmmetrics.testregion.com:port/api/endpoint";
        
        // Example POST request
        ResponseEntity<String> response = restTemplate.postForEntity(
            url, 
            requestBody, 
            String.class
        );
        
        return response.getBody();
    }
}
```

#### 3.2.3 Alternative: Startup Arguments

Or pass as JVM arguments when starting the application:

```bash
java -Djavax.net.ssl.trustStore=C:/path/to/appB-truststore. p12 \
     -Djavax.net. ssl.trustStorePassword=<your-truststore-password> \
     -Djavax. net.ssl.trustStoreType=PKCS12 \
     -jar appB.jar
```

---

## Complete Example

Let's say: 
- App A keystore password: `keystorePass123`
- Alias in keystore: `appakey`
- Truststore password: `truststorePass456`
- Files location: `C:/certs/`

### Commands

```bash
# Navigate to certs directory
cd C:/certs/

# Step 1: Check the alias
keytool -list -keystore appA-keystore.p12 -storetype PKCS12 -storepass keystorePass123

# Step 2: Export the certificate
keytool -exportcert -keystore appA-keystore.p12 -storetype PKCS12 -alias appakey -file appA-cert.crt -storepass keystorePass123

# Step 3: Create truststore
keytool -importcert -file appA-cert.crt -alias appA -keystore appB-truststore.p12 -storetype PKCS12 -storepass truststorePass456 -noverify

# Step 4: Verify
keytool -list -keystore appB-truststore.p12 -storetype PKCS12 -storepass truststorePass456
```

### application.properties

```properties
server.ssl.trust-store=classpath:appB-truststore. p12
server.ssl.trust-store-password=truststorePass456
server.ssl.trust-store-type=PKCS12
```

---

## Which RestTemplate Option Should You Choose?

**Use Option 1 (Custom RestTemplate Bean)** if:
- ✅ You want more control and flexibility
- ✅ You might have multiple RestTemplate beans with different configurations
- ✅ You prefer Spring-managed beans

**Use Option 2 (JVM Properties)** if:
- ✅ You want a simpler solution
- ✅ All SSL connections in your app should use the same truststore
- ✅ You don't want to add extra dependencies

---

## Troubleshooting

### Certificate hostname mismatch error

**Error**: `java.security.cert.CertificateException: No name matching qmmetrics.testregion. com found`

**Solution**: Verify that App A's certificate was issued for the correct domain. Check with: 

```bash
keytool -list -v -keystore appA-keystore.p12 -storetype PKCS12 -storepass <password>
```

Look for the `Owner` (CN field) and `SubjectAlternativeName` - they should match the URL App B uses to connect.

### Keytool not found

**Solution**: Make sure Java JDK is installed and `keytool` is in your PATH.  Or use the full path:

```bash
"C:/Program Files/Java/jdk-17/bin/keytool" -version
```

### Connection still failing

1.  Verify the truststore contains the certificate: 
   ```bash
   keytool -list -keystore appB-truststore.p12 -storetype PKCS12 -storepass <password>
   ```

2. Check that App B is actually using the truststore (check logs for SSL errors)

3. Verify the URL App B uses to connect to App A matches the certificate's CN/SAN

4. Enable SSL debug logging in App B: 
   ```properties
   logging.level.javax.net.ssl=DEBUG
   ```

---

## Security Notes

- ⚠️ Never commit passwords or keystores/truststores to version control
- ⚠️ Use environment variables or secure vaults for passwords in production
- ⚠️ Self-signed certificates are fine for testing, but use CA-signed certificates in production
- ⚠️ Keep your truststore file secure with appropriate file permissions

---

## Summary

✅ You only need to configure App B's truststore (not App A's)  
✅ App A doesn't need any changes - it only responds to requests  
✅ This is one-way SSL (not mutual TLS)  
✅ The certificate must match the hostname App B uses to connect to App A