# Guía PC2 — Mente Clara Consultores (Sección 7385)
### Curso: Desarrollo de Aplicaciones Open Source (1ASI0729)

> **Tu caso:** Mente Clara Consultores · **Bounded context:** `therapy` · **Aggregate root:** `TherapySession`
> **Endpoint único:** `POST /api/v1/psychologist/{psychologistId}/sessions`
> **Puerto:** `8095` · **DB schema:** `therapy` · **Package raíz:** `pe.menteclara.platform.u<código>`

---

## Índice

1. [Configuración inicial (pgAdmin + Spring Initializr)](#1-configuración-inicial)
2. [Configuración del proyecto](#2-configuración-del-proyecto)
3. [Copiar el shared](#3-copiar-el-shared)
4. [Value Objects del dominio](#4-value-objects)
5. [Aggregate Root: TherapySession](#5-aggregate-root-therapysession)
6. [Domain Repository (interfaz)](#6-domain-repository)
7. [CQRS — Command y Query](#7-cqrs--command-y-query)
8. [Application Services](#8-application-services)
9. [Infrastructure — Persistence](#9-infrastructure--persistence)
10. [Interfaces REST](#10-interfaces-rest)
11. [OpenAPI (documentación Swagger)](#11-openapi-swagger)
12. [README.md](#12-readmemd)
13. [Empaquetado final](#13-empaquetado-final)
14. [Checklist rápido antes de entregar](#14-checklist-final)

---

## 1. Configuración inicial

### Base de datos

Abrir **pgAdmin**, ingresar contraseña `12345678` y crear la base de datos: **`therapy`**

### Crear el proyecto en Spring Initializr

Abrir el navegador y pegar el siguiente enlace *(ajusta `artifactId` y `packageName` con tu código)*:

```
https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.5.0&packaging=jar&configurationFileFormat=properties&jvmVersion=25&groupId=pe.menteclara.platform&artifactId=pc27385u202014215&packageName=pe.menteclara.platform.u202014215&dependencies=data-jpa,validation,web,devtools,postgresql,lombok,springdoc-openapi
```

> Reemplaza `7385` con tu NRC y `202014215` con tu código de estudiante.

Generar el proyecto, guardarlo en `IdeaProjects\1ASI0729\` y abrirlo en **IntelliJ IDEA**.

---

## 2. Configuración del proyecto

### SDK

`File → Project Structure → Project` → seleccionar **Java 25**.

### pom.xml

Verificar/ajustar la versión de Java:

```xml
<properties>
    <java.version>25</java.version>
</properties>
```

Agregar la dependencia de pluralize (necesaria para el shared):

```xml
<!-- https://mvnrepository.com/artifact/io.github.encryptorcode/pluralize -->
<dependency>
    <groupId>io.github.encryptorcode</groupId>
    <artifactId>pluralize</artifactId>
    <version>1.0.0</version>
</dependency>
```

Verificar que springdoc esté presente (si no lo generó automáticamente):

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.8</version>
</dependency>
```

### Archivo `application.properties`

Ubicado en `src/main/resources/application.properties`. Reemplazar todo el contenido:

```properties
spring.application.name=pc27385u202014215

# Spring DataSource Configuration
###    JDBC : SGDB :// HOST : PORT / DB
spring.datasource.url: jdbc:postgresql://localhost:5432/therapy
spring.datasource.username: postgres
spring.datasource.password: 12345678
spring.datasource.driver-class-name: org.postgresql.Driver

# Spring Data JPA Configuration
spring.jpa.database: postgresql
spring.jpa.show-sql: true

# Spring Data JPA Hibernate Configuration
spring.jpa.hibernate.ddl-auto: update
spring.jpa.open-in-view=true
spring.jpa.properties.hibernate.format_sql: true
spring.jpa.properties.hibernate.dialect: org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.naming.physical-strategy=pe.menteclara.platform.u202014215.shared.infrastructure.persistence.jpa.configuration.strategy.SnakeCaseWithPluralizedTablePhysicalNamingStrategy

server.port: 8095

# Elements take their values from maven pom.xml build-related information
documentation.application.description=@project.description@
documentation.application.version=@project.version@

# i18n message bundles
spring.messages.basename=messages
spring.messages.encoding=UTF-8
```

### Clase principal de la aplicación

Abrir `Pc27385u202014215Application.java` (o como se llame) y agregar `@EnableJpaAuditing`:

```java
package pe.menteclara.platform.u202014215;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@SpringBootApplication
@EnableJpaAuditing
public class Pc27385u202014215Application {
    public static void main(String[] args) {
        SpringApplication.run(Pc27385u202014215Application.class, args);
    }
}
```

### Archivos i18n

Crear en `src/main/resources/`:

**`messages.properties`** (inglés, default):
```properties
therapy_session.not_found=TherapySession not found
therapy_session.conflict=A session already exists for this psychologist at the same date and time
validation.psychologist_id.invalid=Invalid psychologist ID format
```

**`messages_es.properties`** (español):
```properties
therapy_session.not_found=Sesión de terapia no encontrada
therapy_session.conflict=Ya existe una sesión para este psicólogo en la misma fecha y hora
validation.psychologist_id.invalid=Formato de ID de psicólogo inválido
```

---

## 3. Copiar el shared

Copiar la carpeta `shared` del proyecto Learning Center Platform dentro de:

```
src/main/java/pe/menteclara/platform/u202014215/
```

Después de copiarla, **buscar y reemplazar** en todos los archivos del shared:

| Buscar | Reemplazar por |
|--------|----------------|
| `com.acme.center.platform` | `pe.menteclara.platform.u202014215` |
| `com.sensibo.platform.u202610123` | `pe.menteclara.platform.u202014215` |

> En IntelliJ: `Edit → Find → Replace in Files` (Ctrl+Shift+R), marcar "In Project".

### Actualizar `OpenApiConfiguration.java`

Ubicado en `shared/infrastructure/documentation/openapi/configuration/`. Reemplazar el contenido del método `learningPlatformOpenApi()`:

```java
@Configuration
public class OpenApiConfiguration {

    @Value("${spring.application.name}")
    String applicationName;

    @Value("${documentation.application.description}")
    String applicationDescription;

    @Value("${documentation.application.version}")
    String applicationVersion;

    @Bean
    public OpenAPI menteClara OpenApi() {
        var openApi = new OpenAPI();
        openApi
            .info(new Info()
                .title(this.applicationName)
                .description(this.applicationDescription)
                .version(this.applicationVersion)
                .contact(new Contact()
                    .name("Mente Clara Consultores")
                    .email("support@menteclara.pe")
                    .url("https://menteclara.pe/support"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0.html")))
            .externalDocs(new ExternalDocumentation()
                .description("Mente Clara Platform wiki")
                .url("https://menteclara.wiki.github.io/docs"));

        openApi.servers(List.of(
            new Server().url("http://localhost:8095").description("Local Development Environment"),
            new Server().url("https://staging-api.menteclara.pe").description("Staging Environment"),
            new Server().url("https://api.menteclara.pe").description("Production Environment")
        ));

        return openApi;
    }
}
```

> ⚠️ Corrige el nombre del método a `menteClaraOpenApi()` (sin espacio en el source real).

---

## 4. Value Objects

Todos van en el package:
`pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects`

### `TherapyType.java` (Enum)

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects;

import java.util.Arrays;

/**
 * Enumeration representing the types of therapy offered by Mente Clara Consultores.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public enum TherapyType {
    COGNITIVE_BEHAVIORAL(1),
    SOLUTION_FOCUSED(2),
    PSYCHODYNAMIC(3),
    GESTALT(4),
    FAMILY_SYSTEMIC(5);

    private final int id;

    TherapyType(int id) { this.id = id; }

    public int getId() { return id; }

    public static TherapyType fromValue(int id) {
        return Arrays.stream(TherapyType.values())
            .filter(t -> t.id == id)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "[TherapyType] Invalid value: " + id));
    }

    public static TherapyType fromString(String name) {
        return Arrays.stream(TherapyType.values())
            .filter(t -> t.name().equalsIgnoreCase(name))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "[TherapyType] Invalid string: " + name));
    }
}
```

### `PsychologistId.java` (Value Object — UUID)

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects;

import java.util.Objects;
import java.util.UUID;

/**
 * Value object representing the identifier of a Psychologist (external bounded context).
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record PsychologistId(UUID value) {

    public PsychologistId {
        if (Objects.isNull(value)) {
            throw new IllegalArgumentException("[PsychologistId] value cannot be null");
        }
    }

    /** Constructor from String — validates UUID format. */
    public PsychologistId(String value) {
        this(parseUUID(value));
    }

    private static UUID parseUUID(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("[PsychologistId] value cannot be null or blank");
        }
        try {
            return UUID.fromString(value);
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("[PsychologistId] Invalid UUID format: " + value);
        }
    }

    /** JPA default constructor */
    public PsychologistId() { this((UUID) null); }
}
```

### `DiagnosisCode.java` (Value Object — Long)

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects;

import java.util.Objects;

/**
 * Value object representing a diagnosis code (external bounded context reference).
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record DiagnosisCode(Long value) {

    public DiagnosisCode {
        if (Objects.isNull(value)) {
            throw new IllegalArgumentException("[DiagnosisCode] value cannot be null");
        }
        if (value <= 0) {
            throw new IllegalArgumentException("[DiagnosisCode] value must be greater than zero");
        }
    }

    /** JPA default constructor */
    public DiagnosisCode() { this(null); }
}
```

---

## 5. Aggregate Root: TherapySession

Package: `pe.menteclara.platform.u202014215.therapy.domain.model.aggregates`

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.aggregates;

import pe.menteclara.platform.u202014215.shared.domain.model.aggregates.AbstractDomainAggregateRoot;
import pe.menteclara.platform.u202014215.therapy.domain.model.commands.CreateTherapySessionCommand;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.DiagnosisCode;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.TherapyType;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * Aggregate root representing a therapy session at Mente Clara Consultores.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Getter
@Setter
public class TherapySession extends AbstractDomainAggregateRoot<TherapySession> {

    private Long id;
    private String patientName;
    private PsychologistId psychologistId;
    private LocalDate sessionDate;
    private LocalTime sessionTime;
    private Integer sessionDurationMinutes;
    private TherapyType therapyType;
    private DiagnosisCode diagnosisCode;
    private String sessionNotes;
    private Float feePaid;

    public TherapySession() {}

    /**
     * Constructs a TherapySession from a CreateTherapySessionCommand.
     *
     * @param command the command with all required session data
     */
    public TherapySession(CreateTherapySessionCommand command) {
        this.patientName = command.patientName();
        this.psychologistId = new PsychologistId(command.psychologistId());
        this.sessionDate = command.sessionDate();
        this.sessionTime = command.sessionTime();
        this.sessionDurationMinutes = command.sessionDurationMinutes();
        this.therapyType = TherapyType.fromString(command.therapyType());
        this.diagnosisCode = new DiagnosisCode(command.diagnosisCode());
        this.sessionNotes = command.sessionNotes();
        this.feePaid = command.feePaid();
    }
}
```

---

## 6. Domain Repository

Package: `pe.menteclara.platform.u202014215.therapy.domain.repositories`

```java
package pe.menteclara.platform.u202014215.therapy.domain.repositories;

import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;

import java.time.LocalDate;
import java.time.LocalTime;
import java.util.Optional;

/**
 * Domain repository interface for TherapySession aggregate.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public interface TherapySessionRepository {

    TherapySession save(TherapySession therapySession);

    Optional<TherapySession> findById(Long id);

    boolean existsByPsychologistIdAndSessionDateAndSessionTime(
        PsychologistId psychologistId, LocalDate sessionDate, LocalTime sessionTime);
}
```

---

## 7. CQRS — Command y Query

### Command

Package: `pe.menteclara.platform.u202014215.therapy.domain.model.commands`

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.commands;

import java.time.LocalDate;
import java.time.LocalTime;
import java.util.UUID;

/**
 * Command to create a new TherapySession.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record CreateTherapySessionCommand(
    String patientName,
    UUID psychologistId,
    LocalDate sessionDate,
    LocalTime sessionTime,
    Integer sessionDurationMinutes,
    String therapyType,
    Long diagnosisCode,
    String sessionNotes,
    Float feePaid
) {
    public CreateTherapySessionCommand {
        if (patientName == null || patientName.isBlank())
            throw new IllegalArgumentException("patientName cannot be null or blank");
        if (psychologistId == null)
            throw new IllegalArgumentException("psychologistId cannot be null");
        if (sessionDate == null)
            throw new IllegalArgumentException("sessionDate cannot be null");
        if (sessionTime == null)
            throw new IllegalArgumentException("sessionTime cannot be null");
        if (sessionDurationMinutes == null)
            throw new IllegalArgumentException("sessionDurationMinutes cannot be null");
        if (therapyType == null || therapyType.isBlank())
            throw new IllegalArgumentException("therapyType cannot be null or blank");
        if (diagnosisCode == null)
            throw new IllegalArgumentException("diagnosisCode cannot be null");
        if (sessionNotes == null || sessionNotes.isBlank())
            throw new IllegalArgumentException("sessionNotes cannot be null or blank");
        if (feePaid == null)
            throw new IllegalArgumentException("feePaid cannot be null");
    }
}
```

### Query

Package: `pe.menteclara.platform.u202014215.therapy.domain.model.queries`

```java
package pe.menteclara.platform.u202014215.therapy.domain.model.queries;

/**
 * Query to get a TherapySession by its ID.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record GetTherapySessionById(Long sessionId) {
    public GetTherapySessionById {
        if (sessionId == null)
            throw new IllegalArgumentException("sessionId cannot be null");
    }
}
```

---

## 8. Application Services

### Interfaces (en `therapy.application.commandservices` y `therapy.application.queryservices`)

```java
// TherapySessionCommandService.java
package pe.menteclara.platform.u202014215.therapy.application.commandservices;

import pe.menteclara.platform.u202014215.shared.application.result.ApplicationError;
import pe.menteclara.platform.u202014215.shared.application.result.Result;
import pe.menteclara.platform.u202014215.therapy.domain.model.commands.CreateTherapySessionCommand;

public interface TherapySessionCommandService {
    Result<Long, ApplicationError> handle(CreateTherapySessionCommand command);
}
```

```java
// TherapySessionQueryService.java
package pe.menteclara.platform.u202014215.therapy.application.queryservices;

import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.queries.GetTherapySessionById;

import java.util.Optional;

public interface TherapySessionQueryService {
    Optional<TherapySession> handle(GetTherapySessionById query);
}
```

### Implementaciones (en `therapy.application.internal.commandservices` y `.queryservices`)

```java
// TherapySessionCommandServiceImpl.java
package pe.menteclara.platform.u202014215.therapy.application.internal.commandservices;

import pe.menteclara.platform.u202014215.shared.application.result.ApplicationError;
import pe.menteclara.platform.u202014215.shared.application.result.Result;
import pe.menteclara.platform.u202014215.therapy.application.commandservices.TherapySessionCommandService;
import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.commands.CreateTherapySessionCommand;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;
import pe.menteclara.platform.u202014215.therapy.domain.repositories.TherapySessionRepository;
import org.springframework.stereotype.Service;

/**
 * Implementation of TherapySessionCommandService.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Service
public class TherapySessionCommandServiceImpl implements TherapySessionCommandService {

    private final TherapySessionRepository therapySessionRepository;

    public TherapySessionCommandServiceImpl(TherapySessionRepository therapySessionRepository) {
        this.therapySessionRepository = therapySessionRepository;
    }

    /**
     * Handles the creation of a new therapy session.
     * Enforces the uniqueness rule: no two sessions for the same psychologist at the same date and time.
     *
     * @param command the command containing session data
     * @return Result with the generated session ID, or an ApplicationError on failure
     */
    @Override
    public Result<Long, ApplicationError> handle(CreateTherapySessionCommand command) {
        var psychologistId = new PsychologistId(command.psychologistId());

        if (therapySessionRepository.existsByPsychologistIdAndSessionDateAndSessionTime(
                psychologistId, command.sessionDate(), command.sessionTime())) {
            return Result.failure(ApplicationError.conflict(
                "TherapySession",
                "A session already exists for this psychologist at the same date and time"
            ));
        }

        var session = new TherapySession(command);
        try {
            session = therapySessionRepository.save(session);
        } catch (Exception e) {
            return Result.failure(ApplicationError.unexpected("create-therapy-session", e.getMessage()));
        }
        return Result.success(session.getId());
    }
}
```

```java
// TherapySessionQueryServiceImpl.java
package pe.menteclara.platform.u202014215.therapy.application.internal.queryservices;

import pe.menteclara.platform.u202014215.therapy.application.queryservices.TherapySessionQueryService;
import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.queries.GetTherapySessionById;
import pe.menteclara.platform.u202014215.therapy.domain.repositories.TherapySessionRepository;
import org.springframework.stereotype.Service;

import java.util.Optional;

/**
 * Implementation of TherapySessionQueryService.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Service
public class TherapySessionQueryServiceImpl implements TherapySessionQueryService {

    private final TherapySessionRepository therapySessionRepository;

    public TherapySessionQueryServiceImpl(TherapySessionRepository therapySessionRepository) {
        this.therapySessionRepository = therapySessionRepository;
    }

    @Override
    public Optional<TherapySession> handle(GetTherapySessionById query) {
        return therapySessionRepository.findById(query.sessionId());
    }
}
```

---

## 9. Infrastructure — Persistence

### Converters

Package: `pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.converters`

```java
// PsychologistIdPersistenceConverter.java
@Converter(autoApply = true)
public class PsychologistIdPersistenceConverter implements AttributeConverter<PsychologistId, String> {
    @Override
    public String convertToDatabaseColumn(PsychologistId attribute) {
        return attribute == null ? null : attribute.value().toString();
    }
    @Override
    public PsychologistId convertToEntityAttribute(String dbData) {
        return dbData == null ? null : new PsychologistId(UUID.fromString(dbData));
    }
}
```

```java
// DiagnosisCodePersistenceConverter.java
@Converter(autoApply = true)
public class DiagnosisCodePersistenceConverter implements AttributeConverter<DiagnosisCode, Long> {
    @Override
    public Long convertToDatabaseColumn(DiagnosisCode attribute) {
        return attribute == null ? null : attribute.value();
    }
    @Override
    public DiagnosisCode convertToEntityAttribute(Long dbData) {
        return dbData == null ? null : new DiagnosisCode(dbData);
    }
}
```

### Persistence Entity

Package: `pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.entities`

```java
package pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.entities;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import pe.menteclara.platform.u202014215.shared.infrastructure.persistence.jpa.entities.AuditableAbstractPersistenceEntity;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.DiagnosisCode;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.TherapyType;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.converters.DiagnosisCodePersistenceConverter;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.converters.PsychologistIdPersistenceConverter;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * JPA persistence entity for TherapySession aggregate.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Entity
@Table(name = "therapy_sessions")
@Getter
@Setter
public class TherapySessionPersistenceEntity extends AuditableAbstractPersistenceEntity {

    @Column(name = "patient_name", length = 100, nullable = false)
    private String patientName;

    @Convert(converter = PsychologistIdPersistenceConverter.class)
    @Column(name = "psychologist_id", nullable = false)
    private PsychologistId psychologistId;

    @Column(name = "session_date", nullable = false)
    private LocalDate sessionDate;

    @Column(name = "session_time", nullable = false)
    private LocalTime sessionTime;

    @Column(name = "session_duration_minutes", nullable = false)
    private Integer sessionDurationMinutes;

    @Enumerated(EnumType.ORDINAL)
    @Column(name = "therapy_type", nullable = false)
    private TherapyType therapyType;

    @Convert(converter = DiagnosisCodePersistenceConverter.class)
    @Column(name = "diagnosis_code", nullable = false)
    private DiagnosisCode diagnosisCode;

    @Column(name = "session_notes", length = 500, nullable = false)
    private String sessionNotes;

    @Column(name = "fee_paid", nullable = false)
    private Float feePaid;
}
```

### JPA Repository (Spring Data)

Package: `pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.repositories`

```java
package pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.repositories;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.entities.TherapySessionPersistenceEntity;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * Spring Data JPA repository for TherapySessionPersistenceEntity.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Repository
public interface TherapySessionPersistenceRepository
    extends JpaRepository<TherapySessionPersistenceEntity, Long> {

    boolean existsByPsychologistIdAndSessionDateAndSessionTime(
        PsychologistId psychologistId, LocalDate sessionDate, LocalTime sessionTime);
}
```

### Persistence Assembler

Package: `pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.assemblers`

```java
package pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.assemblers;

import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.entities.TherapySessionPersistenceEntity;

/**
 * Assembler to convert between TherapySession domain object and TherapySessionPersistenceEntity.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public final class TherapySessionPersistenceAssembler {

    private TherapySessionPersistenceAssembler() {}

    public static TherapySessionPersistenceEntity toPersistenceFromDomain(TherapySession session) {
        var entity = new TherapySessionPersistenceEntity();
        entity.setId(session.getId());
        entity.setPatientName(session.getPatientName());
        entity.setPsychologistId(session.getPsychologistId());
        entity.setSessionDate(session.getSessionDate());
        entity.setSessionTime(session.getSessionTime());
        entity.setSessionDurationMinutes(session.getSessionDurationMinutes());
        entity.setTherapyType(session.getTherapyType());
        entity.setDiagnosisCode(session.getDiagnosisCode());
        entity.setSessionNotes(session.getSessionNotes());
        entity.setFeePaid(session.getFeePaid());
        return entity;
    }

    public static TherapySession toDomainFromPersistence(TherapySessionPersistenceEntity entity) {
        var session = new TherapySession();
        session.setId(entity.getId());
        session.setPatientName(entity.getPatientName());
        session.setPsychologistId(entity.getPsychologistId());
        session.setSessionDate(entity.getSessionDate());
        session.setSessionTime(entity.getSessionTime());
        session.setSessionDurationMinutes(entity.getSessionDurationMinutes());
        session.setTherapyType(entity.getTherapyType());
        session.setDiagnosisCode(entity.getDiagnosisCode());
        session.setSessionNotes(entity.getSessionNotes());
        session.setFeePaid(entity.getFeePaid());
        return session;
    }
}
```

### Repository Adapter

Package: `pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.adapters`

```java
package pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.adapters;

import org.springframework.stereotype.Repository;
import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.valueobjects.PsychologistId;
import pe.menteclara.platform.u202014215.therapy.domain.repositories.TherapySessionRepository;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.assemblers.TherapySessionPersistenceAssembler;
import pe.menteclara.platform.u202014215.therapy.infrastructure.persistence.jpa.repositories.TherapySessionPersistenceRepository;

import java.time.LocalDate;
import java.time.LocalTime;
import java.util.Optional;

/**
 * Adapter implementing the domain TherapySessionRepository using Spring Data JPA.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Repository
public class TherapySessionRepositoryImpl implements TherapySessionRepository {

    private final TherapySessionPersistenceRepository persistenceRepository;

    public TherapySessionRepositoryImpl(TherapySessionPersistenceRepository persistenceRepository) {
        this.persistenceRepository = persistenceRepository;
    }

    @Override
    public TherapySession save(TherapySession therapySession) {
        var saved = persistenceRepository.save(
            TherapySessionPersistenceAssembler.toPersistenceFromDomain(therapySession));
        return TherapySessionPersistenceAssembler.toDomainFromPersistence(saved);
    }

    @Override
    public Optional<TherapySession> findById(Long id) {
        return persistenceRepository.findById(id)
            .map(TherapySessionPersistenceAssembler::toDomainFromPersistence);
    }

    @Override
    public boolean existsByPsychologistIdAndSessionDateAndSessionTime(
            PsychologistId psychologistId, LocalDate sessionDate, LocalTime sessionTime) {
        return persistenceRepository.existsByPsychologistIdAndSessionDateAndSessionTime(
            psychologistId, sessionDate, sessionTime);
    }
}
```

---

## 10. Interfaces REST

### Resources

Package: `pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources`

```java
// CreateTherapySessionResource.java
package pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * Resource record for creating a new TherapySession.
 * Note: psychologistId comes from the URL path, not this body.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record CreateTherapySessionResource(
    String patientName,
    LocalDate sessionDate,
    LocalTime sessionTime,
    Integer sessionDurationMinutes,
    String therapyType,
    Long diagnosisCode,
    String sessionNotes,
    Float feePaid
) {}
```

```java
// TherapySessionResource.java
package pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources;

import java.time.LocalDate;
import java.time.LocalTime;

/**
 * Resource record returned in API responses for a TherapySession.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record TherapySessionResource(
    Long id,
    String patientName,
    String psychologistId,
    LocalDate sessionDate,
    LocalTime sessionTime,
    Integer sessionDurationMinutes,
    String therapyType,
    Long diagnosisCode,
    String sessionNotes,
    Float feePaid
) {}
```

### Assemblers (REST layer)

Package: `pe.menteclara.platform.u202014215.therapy.interfaces.rest.transform`

```java
// CreateTherapySessionCommandFromResourceAssembler.java
package pe.menteclara.platform.u202014215.therapy.interfaces.rest.transform;

import pe.menteclara.platform.u202014215.therapy.domain.model.commands.CreateTherapySessionCommand;
import pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources.CreateTherapySessionResource;

import java.util.UUID;

/**
 * Assembler to convert CreateTherapySessionResource + path psychologistId into a command.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public class CreateTherapySessionCommandFromResourceAssembler {

    public static CreateTherapySessionCommand toCommandFromResource(
            CreateTherapySessionResource resource, UUID psychologistId) {
        return new CreateTherapySessionCommand(
            resource.patientName(),
            psychologistId,
            resource.sessionDate(),
            resource.sessionTime(),
            resource.sessionDurationMinutes(),
            resource.therapyType(),
            resource.diagnosisCode(),
            resource.sessionNotes(),
            resource.feePaid()
        );
    }
}
```

```java
// TherapySessionResourceFromEntityAssembler.java
package pe.menteclara.platform.u202014215.therapy.interfaces.rest.transform;

import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources.TherapySessionResource;

/**
 * Assembler to convert a TherapySession domain object into a TherapySessionResource.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public class TherapySessionResourceFromEntityAssembler {

    public static TherapySessionResource toResourceFromEntity(TherapySession entity) {
        return new TherapySessionResource(
            entity.getId(),
            entity.getPatientName(),
            entity.getPsychologistId().value().toString(),
            entity.getSessionDate(),
            entity.getSessionTime(),
            entity.getSessionDurationMinutes(),
            entity.getTherapyType().name(),
            entity.getDiagnosisCode().value(),
            entity.getSessionNotes(),
            entity.getFeePaid()
        );
    }
}
```

### Controller

Package: `pe.menteclara.platform.u202014215.therapy.interfaces.rest`

> ⚠️ **Punto clave del enunciado:** el `psychologistId` viene en el PATH, no en el body. Además el sistema debe validar que sea un UUID válido.

```java
package pe.menteclara.platform.u202014215.therapy.interfaces.rest;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import pe.menteclara.platform.u202014215.shared.application.result.ApplicationError;
import pe.menteclara.platform.u202014215.shared.application.result.Result;
import pe.menteclara.platform.u202014215.shared.interfaces.rest.transform.ResponseEntityAssembler;
import pe.menteclara.platform.u202014215.therapy.application.commandservices.TherapySessionCommandService;
import pe.menteclara.platform.u202014215.therapy.application.queryservices.TherapySessionQueryService;
import pe.menteclara.platform.u202014215.therapy.domain.model.aggregates.TherapySession;
import pe.menteclara.platform.u202014215.therapy.domain.model.queries.GetTherapySessionById;
import pe.menteclara.platform.u202014215.therapy.interfaces.rest.resources.CreateTherapySessionResource;
import pe.menteclara.platform.u202014215.therapy.interfaces.rest.transform.CreateTherapySessionCommandFromResourceAssembler;
import pe.menteclara.platform.u202014215.therapy.interfaces.rest.transform.TherapySessionResourceFromEntityAssembler;

import java.util.UUID;

/**
 * REST controller for managing therapy sessions.
 * Endpoint: /api/v1/psychologist/{psychologistId}/sessions
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@RestController
@RequestMapping(value = "/api/v1/psychologist/{psychologistId}/sessions")
@Tag(name = "Therapy Sessions", description = "Therapy session management endpoints")
public class TherapySessionsController {

    private final TherapySessionCommandService therapySessionCommandService;
    private final TherapySessionQueryService therapySessionQueryService;

    public TherapySessionsController(
            TherapySessionCommandService therapySessionCommandService,
            TherapySessionQueryService therapySessionQueryService) {
        this.therapySessionCommandService = therapySessionCommandService;
        this.therapySessionQueryService = therapySessionQueryService;
    }

    /**
     * Creates a new therapy session for a specific psychologist.
     *
     * @param psychologistId the UUID of the psychologist (from path)
     * @param resource the session data from the request body
     * @return HTTP 201 with created session, or error response
     */
    @PostMapping
    @Operation(
        summary = "Create a new therapy session",
        description = "Creates a new therapy session record for the specified psychologist."
    )
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Therapy session created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input data"),
        @ApiResponse(responseCode = "409", description = "Conflict - Session already exists for this psychologist at the same date and time")
    })
    public ResponseEntity<?> createTherapySession(
            @PathVariable String psychologistId,
            @RequestBody CreateTherapySessionResource resource) {

        UUID psychologistUUID;
        try {
            psychologistUUID = UUID.fromString(psychologistId);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body("Invalid psychologist ID format. Must be a valid UUID.");
        }

        var command = CreateTherapySessionCommandFromResourceAssembler
            .toCommandFromResource(resource, psychologistUUID);

        var result = this.therapySessionCommandService.handle(command)
            .flatMap(sessionId -> this.therapySessionQueryService
                .handle(new GetTherapySessionById(sessionId))
                .<Result<TherapySession, ApplicationError>>map(Result::success)
                .orElseGet(() -> Result.failure(
                    ApplicationError.notFound("TherapySession", sessionId.toString()))));

        return ResponseEntityAssembler.toResponseEntityFromResult(
            result,
            TherapySessionResourceFromEntityAssembler::toResourceFromEntity,
            HttpStatus.CREATED
        );
    }
}
```

---

## 11. OpenAPI (Swagger)

El fragmento de documentación que el profesor mostró en clase corresponde al bloque de anotaciones `@Operation` y `@ApiResponses` que se ve en el controller de arriba (líneas equivalentes a las 55–69 del ejemplo del profe). El bloque seleccionado era:

```java
@Operation(
    summary = "Create a new therapy session",
    description = "Creates a new therapy session record for the specified psychologist."
)
@ApiResponses(value = {
    @ApiResponse(responseCode = "201", description = "Therapy session created successfully"),
    @ApiResponse(responseCode = "400", description = "Invalid input data"),
    @ApiResponse(responseCode = "409", description = "Conflict - Session already exists for this psychologist at the same date and time")
})
```

Esto va **justo antes del método** en el controller, como ya está incluido arriba.

Una vez corriendo el proyecto, acceder a la documentación en:
- **Swagger UI:** `http://localhost:8095/swagger-ui/index.html`
- **OpenAPI JSON:** `http://localhost:8095/v3/api-docs`

---

## 12. README.md

Crear/reemplazar el archivo `README.md` en la raíz del proyecto:

```markdown
# Mente Clara Consultores — Therapy Session Platform

RESTful API for managing therapy sessions at Mente Clara Consultores, a psychology and mental wellness clinic in Miraflores, Lima, Peru.

## Description

This backend application provides endpoints to register and manage therapy sessions (TherapySession) following Domain-Driven Design principles, CQRS pattern, and a layered architecture (domain, application, interfaces, infrastructure).

## Tech Stack

- **Language:** Java 25
- **Framework:** Spring Boot 3.5
- **Database:** PostgreSQL (schema: `therapy`)
- **Persistence:** Spring Data JPA / Hibernate
- **Documentation:** SpringDoc OpenAPI (Swagger UI)
- **Build Tool:** Maven

## Getting Started

### Prerequisites

- Java 25
- PostgreSQL running on `localhost:5432`
- Database `therapy` created

### Run

```bash
./mvnw spring-boot:run
```

### API Documentation

- **Swagger UI:** http://localhost:8095/swagger-ui/index.html
- **OpenAPI JSON:** http://localhost:8095/v3/api-docs

## Endpoint

| Method | URL | Description |
|--------|-----|-------------|
| POST | `/api/v1/psychologist/{psychologistId}/sessions` | Create a new therapy session |

## Author

Rafael Agustin Pacheco Lavado — u202014215
```

---

## 13. Empaquetado final

1. Verificar que el proyecto **compila y corre** sin errores.
2. Probar el endpoint con Swagger o Postman:
   - **URL:** `POST http://localhost:8095/api/v1/psychologist/{uuid}/sessions`
   - Ejemplo de UUID válido en el path: `550e8400-e29b-41d4-a716-446655440000`
   - Body JSON de ejemplo:
```json
{
  "patientName": "Juan Perez",
  "sessionDate": "2025-11-20",
  "sessionTime": "10:00:00",
  "sessionDurationMinutes": 60,
  "therapyType": "COGNITIVE_BEHAVIORAL",
  "diagnosisCode": 1001,
  "sessionNotes": "First session, patient presented with anxiety symptoms.",
  "feePaid": 150.00
}
```
3. Comprimir el proyecto como `.zip`:
   - Nombre: `pc2<NRC>u<código>.zip` — ejemplo: `pc27385u202014215.zip`
   - Incluir toda la carpeta del proyecto (sin la carpeta `target/` si es posible para reducir tamaño)
4. Subir a la actividad indicada en el aula virtual.

---

## 14. Checklist final

- [ ] pgAdmin: base de datos `therapy` creada
- [ ] Puerto configurado en `8095`
- [ ] Package raíz: `pe.menteclara.platform.u<código>`
- [ ] `@EnableJpaAuditing` en la clase principal
- [ ] Shared copiado y packages actualizados
- [ ] Value Objects: `TherapyType` (enum), `PsychologistId` (UUID), `DiagnosisCode` (Long)
- [ ] Aggregate root `TherapySession` extiende `AbstractDomainAggregateRoot`
- [ ] Command `CreateTherapySessionCommand` con validaciones
- [ ] Query `GetTherapySessionById`
- [ ] `TherapySessionRepository` (interfaz de dominio)
- [ ] `TherapySessionCommandServiceImpl` con regla de negocio (no duplicados)
- [ ] `TherapySessionQueryServiceImpl`
- [ ] Persistence entity con `@Entity`, `@Table(name="therapy_sessions")`, extiende `AuditableAbstractPersistenceEntity`
- [ ] Converters para `PsychologistId` y `DiagnosisCode`
- [ ] `TherapySessionPersistenceRepository` con método `existsByPsychologistIdAndSessionDateAndSessionTime`
- [ ] `TherapySessionRepositoryImpl` (adapter)
- [ ] `TherapySessionPersistenceAssembler`
- [ ] Resources: `CreateTherapySessionResource` y `TherapySessionResource`
- [ ] Assemblers REST: `CreateTherapySessionCommandFromResourceAssembler`, `TherapySessionResourceFromEntityAssembler`
- [ ] Controller con `@PathVariable psychologistId`, validación UUID, anotaciones Swagger
- [ ] `OpenApiConfiguration` actualizada con datos de Mente Clara
- [ ] `messages.properties` (EN) y `messages_es.properties` (ES)
- [ ] `README.md` en inglés con nombre como autor
- [ ] Proyecto corre en `localhost:8095` y Swagger accesible
- [ ] Zip nombrado correctamente: `pc2<NRC>u<código>.zip`

---

> **Estructura de packages resumen:**
> ```
> pe.menteclara.platform.u<código>
> ├── shared/                          ← copiado del Learning Center
> └── therapy/
>     ├── domain/
>     │   ├── model/
>     │   │   ├── aggregates/          TherapySession
>     │   │   ├── commands/            CreateTherapySessionCommand
>     │   │   ├── queries/             GetTherapySessionById
>     │   │   └── valueobjects/        TherapyType, PsychologistId, DiagnosisCode
>     │   └── repositories/            TherapySessionRepository (interfaz)
>     ├── application/
>     │   ├── commandservices/         TherapySessionCommandService (interfaz)
>     │   ├── queryservices/           TherapySessionQueryService (interfaz)
>     │   └── internal/
>     │       ├── commandservices/     TherapySessionCommandServiceImpl
>     │       └── queryservices/       TherapySessionQueryServiceImpl
>     ├── infrastructure/
>     │   └── persistence/jpa/
>     │       ├── adapters/            TherapySessionRepositoryImpl
>     │       ├── assemblers/          TherapySessionPersistenceAssembler
>     │       ├── converters/          PsychologistIdPersistenceConverter, DiagnosisCodePersistenceConverter
>     │       ├── entities/            TherapySessionPersistenceEntity
>     │       └── repositories/       TherapySessionPersistenceRepository
>     └── interfaces/
>         └── rest/
>             ├── TherapySessionsController
>             ├── resources/           CreateTherapySessionResource, TherapySessionResource
>             └── transform/           CreateTherapySessionCommandFromResourceAssembler,
>                                      TherapySessionResourceFromEntityAssembler
> ```
