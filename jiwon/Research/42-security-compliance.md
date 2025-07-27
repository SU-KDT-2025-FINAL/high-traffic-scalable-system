# 4.2 보안과 컴플라이언스

## Overview
보안과 컴플라이언스는 고트래픽 시스템에서 데이터 보호와 규정 준수를 보장하는 핵심 요소입니다. **제로 트러스트 보안 모델**을 기반으로 **API 보안**, **데이터 암호화**, **컴플라이언스 준수**를 구현하여 안전한 시스템을 구축할 수 있습니다.

## 제로 트러스트 보안 모델

### 기본 원칙
```
Zero Trust 원칙:
1. 신뢰하지 말고 검증하라 (Never Trust, Always Verify)
2. 최소 권한 원칙 (Principle of Least Privilege)
3. 모든 트래픽을 검사하라 (Inspect All Traffic)
4. 지속적인 모니터링 (Continuous Monitoring)
```

### 아키텍처 구현
```
┌─ Identity Provider ─┐    ┌─ Policy Engine ─┐
│ - Multi-factor Auth │    │ - Access Policies│
│ - Identity Verification│    │ - Risk Assessment│
└─────────────────────┘    └──────────────────┘
         │                          │
         └──────────┬─────────────────┘
                    │
┌─ Network Security ▼─────────────────────────┐
│ ┌─ Micro-segmentation ─┐ ┌─ Traffic Inspection ─┐ │
│ │ - Service Mesh      │ │ - WAF                │ │
│ │ - Network Policies  │ │ - DDoS Protection    │ │
│ └─────────────────────┘ └──────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## API 보안

### OAuth 2.0 + OpenID Connect
```java
@Configuration
@EnableWebSecurity
public class SecurityConfiguration {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/orders/**").hasAnyRole("USER", "PREMIUM")
                .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("MANAGER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtDecoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf.disable())
            .headers(headers -> headers
                .frameOptions().deny()
                .contentTypeOptions().and()
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .includeSubdomains(true)
                    .maxAgeInSeconds(31536000)
                )
            );
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withJwkSetUri("https://auth.ecommerce.com/.well-known/jwks.json")
            .cache(Duration.ofMinutes(5))
            .build();
        
        decoder.setJwtValidator(jwtValidator());
        return decoder;
    }
    
    @Bean
    public Converter<Jwt, AbstractAuthenticationToken> jwtAuthConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> roles = jwt.getClaimAsStringList("roles");
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

### API Rate Limiting
```java
@Component
public class RateLimitingFilter implements Filter {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final RateLimitConfig config;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        
        String clientId = extractClientId(httpRequest);
        String endpoint = httpRequest.getRequestURI();
        
        RateLimitRule rule = config.getRuleForEndpoint(endpoint);
        if (rule != null && !isRateLimitAllowed(clientId, endpoint, rule)) {
            httpResponse.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            httpResponse.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return;
        }
        
        chain.doFilter(request, response);
    }
    
    private boolean isRateLimitAllowed(String clientId, String endpoint, RateLimitRule rule) {
        String key = "rate_limit:" + clientId + ":" + endpoint;
        String windowKey = key + ":" + getCurrentWindow(rule.getWindowSize());
        
        try {
            String countStr = redisTemplate.opsForValue().get(windowKey);
            int currentCount = countStr != null ? Integer.parseInt(countStr) : 0;
            
            if (currentCount >= rule.getLimit()) {
                return false;
            }
            
            // 카운터 증가
            redisTemplate.opsForValue().increment(windowKey);
            redisTemplate.expire(windowKey, Duration.ofSeconds(rule.getWindowSize()));
            
            return true;
            
        } catch (Exception e) {
            log.error("Rate limiting error", e);
            return true; // 에러 시 허용 (fail-open)
        }
    }
    
    private long getCurrentWindow(int windowSizeSeconds) {
        return System.currentTimeMillis() / (windowSizeSeconds * 1000);
    }
}
```

### API Key 관리
```java
@Service
public class ApiKeyService {
    
    private final ApiKeyRepository apiKeyRepository;
    private final EncryptionService encryptionService;
    
    public ApiKey generateApiKey(String clientName, Set<String> scopes) {
        String keyValue = generateSecureKey();
        String hashedKey = hashApiKey(keyValue);
        
        ApiKey apiKey = ApiKey.builder()
            .id(UUID.randomUUID().toString())
            .clientName(clientName)
            .hashedKey(hashedKey)
            .scopes(scopes)
            .status(ApiKeyStatus.ACTIVE)
            .createdAt(Instant.now())
            .expiresAt(Instant.now().plus(Duration.ofDays(365)))
            .rateLimit(RateLimit.builder()
                .requestsPerMinute(1000)
                .requestsPerHour(60000)
                .build())
            .build();
        
        apiKeyRepository.save(apiKey);
        
        // 실제 키 값은 한 번만 반환
        apiKey.setPlainTextKey(keyValue);
        return apiKey;
    }
    
    public boolean validateApiKey(String keyValue, String requiredScope) {
        String hashedKey = hashApiKey(keyValue);
        
        Optional<ApiKey> apiKeyOpt = apiKeyRepository.findByHashedKey(hashedKey);
        
        if (apiKeyOpt.isEmpty()) {
            return false;
        }
        
        ApiKey apiKey = apiKeyOpt.get();
        
        // 상태 체크
        if (apiKey.getStatus() != ApiKeyStatus.ACTIVE) {
            return false;
        }
        
        // 만료 체크
        if (apiKey.getExpiresAt().isBefore(Instant.now())) {
            return false;
        }
        
        // 권한 체크
        if (requiredScope != null && !apiKey.getScopes().contains(requiredScope)) {
            return false;
        }
        
        // 사용 이력 기록
        recordApiKeyUsage(apiKey);
        
        return true;
    }
    
    private String generateSecureKey() {
        SecureRandom random = new SecureRandom();
        byte[] keyBytes = new byte[32];
        random.nextBytes(keyBytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(keyBytes);
    }
    
    private String hashApiKey(String keyValue) {
        return DigestUtils.sha256Hex("api_key_salt" + keyValue);
    }
}
```

## 데이터 암호화

### 전송 중 암호화 (TLS)
```yaml
# nginx.conf
server {
    listen 443 ssl http2;
    server_name api.ecommerce.com;
    
    # TLS 1.3 강제 사용
    ssl_protocols TLSv1.3;
    ssl_certificate /etc/ssl/certs/ecommerce.crt;
    ssl_certificate_key /etc/ssl/private/ecommerce.key;
    
    # 보안 헤더
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 저장 중 암호화
```java
@Entity
public class User {
    
    @Id
    private String userId;
    
    @Column
    @Convert(converter = EncryptedStringConverter.class)
    private String email;
    
    @Column
    @Convert(converter = EncryptedStringConverter.class)
    private String phoneNumber;
    
    @Column
    @Convert(converter = EncryptedStringConverter.class)
    private String socialSecurityNumber;
    
    // 해시된 패스워드 (암호화 불필요)
    @Column
    private String passwordHash;
}

@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    
    private final EncryptionService encryptionService;
    
    public EncryptedStringConverter(EncryptionService encryptionService) {
        this.encryptionService = encryptionService;
    }
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        if (attribute == null) {
            return null;
        }
        return encryptionService.encrypt(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        return encryptionService.decrypt(dbData);
    }
}
```

### 암호화 서비스 구현
```java
@Service
public class EncryptionService {
    
    private final AESUtil aesUtil;
    private final KeyManagementService keyService;
    
    public String encrypt(String plainText) {
        if (plainText == null || plainText.isEmpty()) {
            return plainText;
        }
        
        try {
            // 데이터 암호화 키 조회
            SecretKey dataKey = keyService.getDataEncryptionKey();
            
            // AES-256-GCM으로 암호화
            byte[] encrypted = aesUtil.encrypt(plainText.getBytes(), dataKey);
            
            // Base64 인코딩
            return Base64.getEncoder().encodeToString(encrypted);
            
        } catch (Exception e) {
            throw new EncryptionException("Failed to encrypt data", e);
        }
    }
    
    public String decrypt(String encryptedText) {
        if (encryptedText == null || encryptedText.isEmpty()) {
            return encryptedText;
        }
        
        try {
            // Base64 디코딩
            byte[] encryptedBytes = Base64.getDecoder().decode(encryptedText);
            
            // 데이터 암호화 키 조회
            SecretKey dataKey = keyService.getDataEncryptionKey();
            
            // 복호화
            byte[] decrypted = aesUtil.decrypt(encryptedBytes, dataKey);
            
            return new String(decrypted);
            
        } catch (Exception e) {
            throw new DecryptionException("Failed to decrypt data", e);
        }
    }
}

@Component
public class AESUtil {
    
    private static final String ALGORITHM = "AES";
    private static final String TRANSFORMATION = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 16;
    
    public byte[] encrypt(byte[] plainText, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        
        // IV 생성
        byte[] iv = new byte[GCM_IV_LENGTH];
        SecureRandom.getInstanceStrong().nextBytes(iv);
        
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
        cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);
        
        byte[] encryptedText = cipher.doFinal(plainText);
        
        // IV + 암호화된 데이터 결합
        ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + encryptedText.length);
        byteBuffer.put(iv);
        byteBuffer.put(encryptedText);
        
        return byteBuffer.array();
    }
    
    public byte[] decrypt(byte[] encryptedText, SecretKey key) throws Exception {
        ByteBuffer byteBuffer = ByteBuffer.wrap(encryptedText);
        
        // IV 추출
        byte[] iv = new byte[GCM_IV_LENGTH];
        byteBuffer.get(iv);
        
        // 암호화된 데이터 추출
        byte[] cipherText = new byte[byteBuffer.remaining()];
        byteBuffer.get(cipherText);
        
        Cipher cipher = Cipher.getInstance(TRANSFORMATION);
        GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
        cipher.init(Cipher.DECRYPT_MODE, key, parameterSpec);
        
        return cipher.doFinal(cipherText);
    }
}
```

## 키 관리 시스템

### AWS KMS 통합
```java
@Service
public class KeyManagementService {
    
    private final KmsClient kmsClient;
    private final RedisTemplate<String, byte[]> redisTemplate;
    
    @Value("${aws.kms.master-key-id}")
    private String masterKeyId;
    
    public SecretKey getDataEncryptionKey() {
        String cacheKey = "dek:current";
        
        // 캐시에서 DEK 조회
        byte[] cachedDek = redisTemplate.opsForValue().get(cacheKey);
        if (cachedDek != null) {
            return new SecretKeySpec(cachedDek, "AES");
        }
        
        // KMS에서 새로운 DEK 생성
        GenerateDataKeyRequest request = GenerateDataKeyRequest.builder()
            .keyId(masterKeyId)
            .keySpec(DataKeySpec.AES_256)
            .build();
        
        GenerateDataKeyResponse response = kmsClient.generateDataKey(request);
        
        // 평문 DEK는 메모리에서만 사용
        byte[] plaintextKey = response.plaintext().asByteArray();
        
        // 암호화된 DEK는 안전하게 저장
        byte[] encryptedKey = response.ciphertextBlob().asByteArray();
        storeEncryptedDek(encryptedKey);
        
        // 평문 DEK를 캐시에 저장 (1시간)
        redisTemplate.opsForValue().set(cacheKey, plaintextKey, Duration.ofHours(1));
        
        return new SecretKeySpec(plaintextKey, "AES");
    }
    
    public SecretKey decryptDataEncryptionKey(byte[] encryptedKey) {
        DecryptRequest request = DecryptRequest.builder()
            .ciphertextBlob(SdkBytes.fromByteArray(encryptedKey))
            .build();
        
        DecryptResponse response = kmsClient.decrypt(request);
        byte[] plaintextKey = response.plaintext().asByteArray();
        
        return new SecretKeySpec(plaintextKey, "AES");
    }
    
    @Scheduled(cron = "0 0 */6 * * *") // 6시간마다 키 로테이션
    public void rotateDataEncryptionKey() {
        log.info("Starting DEK rotation");
        
        // 새로운 DEK 생성
        SecretKey newDek = generateNewDataEncryptionKey();
        
        // 캐시 업데이트
        String cacheKey = "dek:current";
        redisTemplate.opsForValue().set(cacheKey, newDek.getEncoded(), Duration.ofHours(1));
        
        log.info("DEK rotation completed");
    }
}
```

## 보안 감사 로깅

### 감사 이벤트 캡처
```java
@Component
@Slf4j
public class SecurityAuditLogger {
    
    private final ObjectMapper objectMapper;
    private final AsyncTaskExecutor taskExecutor;
    
    @EventListener
    @Async
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        AuditLogEntry entry = AuditLogEntry.builder()
            .eventType("AUTHENTICATION_SUCCESS")
            .userId(event.getAuthentication().getName())
            .clientIp(getClientIp(event))
            .userAgent(getUserAgent(event))
            .timestamp(Instant.now())
            .details(Map.of(
                "authenticationMethod", event.getAuthentication().getClass().getSimpleName(),
                "authorities", event.getAuthentication().getAuthorities().toString()
            ))
            .build();
        
        logAuditEvent(entry);
    }
    
    @EventListener
    @Async
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        AuditLogEntry entry = AuditLogEntry.builder()
            .eventType("AUTHENTICATION_FAILURE")
            .userId(event.getAuthentication().getName())
            .clientIp(getClientIp(event))
            .timestamp(Instant.now())
            .details(Map.of(
                "failureReason", event.getException().getMessage(),
                "authenticationMethod", event.getAuthentication().getClass().getSimpleName()
            ))
            .build();
        
        logAuditEvent(entry);
    }
    
    @EventListener
    @Async
    public void handleAccessDenied(AccessDeniedEvent event) {
        AuditLogEntry entry = AuditLogEntry.builder()
            .eventType("ACCESS_DENIED")
            .userId(event.getAuthentication().getName())
            .resource(event.getSource().toString())
            .timestamp(Instant.now())
            .details(Map.of(
                "deniedReason", event.getAccessDeniedException().getMessage(),
                "requiredAuthorities", extractRequiredAuthorities(event)
            ))
            .build();
        
        logAuditEvent(entry);
    }
    
    @EventListener
    @Async
    public void handleSensitiveDataAccess(SensitiveDataAccessEvent event) {
        AuditLogEntry entry = AuditLogEntry.builder()
            .eventType("SENSITIVE_DATA_ACCESS")
            .userId(event.getUserId())
            .resource(event.getResourceType())
            .resourceId(event.getResourceId())
            .timestamp(Instant.now())
            .details(Map.of(
                "operation", event.getOperation(),
                "dataCategory", event.getDataCategory()
            ))
            .build();
        
        logAuditEvent(entry);
    }
    
    private void logAuditEvent(AuditLogEntry entry) {
        try {
            String jsonLog = objectMapper.writeValueAsString(entry);
            
            // 구조화된 로그로 출력 (ELK Stack으로 수집됨)
            log.info("AUDIT: {}", jsonLog);
            
            // 별도 감사 로그 시스템으로 전송 (옵션)
            sendToAuditSystem(entry);
            
        } catch (Exception e) {
            log.error("Failed to log audit event", e);
        }
    }
}
```

## 컴플라이언스 구현

### GDPR 개인정보 처리
```java
@Service
public class GDPRComplianceService {
    
    private final UserRepository userRepository;
    private final DataProcessingLogRepository logRepository;
    
    public void processDataSubjectRequest(DataSubjectRequest request) {
        switch (request.getRequestType()) {
            case ACCESS:
                handleDataAccessRequest(request);
                break;
            case RECTIFICATION:
                handleDataRectificationRequest(request);
                break;
            case ERASURE:
                handleDataErasureRequest(request);
                break;
            case PORTABILITY:
                handleDataPortabilityRequest(request);
                break;
            case RESTRICTION:
                handleProcessingRestrictionRequest(request);
                break;
        }
    }
    
    private void handleDataAccessRequest(DataSubjectRequest request) {
        String userId = request.getUserId();
        
        // 사용자 데이터 수집
        PersonalDataExport export = PersonalDataExport.builder()
            .userId(userId)
            .userProfile(userRepository.findById(userId))
            .orderHistory(orderService.getUserOrders(userId))
            .preferences(preferencesService.getUserPreferences(userId))
            .consentHistory(consentService.getConsentHistory(userId))
            .dataProcessingLog(logRepository.findByUserId(userId))
            .exportedAt(Instant.now())
            .build();
        
        // 30일 내 제공 (GDPR 요구사항)
        scheduleDataDelivery(request, export);
        
        // 요청 처리 로그
        logDataSubjectRequest(request, "ACCESS_REQUEST_PROCESSED");
    }
    
    private void handleDataErasureRequest(DataSubjectRequest request) {
        String userId = request.getUserId();
        
        // 삭제 제외 데이터 확인 (법적 보관 의무)
        List<String> retentionReasons = checkDataRetentionRequirements(userId);
        
        if (retentionReasons.isEmpty()) {
            // 완전 삭제
            performCompleteDataErasure(userId);
        } else {
            // 부분 삭제 및 익명화
            performPartialDataErasure(userId, retentionReasons);
        }
        
        logDataSubjectRequest(request, "ERASURE_REQUEST_PROCESSED");
    }
    
    private void performCompleteDataErasure(String userId) {
        // 개인 정보 완전 삭제
        userRepository.deleteById(userId);
        orderService.anonymizeUserOrders(userId);
        preferencesService.deleteUserPreferences(userId);
        
        // 삭제 증명서 생성
        DeletionCertificate certificate = DeletionCertificate.builder()
            .userId(userId)
            .deletedAt(Instant.now())
            .deletionMethod("COMPLETE_ERASURE")
            .build();
        
        deletionCertificateService.store(certificate);
    }
    
    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    public void performAutomaticDataRetentionCleanup() {
        // 보관 기간 만료 데이터 자동 삭제
        List<String> expiredUsers = findUsersWithExpiredData();
        
        for (String userId : expiredUsers) {
            if (hasValidLegalBasisForProcessing(userId)) {
                continue; // 적법한 처리 근거가 있으면 보관 계속
            }
            
            performCompleteDataErasure(userId);
            log.info("Automatic data retention cleanup completed for user: {}", userId);
        }
    }
}
```

### PCI DSS 결제 보안
```java
@Service
public class PCIComplianceService {
    
    private final PaymentTokenService tokenService;
    private final EncryptionService encryptionService;
    
    public PaymentToken tokenizeCardData(CreditCardData cardData) {
        // PAN (Primary Account Number) 토큰화
        String token = tokenService.generateToken();
        
        // 원본 카드 데이터는 안전한 볼트에 암호화 저장
        String encryptedPan = encryptionService.encrypt(cardData.getCardNumber());
        
        CardDataVault vaultEntry = CardDataVault.builder()
            .token(token)
            .encryptedPan(encryptedPan)
            .cardholderName(encryptionService.encrypt(cardData.getCardholderName()))
            .expiryMonth(cardData.getExpiryMonth())
            .expiryYear(cardData.getExpiryYear())
            .createdAt(Instant.now())
            .build();
        
        vaultRepository.save(vaultEntry);
        
        // 토큰만 반환 (원본 카드 데이터 노출 방지)
        return PaymentToken.builder()
            .token(token)
            .maskedPan(maskCardNumber(cardData.getCardNumber()))
            .cardType(detectCardType(cardData.getCardNumber()))
            .expiryMonth(cardData.getExpiryMonth())
            .expiryYear(cardData.getExpiryYear())
            .build();
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        // 토큰으로 실제 카드 데이터 조회
        CardDataVault vaultEntry = vaultRepository.findByToken(request.getToken())
            .orElseThrow(() -> new InvalidTokenException("Invalid payment token"));
        
        // 카드 데이터 복호화 (메모리에서만 사용)
        String pan = encryptionService.decrypt(vaultEntry.getEncryptedPan());
        String cardholderName = encryptionService.decrypt(vaultEntry.getCardholderName());
        
        try {
            // 결제 처리 (외부 결제 게이트웨이)
            PaymentGatewayRequest gatewayRequest = PaymentGatewayRequest.builder()
                .cardNumber(pan)
                .cardholderName(cardholderName)
                .expiryMonth(vaultEntry.getExpiryMonth())
                .expiryYear(vaultEntry.getExpiryYear())
                .amount(request.getAmount())
                .currency(request.getCurrency())
                .build();
            
            PaymentGatewayResponse response = paymentGateway.processPayment(gatewayRequest);
            
            // 결제 결과 저장 (카드 데이터 제외)
            PaymentTransaction transaction = PaymentTransaction.builder()
                .transactionId(response.getTransactionId())
                .token(request.getToken())
                .maskedPan(vaultEntry.getMaskedPan())
                .amount(request.getAmount())
                .status(response.getStatus())
                .processedAt(Instant.now())
                .build();
            
            transactionRepository.save(transaction);
            
            return PaymentResult.from(response, transaction);
            
        } finally {
            // 메모리에서 카드 데이터 즉시 제거
            pan = null;
            cardholderName = null;
            System.gc();
        }
    }
    
    private String maskCardNumber(String cardNumber) {
        if (cardNumber.length() < 10) {
            return "*".repeat(cardNumber.length());
        }
        
        String firstSix = cardNumber.substring(0, 6);
        String lastFour = cardNumber.substring(cardNumber.length() - 4);
        String middle = "*".repeat(cardNumber.length() - 10);
        
        return firstSix + middle + lastFour;
    }
}
```

## 침입 탐지 시스템

### 이상 행위 탐지
```java
@Service
public class AnomalyDetectionService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final AlertService alertService;
    
    @EventListener
    public void analyzeUserBehavior(UserActivityEvent event) {
        String userId = event.getUserId();
        
        // 사용자 행위 패턴 분석
        UserBehaviorProfile profile = getUserBehaviorProfile(userId);
        
        // 이상 행위 감지
        List<AnomalyIndicator> anomalies = detectAnomalies(event, profile);
        
        if (!anomalies.isEmpty()) {
            SecurityIncident incident = SecurityIncident.builder()
                .incidentId(UUID.randomUUID().toString())
                .userId(userId)
                .incidentType("ANOMALOUS_BEHAVIOR")
                .severity(calculateSeverity(anomalies))
                .anomalies(anomalies)
                .detectedAt(Instant.now())
                .build();
            
            handleSecurityIncident(incident);
        }
        
        // 프로필 업데이트
        updateUserBehaviorProfile(userId, event);
    }
    
    private List<AnomalyIndicator> detectAnomalies(UserActivityEvent event, 
                                                  UserBehaviorProfile profile) {
        List<AnomalyIndicator> anomalies = new ArrayList<>();
        
        // 1. 비정상적인 로그인 시간
        if (isUnusualLoginTime(event.getTimestamp(), profile.getTypicalLoginHours())) {
            anomalies.add(AnomalyIndicator.builder()
                .type("UNUSUAL_LOGIN_TIME")
                .severity(AnomalySeverity.MEDIUM)
                .description("Login outside typical hours")
                .build());
        }
        
        // 2. 비정상적인 IP 주소
        if (!profile.getKnownIpAddresses().contains(event.getClientIp())) {
            GeoLocation location = geoLocationService.getLocation(event.getClientIp());
            if (!profile.getKnownCountries().contains(location.getCountry())) {
                anomalies.add(AnomalyIndicator.builder()
                    .type("UNUSUAL_LOCATION")
                    .severity(AnomalySeverity.HIGH)
                    .description("Login from new country: " + location.getCountry())
                    .build());
            }
        }
        
        // 3. 비정상적인 요청 패턴
        int recentRequestCount = getRecentRequestCount(event.getUserId(), Duration.ofMinutes(5));
        if (recentRequestCount > profile.getMaxRequestsPerMinute() * 3) {
            anomalies.add(AnomalyIndicator.builder()
                .type("UNUSUAL_REQUEST_PATTERN")
                .severity(AnomalySeverity.HIGH)
                .description("Unusually high request rate: " + recentRequestCount)
                .build());
        }
        
        return anomalies;
    }
    
    private void handleSecurityIncident(SecurityIncident incident) {
        // 인시던트 저장
        securityIncidentRepository.save(incident);
        
        // 심각도에 따른 대응
        switch (incident.getSeverity()) {
            case HIGH:
                // 계정 임시 잠금
                userService.lockAccount(incident.getUserId(), Duration.ofHours(1));
                // 즉시 알림
                alertService.sendSecurityAlert(incident);
                break;
                
            case MEDIUM:
                // 추가 인증 요구
                authService.requireAdditionalAuthentication(incident.getUserId());
                // 지연된 알림
                alertService.scheduleSecurityAlert(incident, Duration.ofMinutes(10));
                break;
                
            case LOW:
                // 로그만 기록
                log.info("Low severity security incident: {}", incident);
                break;
        }
    }
}
```

## Best Practices

### 보안 원칙
- **Defense in Depth**: 다층 보안 방어 체계 구축
- **Principle of Least Privilege**: 최소 권한 원칙 적용
- **Fail Secure**: 장애 시 안전한 상태로 전환

### 암호화 관리
- **강력한 알고리즘**: AES-256, RSA-2048 이상 사용
- **키 로테이션**: 정기적인 암호화 키 교체
- **키 분리**: 암호화 키와 데이터 분리 저장

### 컴플라이언스
- **데이터 최소화**: 필요한 최소한의 개인정보만 수집
- **투명성**: 개인정보 처리 목적과 방법 명시
- **사용자 권리**: 개인정보 주체 권리 보장

## Benefits and Challenges

### Benefits
- **데이터 보호**: 개인정보와 민감 데이터의 안전한 보호
- **규정 준수**: 법적 요구사항 충족으로 법적 리스크 최소화
- **신뢰성**: 고객과 파트너의 신뢰 확보
- **비즈니스 연속성**: 보안 사고로 인한 업무 중단 방지

### Challenges
- **복잡성**: 다양한 보안 요구사항의 복잡한 구현
- **성능 영향**: 암호화와 보안 검사로 인한 성능 오버헤드
- **비용**: 보안 시스템 구축과 유지보수 비용
- **사용성**: 보안 강화로 인한 사용자 경험 저하 가능성