# 데이터 거버넌스 가이드

## 1. GDPR 준수를 위한 개인정보 처리

### 개인정보 식별 및 분류
```java
@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface PersonalData {
    
    DataCategory category() default DataCategory.GENERAL;
    String purpose() default "";
    boolean encrypted() default false;
    long retentionDays() default 0;
    String lawfulBasis() default "";
    
    enum DataCategory {
        GENERAL,           // 일반 개인정보
        SENSITIVE,         // 민감정보
        SPECIAL_CATEGORY,  // 특별 범주 개인정보
        BIOMETRIC,         // 생체정보
        FINANCIAL          // 금융정보
    }
}

@Entity
@Table(name = "customers")
@DataGovernance(
    dataClassification = "PERSONAL",
    retentionPolicy = "CUSTOMER_RETENTION_POLICY",
    accessControl = "CUSTOMER_ACCESS_CONTROL"
)
public class Customer {
    
    @Id
    private String customerId;
    
    @PersonalData(
        category = PersonalData.DataCategory.GENERAL,
        purpose = "User identification and communication",
        lawfulBasis = "Contract performance"
    )
    @Column(name = "full_name")
    private String fullName;
    
    @PersonalData(
        category = PersonalData.DataCategory.GENERAL,
        purpose = "Communication and service delivery",
        encrypted = true,
        lawfulBasis = "Contract performance"
    )
    @Column(name = "email")
    private String email;
    
    @PersonalData(
        category = PersonalData.DataCategory.GENERAL,
        purpose = "Order delivery and customer service",
        encrypted = true,
        lawfulBasis = "Contract performance"
    )
    @Column(name = "phone_number")
    private String phoneNumber;
    
    @PersonalData(
        category = PersonalData.DataCategory.GENERAL,
        purpose = "Order delivery",
        retentionDays = 2555, // 7년
        lawfulBasis = "Contract performance"
    )
    @Embedded
    private Address address;
    
    @PersonalData(
        category = PersonalData.DataCategory.SENSITIVE,
        purpose = "Age verification",
        encrypted = true,
        lawfulBasis = "Legal obligation"
    )
    @Column(name = "date_of_birth")
    private LocalDate dateOfBirth;
    
    @Column(name = "created_at")
    @CreationTimestamp
    private Instant createdAt;
    
    @Column(name = "consent_given_at")
    private Instant consentGivenAt;
    
    @ElementCollection
    @CollectionTable(name = "customer_consents")
    private Set<ConsentRecord> consents;
    
    // getters, setters
}
```

### 동의 관리 시스템
```java
@Entity
@Table(name = "consent_records")
public class ConsentRecord {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "customer_id")
    private String customerId;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "consent_type")
    private ConsentType consentType;
    
    @Column(name = "given")
    private boolean given;
    
    @Column(name = "given_at")
    private Instant givenAt;
    
    @Column(name = "withdrawn_at")
    private Instant withdrawnAt;
    
    @Column(name = "purpose")
    private String purpose;
    
    @Column(name = "legal_basis")
    private String legalBasis;
    
    @Column(name = "data_categories")
    private String dataCategories;
    
    public enum ConsentType {
        MARKETING_EMAIL,
        MARKETING_SMS,
        ANALYTICS_TRACKING,
        PERSONALIZATION,
        THIRD_PARTY_SHARING,
        PROFILING
    }
}

@Service
@Transactional
public class ConsentManagementService {
    
    private final ConsentRepository consentRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    public void giveConsent(String customerId, ConsentType consentType, 
                           String purpose, String legalBasis) {
        
        // 기존 동의 철회
        withdrawConsent(customerId, consentType);
        
        // 새로운 동의 기록
        ConsentRecord consent = new ConsentRecord();
        consent.setCustomerId(customerId);
        consent.setConsentType(consentType);
        consent.setGiven(true);
        consent.setGivenAt(Instant.now());
        consent.setPurpose(purpose);
        consent.setLegalBasis(legalBasis);
        
        consentRepository.save(consent);
        
        // 동의 이벤트 발행
        eventPublisher.publishEvent(new ConsentGivenEvent(
            customerId, consentType, purpose, Instant.now()
        ));
    }
    
    public void withdrawConsent(String customerId, ConsentType consentType) {
        List<ConsentRecord> activeConsents = consentRepository
            .findByCustomerIdAndConsentTypeAndGivenTrueAndWithdrawnAtIsNull(
                customerId, consentType);
        
        for (ConsentRecord consent : activeConsents) {
            consent.setGiven(false);
            consent.setWithdrawnAt(Instant.now());
            consentRepository.save(consent);
        }
        
        // 동의 철회 이벤트 발행
        eventPublisher.publishEvent(new ConsentWithdrawnEvent(
            customerId, consentType, Instant.now()
        ));
    }
    
    public boolean hasValidConsent(String customerId, ConsentType consentType) {
        return consentRepository.existsByCustomerIdAndConsentTypeAndGivenTrueAndWithdrawnAtIsNull(
            customerId, consentType);
    }
    
    public ConsentStatus getConsentStatus(String customerId) {
        List<ConsentRecord> consents = consentRepository
            .findCurrentConsentsByCustomerId(customerId);
        
        Map<ConsentType, Boolean> consentMap = consents.stream()
            .collect(Collectors.toMap(
                ConsentRecord::getConsentType,
                ConsentRecord::isGiven,
                (existing, replacement) -> replacement
            ));
        
        return new ConsentStatus(customerId, consentMap, Instant.now());
    }
}
```

### 개인정보 처리 이벤트
```java
public class PersonalDataProcessedEvent extends DomainEvent {
    
    private String customerId;
    private String dataSubject;
    private PersonalData.DataCategory dataCategory;
    private String processingPurpose;
    private String legalBasis;
    private String processingActivity;
    private Instant processedAt;
    private String processedBy; // 시스템 또는 사용자
    
    // constructors, getters
}

public class PersonalDataAccessedEvent extends DomainEvent {
    
    private String customerId;
    private String accessedBy;
    private String accessPurpose;
    private List<String> accessedFields;
    private String sourceSystem;
    private Instant accessedAt;
    
    // constructors, getters
}

@EventListener
@Component
public class PersonalDataAuditLogger {
    
    private final DataProcessingLogRepository logRepository;
    
    @EventListener
    public void handlePersonalDataProcessed(PersonalDataProcessedEvent event) {
        DataProcessingLog log = DataProcessingLog.builder()
            .customerId(event.getCustomerId())
            .dataCategory(event.getDataCategory().name())
            .processingPurpose(event.getProcessingPurpose())
            .legalBasis(event.getLegalBasis())
            .processingActivity(event.getProcessingActivity())
            .processedAt(event.getProcessedAt())
            .processedBy(event.getProcessedBy())
            .build();
        
        logRepository.save(log);
    }
    
    @EventListener
    public void handlePersonalDataAccessed(PersonalDataAccessedEvent event) {
        DataAccessLog log = DataAccessLog.builder()
            .customerId(event.getCustomerId())
            .accessedBy(event.getAccessedBy())
            .accessPurpose(event.getAccessPurpose())
            .accessedFields(String.join(",", event.getAccessedFields()))
            .sourceSystem(event.getSourceSystem())
            .accessedAt(event.getAccessedAt())
            .build();
        
        dataAccessLogRepository.save(log);
    }
}
```

## 2. 이벤트 데이터 보관 정책

### 데이터 보관 정책 설정
```java
@Configuration
public class DataRetentionConfig {
    
    @Bean
    public DataRetentionPolicyRegistry retentionPolicyRegistry() {
        DataRetentionPolicyRegistry registry = new DataRetentionPolicyRegistry();
        
        // 고객 데이터 보관 정책
        registry.addPolicy("customer-data", DataRetentionPolicy.builder()
            .retentionPeriod(Duration.ofDays(2555)) // 7년
            .archivePeriod(Duration.ofDays(1825))   // 5년 후 아카이브
            .deletionMethod(DeletionMethod.SECURE_DELETE)
            .legalBasis("Contract performance, Legal obligation")
            .dataCategories(Set.of("personal_info", "contact_info"))
            .build());
        
        // 주문 데이터 보관 정책
        registry.addPolicy("order-data", DataRetentionPolicy.builder()
            .retentionPeriod(Duration.ofDays(2555)) // 7년 (회계법)
            .archivePeriod(Duration.ofDays(1095))   // 3년 후 아카이브
            .deletionMethod(DeletionMethod.ANONYMIZATION)
            .legalBasis("Legal obligation")
            .dataCategories(Set.of("transaction_data", "financial_data"))
            .build());
        
        // 마케팅 데이터 보관 정책
        registry.addPolicy("marketing-data", DataRetentionPolicy.builder()
            .retentionPeriod(Duration.ofDays(1095)) // 3년
            .archivePeriod(Duration.ofDays(730))    // 2년 후 아카이브
            .deletionMethod(DeletionMethod.PERMANENT_DELETE)
            .legalBasis("Consent")
            .consentRequired(true)
            .dataCategories(Set.of("behavioral_data", "preference_data"))
            .build());
        
        // 로그 데이터 보관 정책
        registry.addPolicy("audit-logs", DataRetentionPolicy.builder()
            .retentionPeriod(Duration.ofDays(2555)) // 7년
            .archivePeriod(Duration.ofDays(365))    // 1년 후 아카이브
            .deletionMethod(DeletionMethod.SECURE_DELETE)
            .legalBasis("Legal obligation")
            .dataCategories(Set.of("audit_trail", "security_logs"))
            .build());
        
        return registry;
    }
}

@Data
@Builder
public class DataRetentionPolicy {
    
    private Duration retentionPeriod;
    private Duration archivePeriod;
    private DeletionMethod deletionMethod;
    private String legalBasis;
    private Set<String> dataCategories;
    private boolean consentRequired;
    private String description;
    
    public enum DeletionMethod {
        PERMANENT_DELETE,   // 영구 삭제
        SECURE_DELETE,      // 안전한 삭제 (복구 불가능)
        ANONYMIZATION,      // 익명화
        PSEUDONYMIZATION    // 가명화
    }
}
```

### 자동 데이터 보관 및 삭제
```java
@Service
@Slf4j
public class DataRetentionService {
    
    private final DataRetentionPolicyRegistry policyRegistry;
    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final EventStoreRepository eventStoreRepository;
    private final DataArchiveService archiveService;
    
    @Scheduled(cron = "0 2 * * * ?") // 매일 새벽 2시 실행
    public void performDataRetentionTasks() {
        log.info("Starting daily data retention tasks");
        
        try {
            // 고객 데이터 보관 처리
            processCustomerDataRetention();
            
            // 주문 데이터 보관 처리
            processOrderDataRetention();
            
            // 이벤트 데이터 보관 처리
            processEventDataRetention();
            
            // 로그 데이터 보관 처리
            processLogDataRetention();
            
        } catch (Exception e) {
            log.error("Data retention task failed", e);
            // 알림 발송
            sendRetentionTaskFailureAlert(e);
        }
    }
    
    private void processCustomerDataRetention() {
        DataRetentionPolicy policy = policyRegistry.getPolicy("customer-data");
        Instant archiveThreshold = Instant.now().minus(policy.getArchivePeriod());
        Instant deletionThreshold = Instant.now().minus(policy.getRetentionPeriod());
        
        // 아카이브 대상 고객 조회
        List<Customer> customersToArchive = customerRepository
            .findCustomersForArchive(archiveThreshold);
        
        for (Customer customer : customersToArchive) {
            try {
                archiveService.archiveCustomerData(customer);
                customer.setArchived(true);
                customer.setArchivedAt(Instant.now());
                customerRepository.save(customer);
                
                log.info("Archived customer data for customer: {}", customer.getCustomerId());
                
            } catch (Exception e) {
                log.error("Failed to archive customer: {}", customer.getCustomerId(), e);
            }
        }
        
        // 삭제 대상 고객 조회
        List<Customer> customersToDelete = customerRepository
            .findCustomersForDeletion(deletionThreshold);
        
        for (Customer customer : customersToDelete) {
            try {
                // 동의 철회 여부 확인
                if (shouldDeleteCustomerData(customer)) {
                    deleteCustomerData(customer, policy.getDeletionMethod());
                    log.info("Deleted customer data for customer: {}", customer.getCustomerId());
                }
                
            } catch (Exception e) {
                log.error("Failed to delete customer: {}", customer.getCustomerId(), e);
            }
        }
    }
    
    private boolean shouldDeleteCustomerData(Customer customer) {
        // 법적 보관 의무 확인
        if (hasLegalHoldRequirement(customer)) {
            return false;
        }
        
        // 동의 기반 데이터인 경우 동의 철회 확인
        if (isConsentBasedData(customer)) {
            return !hasValidConsent(customer);
        }
        
        return true;
    }
    
    private void deleteCustomerData(Customer customer, 
                                  DataRetentionPolicy.DeletionMethod method) {
        switch (method) {
            case PERMANENT_DELETE -> {
                customerRepository.delete(customer);
                eventPublisher.publishEvent(new PersonalDataDeletedEvent(
                    customer.getCustomerId(), 
                    "Retention policy",
                    Instant.now()
                ));
            }
            case ANONYMIZATION -> {
                anonymizeCustomerData(customer);
            }
            case PSEUDONYMIZATION -> {
                pseudonymizeCustomerData(customer);
            }
            case SECURE_DELETE -> {
                secureDeleteCustomerData(customer);
            }
        }
    }
    
    private void anonymizeCustomerData(Customer customer) {
        // 개인 식별 정보 제거
        customer.setFullName("ANONYMIZED");
        customer.setEmail("anonymized@example.com");
        customer.setPhoneNumber("000-0000-0000");
        customer.setDateOfBirth(null);
        customer.getAddress().anonymize();
        customer.setAnonymized(true);
        customer.setAnonymizedAt(Instant.now());
        
        customerRepository.save(customer);
        
        eventPublisher.publishEvent(new PersonalDataAnonymizedEvent(
            customer.getCustomerId(),
            "Retention policy",
            Instant.now()
        ));
    }
    
    private void processEventDataRetention() {
        DataRetentionPolicy policy = policyRegistry.getPolicy("audit-logs");
        Instant archiveThreshold = Instant.now().minus(policy.getArchivePeriod());
        Instant deletionThreshold = Instant.now().minus(policy.getRetentionPeriod());
        
        // 이벤트 스토어 아카이브
        List<EventEntity> eventsToArchive = eventStoreRepository
            .findEventsForArchive(archiveThreshold);
        
        if (!eventsToArchive.isEmpty()) {
            archiveService.archiveEvents(eventsToArchive);
            eventStoreRepository.markAsArchived(eventsToArchive);
            
            log.info("Archived {} events", eventsToArchive.size());
        }
        
        // 오래된 이벤트 삭제
        int deletedEvents = eventStoreRepository.deleteOldEvents(deletionThreshold);
        if (deletedEvents > 0) {
            log.info("Deleted {} old events", deletedEvents);
        }
    }
}
```

## 3. 감사 로그 관리

### 포괄적인 감사 로그 시스템
```java
@Entity
@Table(name = "audit_logs")
@Index(name = "idx_audit_entity_id", columnList = "entity_id")
@Index(name = "idx_audit_timestamp", columnList = "created_at")
public class AuditLog {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "entity_type")
    private String entityType;
    
    @Column(name = "entity_id")
    private String entityId;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "operation")
    private AuditOperation operation;
    
    @Column(name = "user_id")
    private String userId;
    
    @Column(name = "session_id")
    private String sessionId;
    
    @Column(name = "source_ip")
    private String sourceIp;
    
    @Column(name = "user_agent")
    private String userAgent;
    
    @Type(JsonType.class)
    @Column(name = "old_values", columnDefinition = "jsonb")
    private Map<String, Object> oldValues;
    
    @Type(JsonType.class)
    @Column(name = "new_values", columnDefinition = "jsonb")
    private Map<String, Object> newValues;
    
    @Type(JsonType.class)
    @Column(name = "metadata", columnDefinition = "jsonb")
    private Map<String, Object> metadata;
    
    @Column(name = "created_at")
    @CreationTimestamp
    private Instant createdAt;
    
    public enum AuditOperation {
        CREATE, READ, UPDATE, DELETE, 
        LOGIN, LOGOUT, 
        EXPORT, IMPORT,
        CONSENT_GIVEN, CONSENT_WITHDRAWN,
        DATA_ACCESSED, DATA_PROCESSED
    }
}

@Component
@Slf4j
public class AuditLogService {
    
    private final AuditLogRepository auditRepository;
    private final ObjectMapper objectMapper;
    
    @Async
    public void logDataAccess(String entityType, String entityId, 
                             String userId, String purpose) {
        AuditLog auditLog = AuditLog.builder()
            .entityType(entityType)
            .entityId(entityId)
            .operation(AuditLog.AuditOperation.READ)
            .userId(userId)
            .sessionId(getCurrentSessionId())
            .sourceIp(getCurrentClientIp())
            .userAgent(getCurrentUserAgent())
            .metadata(Map.of(
                "access_purpose", purpose,
                "access_time", Instant.now().toString()
            ))
            .build();
        
        auditRepository.save(auditLog);
    }
    
    @Async
    public void logDataModification(String entityType, String entityId,
                                   AuditLog.AuditOperation operation,
                                   Object oldValue, Object newValue,
                                   String userId) {
        try {
            Map<String, Object> oldValues = oldValue != null ? 
                objectMapper.convertValue(oldValue, Map.class) : null;
            Map<String, Object> newValues = newValue != null ? 
                objectMapper.convertValue(newValue, Map.class) : null;
            
            // 민감한 정보 마스킹
            if (oldValues != null) {
                maskSensitiveData(oldValues);
            }
            if (newValues != null) {
                maskSensitiveData(newValues);
            }
            
            AuditLog auditLog = AuditLog.builder()
                .entityType(entityType)
                .entityId(entityId)
                .operation(operation)
                .userId(userId)
                .sessionId(getCurrentSessionId())
                .sourceIp(getCurrentClientIp())
                .userAgent(getCurrentUserAgent())
                .oldValues(oldValues)
                .newValues(newValues)
                .build();
            
            auditRepository.save(auditLog);
            
        } catch (Exception e) {
            log.error("Failed to log audit event", e);
        }
    }
    
    @Async
    public void logConsentChange(String customerId, ConsentType consentType,
                                boolean given, String userId) {
        AuditLog.AuditOperation operation = given ? 
            AuditLog.AuditOperation.CONSENT_GIVEN : 
            AuditLog.AuditOperation.CONSENT_WITHDRAWN;
        
        AuditLog auditLog = AuditLog.builder()
            .entityType("Customer")
            .entityId(customerId)
            .operation(operation)
            .userId(userId)
            .sessionId(getCurrentSessionId())
            .sourceIp(getCurrentClientIp())
            .metadata(Map.of(
                "consent_type", consentType.name(),
                "consent_given", given,
                "change_time", Instant.now().toString()
            ))
            .build();
        
        auditRepository.save(auditLog);
    }
    
    private void maskSensitiveData(Map<String, Object> data) {
        Set<String> sensitiveFields = Set.of(
            "password", "creditCardNumber", "ssn", "accountNumber",
            "email", "phoneNumber", "address"
        );
        
        for (String field : sensitiveFields) {
            if (data.containsKey(field)) {
                data.put(field, "[MASKED]");
            }
        }
    }
    
    public List<AuditLog> getAuditTrail(String entityType, String entityId) {
        return auditRepository.findByEntityTypeAndEntityIdOrderByCreatedAtDesc(
            entityType, entityId);
    }
    
    public List<AuditLog> getUserActivity(String userId, Instant from, Instant to) {
        return auditRepository.findByUserIdAndCreatedAtBetweenOrderByCreatedAtDesc(
            userId, from, to);
    }
}
```

### JPA 감사 어노테이션 통합
```java
@EntityListeners(AuditingEntityListener.class)
@Entity
public class AuditableEntity {
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;
    
    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;
    
    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;
}

@Component
public class AuditAwareImpl implements AuditorAware<String> {
    
    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        
        if (authentication != null && authentication.isAuthenticated() && 
            !authentication.getPrincipal().equals("anonymousUser")) {
            return Optional.of(authentication.getName());
        }
        
        return Optional.of("system");
    }
}

@EventListener
@Component
public class JpaAuditEventListener {
    
    private final AuditLogService auditLogService;
    
    @EventListener
    public void handleEntityCreated(EntityCreatedEvent event) {
        auditLogService.logDataModification(
            event.getEntityClass().getSimpleName(),
            extractEntityId(event.getEntity()),
            AuditLog.AuditOperation.CREATE,
            null,
            event.getEntity(),
            getCurrentUserId()
        );
    }
    
    @EventListener
    public void handleEntityUpdated(EntityUpdatedEvent event) {
        auditLogService.logDataModification(
            event.getEntityClass().getSimpleName(),
            extractEntityId(event.getEntity()),
            AuditLog.AuditOperation.UPDATE,
            event.getOldValue(),
            event.getNewValue(),
            getCurrentUserId()
        );
    }
    
    @EventListener
    public void handleEntityDeleted(EntityDeletedEvent event) {
        auditLogService.logDataModification(
            event.getEntityClass().getSimpleName(),
            extractEntityId(event.getEntity()),
            AuditLog.AuditOperation.DELETE,
            event.getEntity(),
            null,
            getCurrentUserId()
        );
    }
}
```

## 4. 데이터 품질 관리

### 데이터 품질 규칙 정의
```java
@Component
public class DataQualityRuleEngine {
    
    private final List<DataQualityRule> rules;
    private final DataQualityMetricsService metricsService;
    
    public DataQualityRuleEngine(List<DataQualityRule> rules,
                                DataQualityMetricsService metricsService) {
        this.rules = rules;
        this.metricsService = metricsService;
    }
    
    public DataQualityReport evaluateData(Object data) {
        List<DataQualityIssue> issues = new ArrayList<>();
        
        for (DataQualityRule rule : rules) {
            if (rule.appliesTo(data.getClass())) {
                DataQualityResult result = rule.evaluate(data);
                if (!result.isValid()) {
                    issues.addAll(result.getIssues());
                }
            }
        }
        
        DataQualityReport report = new DataQualityReport(
            data.getClass().getSimpleName(),
            issues,
            calculateQualityScore(issues),
            Instant.now()
        );
        
        // 메트릭 업데이트
        metricsService.recordQualityReport(report);
        
        return report;
    }
    
    private double calculateQualityScore(List<DataQualityIssue> issues) {
        if (issues.isEmpty()) {
            return 100.0;
        }
        
        double totalWeight = issues.stream()
            .mapToDouble(issue -> issue.getSeverity().getWeight())
            .sum();
        
        return Math.max(0, 100.0 - totalWeight);
    }
}

public interface DataQualityRule {
    boolean appliesTo(Class<?> dataType);
    DataQualityResult evaluate(Object data);
    String getRuleName();
    String getDescription();
}

@Component
public class CustomerDataQualityRules implements DataQualityRule {
    
    @Override
    public boolean appliesTo(Class<?> dataType) {
        return Customer.class.isAssignableFrom(dataType);
    }
    
    @Override
    public DataQualityResult evaluate(Object data) {
        Customer customer = (Customer) data;
        List<DataQualityIssue> issues = new ArrayList<>();
        
        // 이메일 형식 검증
        if (!isValidEmail(customer.getEmail())) {
            issues.add(new DataQualityIssue(
                "INVALID_EMAIL_FORMAT",
                "Invalid email format: " + customer.getEmail(),
                DataQualityIssue.Severity.HIGH,
                "email"
            ));
        }
        
        // 전화번호 형식 검증
        if (!isValidPhoneNumber(customer.getPhoneNumber())) {
            issues.add(new DataQualityIssue(
                "INVALID_PHONE_FORMAT",
                "Invalid phone number format: " + customer.getPhoneNumber(),
                DataQualityIssue.Severity.MEDIUM,
                "phoneNumber"
            ));
        }
        
        // 이름 길이 검증
        if (customer.getFullName() == null || customer.getFullName().trim().length() < 2) {
            issues.add(new DataQualityIssue(
                "INVALID_NAME_LENGTH",
                "Name is too short or empty",
                DataQualityIssue.Severity.HIGH,
                "fullName"
            ));
        }
        
        // 주소 완성도 검증
        if (customer.getAddress() != null) {
            if (customer.getAddress().getStreet() == null || 
                customer.getAddress().getCity() == null ||
                customer.getAddress().getZipCode() == null) {
                issues.add(new DataQualityIssue(
                    "INCOMPLETE_ADDRESS",
                    "Address information is incomplete",
                    DataQualityIssue.Severity.MEDIUM,
                    "address"
                ));
            }
        }
        
        // 나이 유효성 검증
        if (customer.getDateOfBirth() != null) {
            LocalDate now = LocalDate.now();
            LocalDate birthDate = customer.getDateOfBirth();
            
            if (birthDate.isAfter(now)) {
                issues.add(new DataQualityIssue(
                    "FUTURE_BIRTH_DATE",
                    "Birth date cannot be in the future",
                    DataQualityIssue.Severity.HIGH,
                    "dateOfBirth"
                ));
            }
            
            long age = ChronoUnit.YEARS.between(birthDate, now);
            if (age > 150) {
                issues.add(new DataQualityIssue(
                    "UNREALISTIC_AGE",
                    "Age appears to be unrealistic: " + age,
                    DataQualityIssue.Severity.MEDIUM,
                    "dateOfBirth"
                ));
            }
        }
        
        return new DataQualityResult(issues.isEmpty(), issues);
    }
    
    @Override
    public String getRuleName() {
        return "CustomerDataQualityRules";
    }
    
    @Override
    public String getDescription() {
        return "Validates customer data for format, completeness, and consistency";
    }
    
    private boolean isValidEmail(String email) {
        if (email == null) return false;
        return email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    }
    
    private boolean isValidPhoneNumber(String phone) {
        if (phone == null) return false;
        return phone.matches("^\\+?[1-9]\\d{1,14}$");
    }
}
```

### 자동 데이터 정제
```java
@Service
public class DataCleansingService {
    
    private final DataQualityRuleEngine ruleEngine;
    private final ApplicationEventPublisher eventPublisher;
    
    @EventListener
    @Async
    public void handleDataQualityIssue(DataQualityIssueDetectedEvent event) {
        try {
            DataQualityIssue issue = event.getIssue();
            Object data = event.getData();
            
            if (isAutoFixableIssue(issue)) {
                Object cleanedData = applyDataCleansing(data, issue);
                
                if (cleanedData != null) {
                    eventPublisher.publishEvent(new DataCleansingAppliedEvent(
                        data,
                        cleanedData,
                        issue,
                        Instant.now()
                    ));
                }
            } else {
                // 수동 처리 필요
                eventPublisher.publishEvent(new ManualDataReviewRequiredEvent(
                    data,
                    issue,
                    Instant.now()
                ));
            }
            
        } catch (Exception e) {
            log.error("Failed to handle data quality issue", e);
        }
    }
    
    private boolean isAutoFixableIssue(DataQualityIssue issue) {
        return switch (issue.getIssueType()) {
            case "EXTRA_WHITESPACE", "INCONSISTENT_CASE", "MISSING_COUNTRY_CODE" -> true;
            default -> false;
        };
    }
    
    private Object applyDataCleansing(Object data, DataQualityIssue issue) {
        if (data instanceof Customer customer) {
            return cleanseCustomerData(customer, issue);
        }
        return null;
    }
    
    private Customer cleanseCustomerData(Customer customer, DataQualityIssue issue) {
        Customer cleaned = customer.copy();
        
        switch (issue.getIssueType()) {
            case "EXTRA_WHITESPACE" -> {
                if ("fullName".equals(issue.getField())) {
                    cleaned.setFullName(customer.getFullName().trim());
                }
                if ("email".equals(issue.getField())) {
                    cleaned.setEmail(customer.getEmail().trim().toLowerCase());
                }
            }
            case "INCONSISTENT_CASE" -> {
                if ("email".equals(issue.getField())) {
                    cleaned.setEmail(customer.getEmail().toLowerCase());
                }
            }
            case "MISSING_COUNTRY_CODE" -> {
                if ("phoneNumber".equals(issue.getField())) {
                    String phone = customer.getPhoneNumber();
                    if (!phone.startsWith("+")) {
                        cleaned.setPhoneNumber("+82" + phone); // 기본 한국 국가코드
                    }
                }
            }
        }
        
        return cleaned;
    }
    
    @Scheduled(cron = "0 3 * * * ?") // 매일 새벽 3시
    public void performBatchDataCleansing() {
        log.info("Starting batch data cleansing");
        
        // 품질 이슈가 있는 데이터 조회
        List<DataQualityReport> issueReports = dataQualityRepository
            .findUnresolvedQualityIssues();
        
        for (DataQualityReport report : issueReports) {
            try {
                processQualityReport(report);
            } catch (Exception e) {
                log.error("Failed to process quality report: {}", report.getId(), e);
            }
        }
        
        log.info("Batch data cleansing completed");
    }
}
```

## 5. 마스터 데이터 관리

### 참조 데이터 관리
```java
@Entity
@Table(name = "reference_data")
public class ReferenceData {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "category")
    private String category;
    
    @Column(name = "code")
    private String code;
    
    @Column(name = "value")
    private String value;
    
    @Column(name = "description")
    private String description;
    
    @Column(name = "active")
    private boolean active;
    
    @Column(name = "sort_order")
    private Integer sortOrder;
    
    @Type(JsonType.class)
    @Column(name = "attributes", columnDefinition = "jsonb")
    private Map<String, Object> attributes;
    
    @Column(name = "effective_from")
    private Instant effectiveFrom;
    
    @Column(name = "effective_to")
    private Instant effectiveTo;
    
    // constructors, getters, setters
}

@Service
public class ReferenceDataService {
    
    private final ReferenceDataRepository repository;
    private final CacheManager cacheManager;
    
    @Cacheable(value = "referenceData", key = "#category")
    public List<ReferenceData> getReferenceData(String category) {
        return repository.findByCategoryAndActiveTrue(category);
    }
    
    @Cacheable(value = "referenceDataByCode", key = "#category + '_' + #code")
    public Optional<ReferenceData> getReferenceDataByCode(String category, String code) {
        return repository.findByCategoryAndCodeAndActiveTrue(category, code);
    }
    
    @CacheEvict(value = {"referenceData", "referenceDataByCode"}, allEntries = true)
    public ReferenceData updateReferenceData(ReferenceData data) {
        ReferenceData saved = repository.save(data);
        
        // 변경 이벤트 발행
        eventPublisher.publishEvent(new ReferenceDataChangedEvent(
            data.getCategory(),
            data.getCode(),
            "UPDATE",
            Instant.now()
        ));
        
        return saved;
    }
    
    public void validateReferenceDataIntegrity() {
        List<String> categories = repository.findDistinctCategories();
        
        for (String category : categories) {
            List<ReferenceData> data = repository.findByCategory(category);
            
            // 중복 코드 검사
            Set<String> codes = new HashSet<>();
            for (ReferenceData item : data) {
                if (!codes.add(item.getCode())) {
                    log.warn("Duplicate reference data code found: {} in category: {}", 
                            item.getCode(), category);
                }
            }
            
            // 유효 기간 검사
            validateEffectivePeriods(data);
        }
    }
    
    private void validateEffectivePeriods(List<ReferenceData> data) {
        for (ReferenceData item : data) {
            if (item.getEffectiveFrom() != null && item.getEffectiveTo() != null) {
                if (item.getEffectiveFrom().isAfter(item.getEffectiveTo())) {
                    log.warn("Invalid effective period for reference data: {}", item.getCode());
                }
            }
        }
    }
}
```

## 실습 과제

1. **GDPR 준수 시스템**: 완전한 개인정보 처리 및 동의 관리 시스템 구현
2. **자동 데이터 보관**: 정책 기반 자동 데이터 아카이브 및 삭제 시스템 구축
3. **포괄적 감사**: 모든 데이터 접근 및 변경을 추적하는 감사 시스템 구현
4. **데이터 품질 자동화**: 실시간 데이터 품질 검사 및 자동 정제 시스템 구현
5. **데이터 거버넌스 대시보드**: 데이터 품질, 보관 상태, 규정 준수를 모니터링하는 대시보드 구축

## 참고 자료

- [GDPR 가이드라인](https://gdpr.eu/)
- [ISO 27001 정보보안관리시스템](https://www.iso.org/isoiec-27001-information-security.html)
- [데이터 거버넌스 모범 사례](https://www.dama.org/)
- [Spring Data JPA Auditing](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#auditing)