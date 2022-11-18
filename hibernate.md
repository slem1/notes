Auditing fields

Required: @EntityListeners(AuditingEntityListener.class) on entity class or super class (@MappedSuperclass)

Spring : AnnotationAuditingMetadata --> Detect and retrieve @CreatedDate, @CreatedBy, @LastModifiedBy, @LastModifiedDatefield on classes

ReflectionAuditingBeanWrapper use --> AnnotationAuditingMetadata to fill in the fields on target object

AuditingEntityListener --> bring the @PrePersist and @PreUpdate ORM listener

so the code path is 

AuditingEntity --> ReflectionAuditingBeanWrapper --> AnnotationAuditingMetadata
