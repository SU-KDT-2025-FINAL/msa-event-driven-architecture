# 이벤트 보안 가이드

## 1. Kafka SSL/SASL 설정

### SSL 인증서 생성 및 구성
```bash
#!/bin/bash
# scripts/create-kafka-ssl-certs.sh

set -e

CERT_DIR="ssl-certs"
KEYSTORE_PASSWORD="kafka-keystore-password"
TRUSTSTORE_PASSWORD="kafka-truststore-password"
KEY_PASSWORD="kafka-key-password"
VALIDITY_DAYS=365

mkdir -p $CERT_DIR

# 1. CA(Certificate Authority) 생성
openssl req -new -x509 -keyout $CERT_DIR/ca-key -out $CERT_DIR/ca-cert \
    -days $VALIDITY_DAYS -passout pass:$KEY_PASSWORD -subj \
    "/C=KR/ST=Seoul/L=Seoul/O=ECommerce/OU=Security/CN=kafka-ca"

# 2. Kafka 브로커용 키스토어 생성
for broker in kafka-1 kafka-2 kafka-3; do
    # 키스토어 생성
    keytool -keystore $CERT_DIR/${broker}.server.keystore.jks \
        -alias ${broker} -validity $VALIDITY_DAYS -genkey \
        -keyalg RSA -keysize 2048 \
        -storepass $KEYSTORE_PASSWORD -keypass $KEY_PASSWORD \
        -dname "CN=${broker},OU=Security,O=ECommerce,L=Seoul,ST=Seoul,C=KR"
    
    # 인증서 서명 요청(CSR) 생성
    keytool -keystore $CERT_DIR/${broker}.server.keystore.jks \
        -alias ${broker} -certreq -file $CERT_DIR/${broker}.csr \
        -storepass $KEYSTORE_PASSWORD
    
    # CA로 인증서 서명
    openssl x509 -req -CA $CERT_DIR/ca-cert -CAkey $CERT_DIR/ca-key \
        -in $CERT_DIR/${broker}.csr -out $CERT_DIR/${broker}-signed.cert \
        -days $VALIDITY_DAYS -CAcreateserial -passin pass:$KEY_PASSWORD
    
    # 트러스트스토어 생성 및 CA 인증서 추가
    keytool -keystore $CERT_DIR/${broker}.server.truststore.jks \
        -alias ca-cert -import -file $CERT_DIR/ca-cert \
        -storepass $TRUSTSTORE_PASSWORD -noprompt
    
    # 키스토어에 CA 인증서 추가
    keytool -keystore $CERT_DIR/${broker}.server.keystore.jks \
        -alias ca-cert -import -file $CERT_DIR/ca-cert \
        -storepass $KEYSTORE_PASSWORD -noprompt
    
    # 키스토어에 서명된 인증서 추가
    keytool -keystore $CERT_DIR/${broker}.server.keystore.jks \
        -alias ${broker} -import -file $CERT_DIR/${broker}-signed.cert \
        -storepass $KEYSTORE_PASSWORD -noprompt
done

# 3. 클라이언트용 키스토어 생성
keytool -keystore $CERT_DIR/client.keystore.jks \
    -alias client -validity $VALIDITY_DAYS -genkey \
    -keyalg RSA -keysize 2048 \
    -storepass $KEYSTORE_PASSWORD -keypass $KEY_PASSWORD \
    -dname "CN=kafka-client,OU=Security,O=ECommerce,L=Seoul,ST=Seoul,C=KR"

# CSR 생성 및 서명
keytool -keystore $CERT_DIR/client.keystore.jks \
    -alias client -certreq -file $CERT_DIR/client.csr \
    -storepass $KEYSTORE_PASSWORD

openssl x509 -req -CA $CERT_DIR/ca-cert -CAkey $CERT_DIR/ca-key \
    -in $CERT_DIR/client.csr -out $CERT_DIR/client-signed.cert \
    -days $VALIDITY_DAYS -CAcreateserial -passin pass:$KEY_PASSWORD

# 클라이언트 트러스트스토어 생성
keytool -keystore $CERT_DIR/client.truststore.jks \
    -alias ca-cert -import -file $CERT_DIR/ca-cert \
    -storepass $TRUSTSTORE_PASSWORD -noprompt

# 클라이언트 키스토어에 인증서 추가
keytool -keystore $CERT_DIR/client.keystore.jks \
    -alias ca-cert -import -file $CERT_DIR/ca-cert \
    -storepass $KEYSTORE_PASSWORD -noprompt

keytool -keystore $CERT_DIR/client.keystore.jks \
    -alias client -import -file $CERT_DIR/client-signed.cert \
    -storepass $KEYSTORE_PASSWORD -noprompt

echo "SSL certificates created successfully in $CERT_DIR directory"
```

### Kafka 브로커 SSL 설정
```properties
# server.properties - SSL 구성
listeners=PLAINTEXT://localhost:9092,SSL://localhost:9093
advertised.listeners=PLAINTEXT://localhost:9092,SSL://localhost:9093
listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL

# SSL 키스토어 설정
ssl.keystore.location=/opt/kafka/ssl-certs/kafka-1.server.keystore.jks
ssl.keystore.password=kafka-keystore-password
ssl.key.password=kafka-key-password

# SSL 트러스트스토어 설정
ssl.truststore.location=/opt/kafka/ssl-certs/kafka-1.server.truststore.jks
ssl.truststore.password=kafka-truststore-password

# 클라이언트 인증 필수 설정
ssl.client.auth=required

# SSL 프로토콜 설정
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

# SSL 엔드포인트 식별 알고리즘
ssl.endpoint.identification.algorithm=HTTPS
```

### SASL 인증 설정
```properties
# server.properties - SASL 구성
listeners=SASL_SSL://localhost:9094
advertised.listeners=SASL_SSL://localhost:9094
listener.security.protocol.map=SASL_SSL:SASL_SSL

# SASL 메커니즘 설정
sasl.enabled.mechanisms=SCRAM-SHA-512,PLAIN
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512

# JAAS 설정 파일 경로
java.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf

# 스키마 레지스트리와의 통신을 위한 설정
listener.name.sasl_ssl.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
```

### JAAS 설정 파일
```java
// kafka_server_jaas.conf
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret";
};

Client {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin-secret";
};
```

## 2. 이벤트 암호화 및 서명

### 메시지 레벨 암호화
```java
@Component
public class EventEncryptionService {
    
    private final AESUtil aesUtil;
    private final RSAUtil rsaUtil;
    private final ObjectMapper objectMapper;
    
    @Value("${encryption.secret.key}")
    private String secretKey;
    
    public EventEncryptionService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        this.aesUtil = new AESUtil();
        this.rsaUtil = new RSAUtil();
    }
    
    public EncryptedEvent encryptEvent(DomainEvent event) throws Exception {
        // 1. 이벤트를 JSON으로 직렬화
        String eventJson = objectMapper.writeValueAsString(event);
        
        // 2. AES 키 생성
        SecretKey aesKey = aesUtil.generateKey();
        
        // 3. 이벤트 데이터를 AES로 암호화
        byte[] encryptedData = aesUtil.encrypt(eventJson.getBytes(StandardCharsets.UTF_8), aesKey);
        
        // 4. AES 키를 RSA 공개키로 암호화
        byte[] encryptedAesKey = rsaUtil.encrypt(aesKey.getEncoded(), getRSAPublicKey());
        
        // 5. 디지털 서명 생성
        String signature = createDigitalSignature(eventJson);
        
        return EncryptedEvent.builder()
            .eventType(event.getClass().getSimpleName())
            .encryptedData(Base64.getEncoder().encodeToString(encryptedData))
            .encryptedKey(Base64.getEncoder().encodeToString(encryptedAesKey))
            .signature(signature)
            .algorithm("AES-256-GCM")
            .keyAlgorithm("RSA-2048")
            .timestamp(Instant.now())
            .build();
    }
    
    public <T extends DomainEvent> T decryptEvent(EncryptedEvent encryptedEvent, 
                                                  Class<T> eventClass) throws Exception {
        // 1. RSA 개인키로 AES 키 복호화
        byte[] encryptedAesKey = Base64.getDecoder().decode(encryptedEvent.getEncryptedKey());
        byte[] aesKeyBytes = rsaUtil.decrypt(encryptedAesKey, getRSAPrivateKey());
        SecretKey aesKey = new SecretKeySpec(aesKeyBytes, "AES");
        
        // 2. AES 키로 이벤트 데이터 복호화
        byte[] encryptedData = Base64.getDecoder().decode(encryptedEvent.getEncryptedData());
        byte[] decryptedData = aesUtil.decrypt(encryptedData, aesKey);
        String eventJson = new String(decryptedData, StandardCharsets.UTF_8);
        
        // 3. 디지털 서명 검증
        if (!verifyDigitalSignature(eventJson, encryptedEvent.getSignature())) {
            throw new SecurityException("Digital signature verification failed");
        }
        
        // 4. JSON을 이벤트 객체로 역직렬화
        return objectMapper.readValue(eventJson, eventClass);
    }
    
    private String createDigitalSignature(String data) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initSign(getRSAPrivateKey());
        signature.update(data.getBytes(StandardCharsets.UTF_8));
        byte[] signatureBytes = signature.sign();
        return Base64.getEncoder().encodeToString(signatureBytes);
    }
    
    private boolean verifyDigitalSignature(String data, String signatureStr) throws Exception {
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initVerify(getRSAPublicKey());
        signature.update(data.getBytes(StandardCharsets.UTF_8));
        byte[] signatureBytes = Base64.getDecoder().decode(signatureStr);
        return signature.verify(signatureBytes);
    }
    
    private PublicKey getRSAPublicKey() throws Exception {
        // 공개키 로드 (파일, 키 저장소, 또는 환경변수에서)
        byte[] keyBytes = Base64.getDecoder().decode(loadPublicKeyString());
        X509EncodedKeySpec spec = new X509EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePublic(spec);
    }
    
    private PrivateKey getRSAPrivateKey() throws Exception {
        // 개인키 로드 (보안 키 저장소에서)
        byte[] keyBytes = Base64.getDecoder().decode(loadPrivateKeyString());
        PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePrivate(spec);
    }
}
```

### AES 암호화 유틸리티
```java
@Component
public class AESUtil {
    
    private static final String ALGORITHM = "AES";
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 16;
    
    public SecretKey generateKey() throws NoSuchAlgorithmException {
        KeyGenerator keyGenerator = KeyGenerator.getInstance(ALGORITHM);
        keyGenerator.init(256);
        return keyGenerator.generateKey();
    }
    
    public byte[] encrypt(byte[] data, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        
        // IV(Initialization Vector) 생성
        byte[] iv = new byte[GCM_IV_LENGTH];
        SecureRandom.getInstanceStrong().nextBytes(iv);
        GCMParameterSpec gcmParameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
        
        cipher.init(Cipher.ENCRYPT_MODE, key, gcmParameterSpec);
        byte[] encryptedData = cipher.doFinal(data);
        
        // IV와 암호화된 데이터를 결합
        byte[] result = new byte[GCM_IV_LENGTH + encryptedData.length];
        System.arraycopy(iv, 0, result, 0, GCM_IV_LENGTH);
        System.arraycopy(encryptedData, 0, result, GCM_IV_LENGTH, encryptedData.length);
        
        return result;
    }
    
    public byte[] decrypt(byte[] encryptedData, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        
        // IV 추출
        byte[] iv = new byte[GCM_IV_LENGTH];
        System.arraycopy(encryptedData, 0, iv, 0, GCM_IV_LENGTH);
        GCMParameterSpec gcmParameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
        
        cipher.init(Cipher.DECRYPT_MODE, key, gcmParameterSpec);
        
        // 암호화된 데이터 추출 및 복호화
        byte[] cipherText = new byte[encryptedData.length - GCM_IV_LENGTH];
        System.arraycopy(encryptedData, GCM_IV_LENGTH, cipherText, 0, cipherText.length);
        
        return cipher.doFinal(cipherText);
    }
}
```

### 암호화된 이벤트 DTO
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class EncryptedEvent {
    
    private String eventType;
    private String encryptedData;
    private String encryptedKey;
    private String signature;
    private String algorithm;
    private String keyAlgorithm;
    private Instant timestamp;
    private String keyId; // 키 교체를 위한 키 식별자
    
    // 암호화 메타데이터
    private Map<String, String> encryptionMetadata;
}
```

## 3. 마이크로서비스 간 인증/인가

### JWT 기반 서비스 간 인증
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Bean
    public JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint() {
        return new JwtAuthenticationEntryPoint();
    }
    
    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter() {
        return new JwtAuthenticationFilter();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeHttpRequests(authz -> authz
                // 공개 엔드포인트
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/auth/login", "/auth/register").permitAll()
                
                // 관리자 전용 엔드포인트
                .requestMatchers("/admin/**").hasRole("ADMIN")
                
                // 서비스 간 통신 엔드포인트
                .requestMatchers("/internal/**").hasRole("SERVICE")
                
                // 사용자 엔드포인트
                .requestMatchers("/orders/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/payments/**").hasAnyRole("USER", "ADMIN")
                
                .anyRequest().authenticated()
            )
            .exceptionHandling()
                .authenticationEntryPoint(jwtAuthenticationEntryPoint())
            .and()
            .addFilterBefore(jwtAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
}

@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private Long jwtExpiration;
    
    @Value("${jwt.service.expiration}")
    private Long serviceTokenExpiration;
    
    public String generateUserToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);
        
        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()))
            .claim("type", "USER")
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public String generateServiceToken(String serviceName, List<String> permissions) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + serviceTokenExpiration);
        
        return Jwts.builder()
            .setSubject(serviceName)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .claim("type", "SERVICE")
            .claim("permissions", permissions)
            .claim("roles", List.of("ROLE_SERVICE"))
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public Claims getClaimsFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            log.error("Invalid JWT token: {}", e.getMessage());
            return false;
        }
    }
}
```

### OAuth 2.0 / OpenID Connect 통합
```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
    
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .requestMatchers("/internal/**").access("#oauth2.hasScope('service')")
            .requestMatchers("/orders/**").access("#oauth2.hasScope('read') or #oauth2.hasScope('write')")
            .requestMatchers("/admin/**").access("#oauth2.hasScope('admin')")
            .anyRequest().authenticated();
    }
    
    @Bean
    public RemoteTokenServices tokenServices() {
        RemoteTokenServices tokenServices = new RemoteTokenServices();
        tokenServices.setCheckTokenEndpointUrl("http://auth-server:8080/oauth/check_token");
        tokenServices.setClientId("order-service");
        tokenServices.setClientSecret("order-service-secret");
        return tokenServices;
    }
}

@Service
public class ServiceTokenService {
    
    private final OAuth2RestTemplate oauth2RestTemplate;
    
    @Bean
    public OAuth2RestTemplate oauth2RestTemplate() {
        ClientCredentialsResourceDetails resourceDetails = new ClientCredentialsResourceDetails();
        resourceDetails.setId("order-service");
        resourceDetails.setClientId("order-service");
        resourceDetails.setClientSecret("order-service-secret");
        resourceDetails.setAccessTokenUri("http://auth-server:8080/oauth/token");
        resourceDetails.setScope(List.of("service", "read", "write"));
        
        return new OAuth2RestTemplate(resourceDetails);
    }
    
    public String callPaymentService(PaymentRequest request) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        
        HttpEntity<PaymentRequest> entity = new HttpEntity<>(request, headers);
        
        ResponseEntity<String> response = oauth2RestTemplate.exchange(
            "http://payment-service:8080/internal/payments",
            HttpMethod.POST,
            entity,
            String.class
        );
        
        return response.getBody();
    }
}
```

## 4. API Gateway 접근 제어

### Spring Cloud Gateway 보안 설정
```java
@Configuration
@EnableWebFluxSecurity
public class GatewaySecurityConfig {
    
    private final JwtAuthenticationManager authenticationManager;
    private final SecurityContextRepository securityContextRepository;
    
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
            .csrf().disable()
            .formLogin().disable()
            .httpBasic().disable()
            .authenticationManager(authenticationManager)
            .securityContextRepository(securityContextRepository)
            .authorizeExchange(exchanges -> exchanges
                // 공개 경로
                .pathMatchers("/auth/**", "/actuator/health").permitAll()
                
                // 관리자 전용
                .pathMatchers("/admin/**").hasRole("ADMIN")
                
                // 사용자 인증 필요
                .pathMatchers("/orders/**", "/payments/**").hasAnyRole("USER", "ADMIN")
                
                // 서비스 간 통신
                .pathMatchers("/internal/**").hasRole("SERVICE")
                
                .anyExchange().authenticated()
            )
            .exceptionHandling()
                .authenticationEntryPoint((exchange, ex) -> {
                    ServerHttpResponse response = exchange.getResponse();
                    response.setStatusCode(HttpStatus.UNAUTHORIZED);
                    return response.setComplete();
                })
            .and()
            .build();
    }
}

@Component
public class JwtAuthenticationManager implements ReactiveAuthenticationManager {
    
    private final JwtTokenProvider tokenProvider;
    
    @Override
    public Mono<Authentication> authenticate(Authentication authentication) {
        String authToken = authentication.getCredentials().toString();
        
        try {
            if (tokenProvider.validateToken(authToken)) {
                Claims claims = tokenProvider.getClaimsFromToken(authToken);
                
                List<SimpleGrantedAuthority> authorities = ((List<String>) claims.get("roles"))
                    .stream()
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toList());
                
                return Mono.just(new UsernamePasswordAuthenticationToken(
                    claims.getSubject(),
                    null,
                    authorities
                ));
            }
        } catch (Exception e) {
            return Mono.empty();
        }
        
        return Mono.empty();
    }
}
```

### Rate Limiting 구현
```java
@Configuration
public class RateLimitingConfig {
    
    @Bean
    public RedisRateLimiter userRateLimiter() {
        return new RedisRateLimiter(
            10,  // replenishRate: 초당 토큰 보충량
            20,  // burstCapacity: 최대 토큰 용량
            1    // requestedTokens: 요청당 소모 토큰
        );
    }
    
    @Bean
    public RedisRateLimiter adminRateLimiter() {
        return new RedisRateLimiter(100, 200, 1);
    }
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-orders", r -> r.path("/orders/**")
                .and().method(HttpMethod.POST, HttpMethod.PUT)
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(userRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    )
                )
                .uri("http://order-service:8080")
            )
            .route("admin-routes", r -> r.path("/admin/**")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(adminRateLimiter())
                        .setKeyResolver(userKeyResolver())
                    )
                )
                .uri("http://admin-service:8080")
            )
            .build();
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            // JWT 토큰에서 사용자 ID 추출
            return exchange.getRequest().getHeaders().getFirst("Authorization")
                .map(authHeader -> {
                    if (authHeader.startsWith("Bearer ")) {
                        String token = authHeader.substring(7);
                        try {
                            Claims claims = tokenProvider.getClaimsFromToken(token);
                            return Mono.just(claims.getSubject());
                        } catch (Exception e) {
                            return Mono.just("anonymous");
                        }
                    }
                    return Mono.just("anonymous");
                })
                .orElse(Mono.just("anonymous"));
        };
    }
}
```

## 5. 보안 감사 및 로깅

### 보안 이벤트 로깅
```java
@Component
@Slf4j
public class SecurityAuditLogger {
    
    private final ApplicationEventPublisher eventPublisher;
    private final ObjectMapper objectMapper;
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        SecurityAuditEvent auditEvent = SecurityAuditEvent.builder()
            .eventType("AUTHENTICATION_SUCCESS")
            .principal(event.getAuthentication().getName())
            .source(getClientInfo(event))
            .timestamp(Instant.now())
            .details(Map.of(
                "authorities", event.getAuthentication().getAuthorities().toString(),
                "authType", event.getAuthentication().getClass().getSimpleName()
            ))
            .build();
        
        logSecurityEvent(auditEvent);
        eventPublisher.publishEvent(auditEvent);
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        SecurityAuditEvent auditEvent = SecurityAuditEvent.builder()
            .eventType("AUTHENTICATION_FAILURE")
            .principal(event.getAuthentication().getName())
            .source(getClientInfo(event))
            .timestamp(Instant.now())
            .details(Map.of(
                "failureReason", event.getException().getMessage(),
                "attemptedCredentials", event.getAuthentication().getCredentials() != null ? "[REDACTED]" : "null"
            ))
            .severity("HIGH")
            .build();
        
        logSecurityEvent(auditEvent);
        eventPublisher.publishEvent(auditEvent);
    }
    
    @EventListener
    public void handleAccessDenied(AccessDeniedEvent event) {
        SecurityAuditEvent auditEvent = SecurityAuditEvent.builder()
            .eventType("ACCESS_DENIED")
            .principal(event.getAuthentication().getName())
            .timestamp(Instant.now())
            .details(Map.of(
                "resource", event.getSource().toString(),
                "authorities", event.getAuthentication().getAuthorities().toString()
            ))
            .severity("MEDIUM")
            .build();
        
        logSecurityEvent(auditEvent);
        eventPublisher.publishEvent(auditEvent);
    }
    
    public void logPrivilegedOperation(String operation, String principal, 
                                     Object target, Map<String, Object> context) {
        SecurityAuditEvent auditEvent = SecurityAuditEvent.builder()
            .eventType("PRIVILEGED_OPERATION")
            .principal(principal)
            .timestamp(Instant.now())
            .details(Map.of(
                "operation", operation,
                "target", target.toString(),
                "context", context
            ))
            .severity("LOW")
            .build();
        
        logSecurityEvent(auditEvent);
        eventPublisher.publishEvent(auditEvent);
    }
    
    private void logSecurityEvent(SecurityAuditEvent event) {
        try {
            // 구조화된 로그 출력
            String logMessage = objectMapper.writeValueAsString(Map.of(
                "logType", "SECURITY_AUDIT",
                "event", event
            ));
            
            switch (event.getSeverity()) {
                case "HIGH" -> log.error("Security Event: {}", logMessage);
                case "MEDIUM" -> log.warn("Security Event: {}", logMessage);
                default -> log.info("Security Event: {}", logMessage);
            }
            
        } catch (Exception e) {
            log.error("Failed to log security event", e);
        }
    }
    
    private String getClientInfo(ApplicationEvent event) {
        // HTTP 요청에서 클라이언트 정보 추출
        HttpServletRequest request = getCurrentHttpRequest();
        if (request != null) {
            return Map.of(
                "clientIp", getClientIpAddress(request),
                "userAgent", request.getHeader("User-Agent"),
                "requestUri", request.getRequestURI()
            ).toString();
        }
        return "Unknown";
    }
    
    private String getClientIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        
        return request.getRemoteAddr();
    }
}
```

### 보안 메트릭 수집
```java
@Component
public class SecurityMetrics {
    
    private final Counter authenticationAttempts;
    private final Counter authenticationFailures;
    private final Counter accessDeniedEvents;
    private final Timer authenticationDuration;
    private final Gauge activeSessions;
    
    public SecurityMetrics(MeterRegistry meterRegistry) {
        this.authenticationAttempts = Counter.builder("security.authentication.attempts")
            .description("Total authentication attempts")
            .register(meterRegistry);
            
        this.authenticationFailures = Counter.builder("security.authentication.failures")
            .description("Failed authentication attempts")
            .register(meterRegistry);
            
        this.accessDeniedEvents = Counter.builder("security.access.denied")
            .description("Access denied events")
            .register(meterRegistry);
            
        this.authenticationDuration = Timer.builder("security.authentication.duration")
            .description("Authentication processing time")
            .register(meterRegistry);
            
        this.activeSessions = Gauge.builder("security.sessions.active")
            .description("Active user sessions")
            .register(meterRegistry, this, this::getActiveSessionCount);
    }
    
    @EventListener
    public void handleAuthenticationAttempt(AuthenticationEvent event) {
        authenticationAttempts.increment(
            Tags.of("type", event.getAuthenticationType())
        );
    }
    
    @EventListener
    public void handleAuthenticationFailure(AuthenticationFailureEvent event) {
        authenticationFailures.increment(
            Tags.of(
                "reason", event.getFailureReason(),
                "type", event.getAuthenticationType()
            )
        );
    }
    
    @EventListener
    public void handleAccessDenied(AccessDeniedEvent event) {
        accessDeniedEvents.increment(
            Tags.of("resource", event.getResource())
        );
    }
    
    private double getActiveSessionCount(SecurityMetrics metrics) {
        // 활성 세션 수 계산 로직
        return sessionRegistry.getAllPrincipals().size();
    }
}
```

## 실습 과제

1. **완전한 보안 설정**: Kafka SSL/SASL, 서비스 간 인증, API Gateway 보안을 포함한 전체 보안 구성
2. **메시지 레벨 암호화**: 민감한 이벤트 데이터의 종단간 암호화 구현
3. **보안 감사 시스템**: 모든 보안 이벤트를 추적하고 분석하는 감사 시스템 구축
4. **침입 탐지 시스템**: 비정상적인 접근 패턴을 감지하는 실시간 모니터링 시스템 구현
5. **보안 테스트**: 취약점 스캔, 침투 테스트, 보안 회귀 테스트 자동화

## 참고 자료

- [Kafka Security Documentation](https://kafka.apache.org/documentation/#security)
- [Spring Security Documentation](https://docs.spring.io/spring-security/reference/)
- [OAuth 2.0 and OpenID Connect](https://tools.ietf.org/html/rfc6749)
- [OWASP Security Guidelines](https://owasp.org/www-project-top-ten/)