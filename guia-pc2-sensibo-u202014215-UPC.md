# Guía PC2 — Sensibo Climate Control
## Desarrollo de Aplicaciones Open Source (1ASI0729) — Sección 7377

> Esta guía sigue el mismo flujo de trabajo usado en clase (modelo `daos-sensibo`), pero
> personalizada con los datos del proyecto y completada hasta el endpoint final exigido
> por el enunciado de la PC2 (`POST /api/v1/devices`).

### Datos del proyecto

| Dato | Valor |
|---|---|
| Estudiante | Rafael Agustin Pacheco Lavado |
| Código | `u202014215` |
| NRC | `11990` |
| Nombre del proyecto (zip de entrega) | `pc211990u202014215.zip` |
| groupId | `com.sensibo.platform` |
| artifactId | `pc211990u202014215` |
| packageName (raíz) | `com.sensibo.platform.u202014215` |
| Puerto | `8096` |
| Esquema BD | `sensibo` |
| Bounded Context del enunciado | `registry` |

---

## 1. Configuración inicial

### 1.1 Base de datos

**Abrir** `pgAdmin`, ingresar la contraseña `12345678` y **crear** la base de datos: `sensibo`.

### 1.2 Creación del project

**Cargar** el navegador y **pegar** el siguiente enlace (nota: se usa Java 26 con Spring
Boot 3.5.6, no las versiones del modelo de clase):

```
https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.5.6&packaging=jar&configurationFileFormat=properties&jvmVersion=26&groupId=com.sensibo.platform&artifactId=pc211990u202014215&packageName=com.sensibo.platform.u202014215&dependencies=data-jpa,validation,web,devtools,postgresql,lombok,springdoc-openapi
```

**Generar** el project Spring, **guardarlo** en la carpeta `IdeaProjects\1ASI0729\` y luego
**abrirlo** en `IntelliJ IDEA`.

### 1.3 Configuración del SDK

**Hacer** click en `File` → `Project Structure`. **Seleccionar** `Project` en `Project
Settings` y elegir la versión `26` de Java como `SDK` (descargarla si no está instalada).
Luego **seleccionar** `SDKs` en `Platform Settings` y verificar la versión 26. **Hacer**
click en `OK`.

### 1.4 Archivo pom.xml

**Abrir** `pom.xml` y **modificar** la versión de Java:

```xml
<properties>
    <java.version>26</java.version>
</properties>
```

**Agregar** la dependencia necesaria para el physical naming strategy (pluralización de
tablas), usada por el `shared`:

```xml
<!-- https://mvnrepository.com/artifact/io.github.encryptorcode/pluralize -->
<dependency>
    <groupId>io.github.encryptorcode</groupId>
    <artifactId>pluralize</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 1.5 Clase principal de la aplicación

**Abrir** `Pc211990u202014215Application.java` (paquete raíz `com.sensibo.platform.u202014215`)
y **agregar** el import y la anotación de auditoría JPA:

```java
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@SpringBootApplication
@EnableJpaAuditing
public class Pc211990u202014215Application {
    public static void main(String[] args) {
        SpringApplication.run(Pc211990u202014215Application.class, args);
    }
}
```

### 1.6 Archivo application.properties

**Abrir** `application.properties` en `src\main\resources` y **reemplazar** el contenido:

```ini
# Spring DataSource Configuration
###    JDBC : SGDB :// HOST : PORT / DB
spring.datasource.url=jdbc:postgresql://localhost:5432/sensibo
spring.datasource.username=postgres
spring.datasource.password=12345678
spring.datasource.driver-class-name=org.postgresql.Driver

# Spring Data JPA Configuration
spring.jpa.database=postgresql
spring.jpa.show-sql=true

# Spring Data JPA Hibernate Configuration
spring.jpa.hibernate.ddl-auto=update
spring.jpa.open-in-view=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.naming.physical-strategy=com.sensibo.platform.u202014215.shared.infrastructure.persistence.jpa.configuration.strategy.SnakeCaseWithPluralizedTablePhysicalNamingStrategy

server.port=8096

# Elements take their values from maven pom.xml build-related information
documentation.application.description=@project.description@
documentation.application.version=@project.version@

# i18n message bundles
spring.messages.basename=messages
spring.messages.encoding=UTF-8
```

> ⚠️ El esquema de BD pedido por el enunciado es `sensibo` y la base de datos también se
> llama `sensibo`, por eso basta con la URL `jdbc:postgresql://localhost:5432/sensibo`.

---

## 2. Shared package

Ya tienes el `shared` descargado (`shared.zip`), así que **no** lo copias desde el repo del
Learning Center, sino desde tu zip:

1. **Descomprimir** `shared.zip`.
2. **Copiar** la carpeta `shared` completa dentro de
   `src/main/java/com/sensibo/platform/u202014215/`.
3. **Reemplazar en todos los archivos** el paquete original por el de tu proyecto. En
   IntelliJ: `Edit` → `Find` → `Replace in Files`, con scope la carpeta `shared`:

   - Buscar: `com.acme.center.platform.shared`
   - Reemplazar con: `com.sensibo.platform.u202014215.shared`

Esto deja disponibles los siguientes elementos (ya implementados, no se reescriben):

| Clase | Paquete | Propósito |
|---|---|---|
| `AbstractDomainAggregateRoot` | `shared.domain.model.aggregates` | Superclase de todo aggregate root (eventos de dominio, sin JPA) |
| `AuditableAbstractPersistenceEntity` | `shared.infrastructure.persistence.jpa.entities` | Superclase de las entidades JPA: `id`, `createdAt`, `updatedAt` |
| `SnakeCaseWithPluralizedTablePhysicalNamingStrategy` | `shared.infrastructure.persistence.jpa.configuration.strategy` | Naming strategy: snake_case + tablas en plural |
| `OpenApiConfiguration` | `shared.infrastructure.documentation.openapi.configuration` | Configuración de Swagger/OpenAPI |
| `LocaleConfiguration` | `shared.infrastructure.i18n.configuration` | Resolver de locale por header `Accept-Language` (EN por defecto, soporta ES) |
| `Result<T, E>` | `shared.application.result` | Tipo sellado `Success`/`Failure` para CQRS sin excepciones |
| `ApplicationError` | `shared.application.result` | Error de aplicación con `code`, `message`, `details` + factory methods (`notFound`, `conflict`, `validationError`, `businessRuleViolation`, `unexpected`) |
| `GlobalExceptionHandler` | `shared.interfaces.rest` | `@RestControllerAdvice` que traduce excepciones a `ErrorResource` |
| `ErrorResource` / `MessageResource` | `shared.interfaces.rest.resources` | DTOs de respuesta de error / mensaje |
| `ErrorResponseAssembler` | `shared.interfaces.rest.transform` | Mapea `ApplicationError` → `ResponseEntity<ErrorResource>` con i18n |
| `ResponseEntityAssembler` | `shared.interfaces.rest.transform` | Mapea `Result<T, ApplicationError>` → `ResponseEntity<?>` |

### 2.1 Actualizar el `OpenApiConfiguration`

Dentro de `shared.infrastructure.documentation.openapi.configuration`, **actualizar** el
contenido descriptivo del proyecto (servers, contacto):

```java
@Bean
public OpenAPI sensiboPlatformOpenApi() {
    var openApi = new OpenAPI();
    openApi
        .info(new Info()
            .title(this.applicationName)
            .description(this.applicationDescription)
            .version(this.applicationVersion)
            .contact(new Contact()
                .name("Sensibo Climate Control")
                .email("support@sensibo.com")
                .url("https://sensibo.com/support"))
            .license(new License()
                .name("Apache 2.0")
                .url("https://www.apache.org/licenses/LICENSE-2.0.html")))
        .externalDocs(new ExternalDocumentation()
            .description("Sensibo Climate Control Platform wiki Documentation")
            .url("https://sensibo.wiki.github.io/docs"));

    openApi.servers(List.of(
        new Server().url("http://localhost:8096").description("Local Development Environment"),
        new Server().url("https://staging-api.sensibo.com").description("Staging Environment"),
        new Server().url("https://api.sensibo.com").description("Production Environment")
    ));

    return openApi;
}
```

### 2.2 Mensajes i18n (`messages.properties` / `messages_es.properties`)

El `ErrorResponseAssembler` y el `GlobalExceptionHandler` leen un bundle `messages`. Como
el enunciado pide soporte EN/ES vía `Accept-Language`, **crear** en `src/main/resources`:

`messages.properties` (inglés, default):
```properties
error.validation.message=Validation failed
error.business-rule.message=Business rule violation
error.unexpected.message=An unexpected error occurred
error.not-found.message={0} not found
error.conflict.message=Conflict with {0}
error.generic.message=An error occurred
validation.field.prefix=Field
validation.request.failed=Request validation failed
validation.request.argument=request-argument
error.unexpected.context=global-exception-handler
error.device-mac-address-already-registered.message=A device with this MAC address is already registered
```

`messages_es.properties` (español):
```properties
error.validation.message=Error de validación
error.business-rule.message=Violación de regla de negocio
error.unexpected.message=Ocurrió un error inesperado
error.not-found.message=No se encontró {0}
error.conflict.message=Conflicto con {0}
error.generic.message=Ocurrió un error
validation.field.prefix=Campo
validation.request.failed=La validación de la solicitud falló
validation.request.argument=argumento-de-la-solicitud
error.unexpected.context=manejador-global-de-excepciones
error.device-mac-address-already-registered.message=Ya existe un dispositivo registrado con esta dirección MAC
```

---

## 3. Bounded Context `registry`

### 3.1 Estructura del Bounded Context

**Descomprimir** `bounded.zip` (plantilla vacía de un Bounded Context con arquitectura por
capas + DDD táctico). **Copiar** esa carpeta dentro de
`src/main/java/com/sensibo/platform/u202014215/` y **renombrarla** de `bounded` a `registry`:

```
registry
├── application
│   ├── acl
│   ├── commandservices
│   ├── internal
│   │   ├── commandservices
│   │   ├── eventhandlers
│   │   ├── outboundservices
│   │   │   └── acl
│   │   └── queryservices
│   └── queryservices
├── domain
│   ├── exceptions
│   ├── model
│   │   ├── aggregates
│   │   ├── commands
│   │   ├── entities
│   │   ├── events
│   │   ├── queries
│   │   └── valueobjects
│   └── repositories
├── infrastructure
│   └── persistence
│       └── jpa
│           ├── adapters
│           ├── assemblers
│           ├── converters
│           ├── embeddables
│           ├── entities
│           └── repositories
└── interfaces
    ├── acl
    ├── events
    └── rest
        ├── resources
        └── transform
```

### 3.2 `domain.model.valueobjects`

**Crear** el enum `DeviceTypes`:

```java
package com.sensibo.platform.u202014215.registry.domain.model.valueobjects;

import java.util.Arrays;

/**
 * Enumerates the supported Sensibo device types.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public enum DeviceTypes {
    SMART_AC_CONTROLLER(1),
    ROOM_SENSOR(2),
    AIR_QUALITY_MONITOR(3),
    DOOR_WINDOW_SENSOR(4);

    private final int id;

    DeviceTypes(int id) { this.id = id; }

    public int getId() { return id; }

    public static DeviceTypes fromValue(int id) {
        return Arrays.stream(DeviceTypes.values())
            .filter(dt -> dt.id == id)
            .findFirst()
            .orElseThrow(() ->
                new IllegalArgumentException("[DeviceTypes] Invalid value for DeviceTypes: " + id));
    }

    public static DeviceTypes fromString(String name) {
        return Arrays.stream(DeviceTypes.values())
            .filter(dt -> dt.name().equalsIgnoreCase(name))
            .findFirst()
            .orElseThrow(() ->
                new IllegalArgumentException("[DeviceTypes] Invalid string for DeviceTypes: " + name));
    }
}
```

**Crear** el record `SensiboUserId`:

```java
package com.sensibo.platform.u202014215.registry.domain.model.valueobjects;

import java.util.Objects;

/**
 * Value object representing the identifier of a Sensibo platform user.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record SensiboUserId(Long value) {

    public SensiboUserId {
        if (Objects.isNull(value)) {
            throw new IllegalArgumentException("[SensiboUserId] value cannot be null");
        }
        if (value <= 0) {
            throw new IllegalArgumentException("[SensiboUserId] value must be greater than zero");
        }
    }

    public SensiboUserId() { this(null); }
}
```

**Crear** el record `MacAddress`:

```java
package com.sensibo.platform.u202014215.registry.domain.model.valueobjects;

import java.util.Objects;

/**
 * Value object representing a device's MAC address in standard hexadecimal format.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record MacAddress(String value) {

    private static final String MAC_REGEX = "^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$";

    public MacAddress {
        if (Objects.isNull(value) || value.isBlank()) {
            throw new IllegalArgumentException("[MacAddress] value cannot be null or blank");
        }
        if (!value.matches(MAC_REGEX)) {
            throw new IllegalArgumentException(
                "[MacAddress] Invalid MAC address format. Expected AA:BB:CC:DD:EE:FF");
        }
    }

    public MacAddress() { this(null); }
}
```

**Crear** el record `SerialNumber` (autogenerado, UUID):

```java
package com.sensibo.platform.u202014215.registry.domain.model.valueobjects;

import java.util.Objects;
import java.util.UUID;

/**
 * Value object representing the unique, auto-generated serial number of a device.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record SerialNumber(String value) {

    public SerialNumber {
        if (Objects.isNull(value) || value.isBlank()) {
            throw new IllegalArgumentException("[SerialNumber] value cannot be null or blank");
        }
        if (value.length() != 36) {
            throw new IllegalArgumentException("[SerialNumber] must be 36 characters long");
        }
        if (!value.matches("[a-f0-9]{8}-([a-f0-9]{4}-){3}[a-f0-9]{12}")) {
            throw new IllegalArgumentException("[SerialNumber] must be a valid UUID");
        }
    }

    public SerialNumber() {
        this(UUID.randomUUID().toString());
    }
}
```

**Crear** el record `Version`:

```java
package com.sensibo.platform.u202014215.registry.domain.model.valueobjects;

import java.util.Objects;

/**
 * Value object representing a firmware version in X.Y.Z format.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record Version(String value) {

    private static final String VERSION_REGEX = "^\\d+\\.\\d+\\.\\d+$";

    public Version {
        if (Objects.isNull(value) || value.isBlank()) {
            throw new IllegalArgumentException("[Version] value cannot be null or blank");
        }
        if (!value.matches(VERSION_REGEX)) {
            throw new IllegalArgumentException("[Version] Invalid version format. Expected X.Y.Z");
        }
    }

    public Version() { this(null); }
}
```

### 3.3 `domain.model.aggregates` — Device

```java
package com.sensibo.platform.u202014215.registry.domain.model.aggregates;

import com.sensibo.platform.u202014215.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.u202014215.registry.domain.model.valueobjects.*;
import com.sensibo.platform.u202014215.shared.domain.model.aggregates.AbstractDomainAggregateRoot;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDate;

/**
 * Aggregate root representing a Sensibo smart device registered on the platform.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Getter
public class Device extends AbstractDomainAggregateRoot<Device> {

    @Setter
    private Long id;

    private SensiboUserId userId;

    private DeviceTypes deviceType;

    @Setter
    private String modelName;

    private SerialNumber serialNumber;

    private MacAddress macAddress;

    private Version firmwareVersion;

    @Setter
    private LocalDate installationDate;

    @Setter
    private String roomLocation;

    /** JPA/assembler default constructor. */
    public Device() {
        this.serialNumber = new SerialNumber();
    }

    /**
     * Creates a new Device from a registration command.
     * The serial number is always auto-generated, regardless of the command.
     *
     * @param command the device registration command
     */
    public Device(CreateDeviceCommand command) {
        this.userId = new SensiboUserId(command.userId());
        this.deviceType = DeviceTypes.fromString(command.deviceType());
        this.modelName = command.modelName();
        this.serialNumber = new SerialNumber();
        this.macAddress = new MacAddress(command.macAddress());
        this.firmwareVersion = new Version(command.firmwareVersion());
        this.installationDate = command.installationDate();
        this.roomLocation = command.roomLocation();
    }
}
```

### 3.4 `domain.model.commands`

```java
package com.sensibo.platform.u202014215.registry.domain.model.commands;

import java.time.LocalDate;

/**
 * Command to register a new device on the Sensibo platform.
 *
 * @param userId the platform user identifier
 * @param deviceType the name of one of the existing DeviceTypes
 * @param modelName the commercial model name of the device
 * @param macAddress the device's MAC address (AA:BB:CC:DD:EE:FF)
 * @param firmwareVersion the device's firmware version (X.Y.Z)
 * @param installationDate the installation date, must not be after today
 * @param roomLocation the room or location where the device is installed
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record CreateDeviceCommand(
    Long userId,
    String deviceType,
    String modelName,
    String macAddress,
    String firmwareVersion,
    LocalDate installationDate,
    String roomLocation) {
}
```

### 3.5 `domain.repositories` — puerto de salida

```java
package com.sensibo.platform.u202014215.registry.domain.repositories;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.domain.model.valueobjects.MacAddress;

/**
 * Repository contract (output port) for the Device aggregate.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public interface DeviceRepository {

    Device save(Device device);

    boolean existsByMacAddress(MacAddress macAddress);
}
```

### 3.6 `application.internal.commandservices`

```java
package com.sensibo.platform.u202014215.registry.application.internal.commandservices;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.u202014215.shared.application.result.ApplicationError;
import com.sensibo.platform.u202014215.shared.application.result.Result;

/**
 * Command service contract for Device registration.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public interface CreateDeviceCommandService {
    Result<Device, ApplicationError> handle(CreateDeviceCommand command);
}
```

```java
package com.sensibo.platform.u202014215.registry.application.internal.commandservices;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.u202014215.registry.domain.model.valueobjects.MacAddress;
import com.sensibo.platform.u202014215.registry.domain.repositories.DeviceRepository;
import com.sensibo.platform.u202014215.shared.application.result.ApplicationError;
import com.sensibo.platform.u202014215.shared.application.result.Result;
import org.springframework.stereotype.Service;

/**
 * Default implementation of {@link CreateDeviceCommandService}.
 * Enforces device registration business rules before persisting the aggregate.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Service
public class CreateDeviceCommandServiceImpl implements CreateDeviceCommandService {

    private final DeviceRepository deviceRepository;

    public CreateDeviceCommandServiceImpl(DeviceRepository deviceRepository) {
        this.deviceRepository = deviceRepository;
    }

    @Override
    public Result<Device, ApplicationError> handle(CreateDeviceCommand command) {
        try {
            var macAddress = new MacAddress(command.macAddress());

            if (deviceRepository.existsByMacAddress(macAddress)) {
                return Result.failure(ApplicationError.conflict(
                    "device", "A device with MAC address %s is already registered"
                        .formatted(command.macAddress())));
            }

            var device = new Device(command);
            var savedDevice = deviceRepository.save(device);
            return Result.success(savedDevice);

        } catch (IllegalArgumentException ex) {
            return Result.failure(ApplicationError.validationError("device", ex.getMessage()));
        }
    }
}
```

### 3.7 `infrastructure.persistence.jpa`

**Entidad JPA** (`infrastructure.persistence.jpa.entities`):

```java
package com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.entities;

import com.sensibo.platform.u202014215.shared.infrastructure.persistence.jpa.entities.AuditableAbstractPersistenceEntity;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDate;

/**
 * JPA persistence entity mapping the Device aggregate to the {@code devices} table.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Getter
@Setter
@Entity
@Table(name = "devices", uniqueConstraints = @UniqueConstraint(columnNames = "mac_address"))
public class DeviceEntity extends AuditableAbstractPersistenceEntity {

    @Column(nullable = false)
    private Long userId;

    @Column(nullable = false)
    private Integer deviceType;

    @Column(nullable = false, length = 50)
    private String modelName;

    @Column(nullable = false, unique = true, length = 36)
    private String serialNumber;

    @Column(nullable = false, unique = true)
    private String macAddress;

    @Column(nullable = false)
    private String firmwareVersion;

    @Column(nullable = false)
    private LocalDate installationDate;

    @Column(nullable = false)
    private String roomLocation;
}
```

**Assembler entidad ⇄ dominio** (`infrastructure.persistence.jpa.assemblers`):

```java
package com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.assemblers;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.domain.model.valueobjects.*;
import com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.entities.DeviceEntity;

/**
 * Converts between the {@link Device} domain aggregate and its {@link DeviceEntity}
 * persistence representation.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public class DeviceEntityAssembler {

    public static DeviceEntity toEntityFromDevice(Device device) {
        var entity = new DeviceEntity();
        entity.setId(device.getId());
        entity.setUserId(device.getUserId().value());
        entity.setDeviceType(device.getDeviceType().getId());
        entity.setModelName(device.getModelName());
        entity.setSerialNumber(device.getSerialNumber().value());
        entity.setMacAddress(device.getMacAddress().value());
        entity.setFirmwareVersion(device.getFirmwareVersion().value());
        entity.setInstallationDate(device.getInstallationDate());
        entity.setRoomLocation(device.getRoomLocation());
        return entity;
    }

    public static Device toDeviceFromEntity(DeviceEntity entity) {
        var device = new Device();
        device.setId(entity.getId());
        device.setModelName(entity.getModelName());
        device.setInstallationDate(entity.getInstallationDate());
        device.setRoomLocation(entity.getRoomLocation());
        // userId, deviceType, serialNumber, macAddress, firmwareVersion are read-only
        // value objects on Device; expose them through dedicated reconstruction logic
        // if your Device class adds package-private setters, or via reflection-free
        // reconstruction constructor as needed by your final design.
        return device;
    }
}
```

> 📝 Nota: si tu profesor exige que `Device` quede 100% inmutable para esos campos
> (sin setters), agrega un constructor adicional en `Device` que reciba directamente los
> value objects ya construidos (`SensiboUserId`, `DeviceTypes`, `SerialNumber`,
> `MacAddress`, `Version`) y úsalo desde `toDeviceFromEntity`, en vez de exponer setters.

**Repositorio Spring Data** (`infrastructure.persistence.jpa.repositories`):

```java
package com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.repositories;

import com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.entities.DeviceEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface DeviceJpaRepository extends JpaRepository<DeviceEntity, Long> {
    boolean existsByMacAddress(String macAddress);
}
```

**Adaptador del puerto de dominio** (`infrastructure.persistence.jpa.adapters`):

```java
package com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.adapters;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.domain.model.valueobjects.MacAddress;
import com.sensibo.platform.u202014215.registry.domain.repositories.DeviceRepository;
import com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.assemblers.DeviceEntityAssembler;
import com.sensibo.platform.u202014215.registry.infrastructure.persistence.jpa.repositories.DeviceJpaRepository;
import org.springframework.stereotype.Repository;

/**
 * Adapts {@link DeviceJpaRepository} to the {@link DeviceRepository} domain port.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@Repository
public class DeviceRepositoryAdapter implements DeviceRepository {

    private final DeviceJpaRepository deviceJpaRepository;

    public DeviceRepositoryAdapter(DeviceJpaRepository deviceJpaRepository) {
        this.deviceJpaRepository = deviceJpaRepository;
    }

    @Override
    public Device save(Device device) {
        var entity = DeviceEntityAssembler.toEntityFromDevice(device);
        var savedEntity = deviceJpaRepository.save(entity);
        return DeviceEntityAssembler.toDeviceFromEntity(savedEntity);
    }

    @Override
    public boolean existsByMacAddress(MacAddress macAddress) {
        return deviceJpaRepository.existsByMacAddress(macAddress.value());
    }
}
```

### 3.8 `interfaces.rest` — Resources y Assemblers

**Resource de entrada** (`interfaces.rest.resources`):

```java
package com.sensibo.platform.u202014215.registry.interfaces.rest.resources;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.PastOrPresent;
import jakarta.validation.constraints.Size;

import java.time.LocalDate;

/**
 * Request resource for registering a new device.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record CreateDeviceResource(
    @NotNull Long userId,
    @NotBlank String deviceType,
    @NotBlank @Size(max = 50) String modelName,
    @NotBlank String macAddress,
    @NotBlank String firmwareVersion,
    @NotNull @PastOrPresent LocalDate installationDate,
    @NotBlank String roomLocation) {
}
```

**Resource de salida**:

```java
package com.sensibo.platform.u202014215.registry.interfaces.rest.resources;

import java.time.LocalDate;

/**
 * Response resource representing a registered device.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public record DeviceResource(
    Long id,
    String serialNumber,
    String deviceType,
    String modelName,
    String macAddress,
    String firmwareVersion,
    LocalDate installationDate,
    String roomLocation) {
}
```

**Assemblers** (`interfaces.rest.transform`):

```java
package com.sensibo.platform.u202014215.registry.interfaces.rest.transform;

import com.sensibo.platform.u202014215.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.u202014215.registry.interfaces.rest.resources.CreateDeviceResource;

/**
 * Maps a {@link CreateDeviceResource} into a {@link CreateDeviceCommand}.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public class CreateDeviceCommandFromResourceAssembler {
    public static CreateDeviceCommand toCommandFromResource(CreateDeviceResource resource) {
        return new CreateDeviceCommand(
            resource.userId(),
            resource.deviceType(),
            resource.modelName(),
            resource.macAddress(),
            resource.firmwareVersion(),
            resource.installationDate(),
            resource.roomLocation());
    }
}
```

```java
package com.sensibo.platform.u202014215.registry.interfaces.rest.transform;

import com.sensibo.platform.u202014215.registry.domain.model.aggregates.Device;
import com.sensibo.platform.u202014215.registry.interfaces.rest.resources.DeviceResource;

/**
 * Maps a {@link Device} aggregate into a {@link DeviceResource}.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
public class DeviceResourceFromEntityAssembler {
    public static DeviceResource toResourceFromEntity(Device device) {
        return new DeviceResource(
            device.getId(),
            device.getSerialNumber().value(),
            device.getDeviceType().name(),
            device.getModelName(),
            device.getMacAddress().value(),
            device.getFirmwareVersion().value(),
            device.getInstallationDate(),
            device.getRoomLocation());
    }
}
```

---

## 4. Controlador del endpoint y documentación OpenAPI (punto 12 del enunciado)

El profesor pidió documentar el endpoint con OpenAPI/Swagger, como en la captura
(bloque `@Operation` + `@ApiResponses`, líneas 55-69, sobre un `@PostMapping`). El mismo
patrón se aplica aquí al controlador de `Device`:

```java
package com.sensibo.platform.u202014215.registry.interfaces.rest;

import com.sensibo.platform.u202014215.registry.application.internal.commandservices.CreateDeviceCommandService;
import com.sensibo.platform.u202014215.registry.interfaces.rest.resources.CreateDeviceResource;
import com.sensibo.platform.u202014215.registry.interfaces.rest.resources.DeviceResource;
import com.sensibo.platform.u202014215.registry.interfaces.rest.transform.CreateDeviceCommandFromResourceAssembler;
import com.sensibo.platform.u202014215.registry.interfaces.rest.transform.DeviceResourceFromEntityAssembler;
import com.sensibo.platform.u202014215.shared.interfaces.rest.transform.ResponseEntityAssembler;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

/**
 * REST controller exposing device registration operations for the Sensibo platform.
 *
 * @author Rafael Agustin Pacheco Lavado
 */
@RestController
@RequestMapping(value = "/api/v1/devices", produces = "application/json")
@Tag(name = "Devices", description = "Device registration management endpoints")
public class DevicesController {

    private final CreateDeviceCommandService createDeviceCommandService;

    public DevicesController(CreateDeviceCommandService createDeviceCommandService) {
        this.createDeviceCommandService = createDeviceCommandService;
    }

    /**
     * Registers a new Sensibo device on the platform.
     *
     * @param resource the device registration data
     * @return the created device, or an error response
     */
    @PostMapping
    @Operation(
        summary = "Register a new device",
        description = "Registers a new Sensibo device record with complete device information."
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "201",
            description = "Device registered successfully",
            content = @Content(schema = @Schema(implementation = DeviceResource.class))
        ),
        @ApiResponse(responseCode = "400", description = "Invalid input data"),
        @ApiResponse(responseCode = "409", description = "Conflict - Device already registered with this MAC address"),
        @ApiResponse(responseCode = "500", description = "Unexpected server error")
    })
    public ResponseEntity<?> createDevice(@Valid @RequestBody CreateDeviceResource resource) {
        var command = CreateDeviceCommandFromResourceAssembler.toCommandFromResource(resource);
        var result = createDeviceCommandService.handle(command);
        return ResponseEntityAssembler.toResponseEntityFromResult(
            result,
            DeviceResourceFromEntityAssembler::toResourceFromEntity,
            HttpStatus.CREATED);
    }
}
```

> Nota: en el enunciado no se pide soportar `401`/`403` (no hay Security en el alcance del
> proyecto — ver "NO forma parte del alcance"), por eso esos códigos no se incluyen, a
> diferencia del ejemplo de la captura que sí los tenía (ese controlador de ejemplo sí
> usaba JWT).

Una vez levantado el proyecto, Swagger UI queda disponible en:
`http://localhost:8096/swagger-ui/index.html`

---

## 5. README.md

**Crear** en la raíz del proyecto un `README.md` en inglés:

```markdown
# Sensibo Climate Control Platform — Device Registry API

RESTful API built with Spring Boot and Spring Data JPA that supports the registration
of Sensibo smart devices (AC controllers, room sensors, air quality monitors and
door/window sensors) within the Sensibo Climate Control ecosystem.

## Endpoint

- `POST /api/v1/devices` — registers a new device.

## Prerequisites

- Java 26
- Maven
- PostgreSQL (schema `sensibo`)

## Tech stack

- Java 26
- Spring Boot 3.5.6
- Spring Data JPA
- PostgreSQL
- springdoc-openapi (Swagger UI)

## Author

Rafael Agustin Pacheco Lavado — u202014215
```

---

## 6. Empaquetado final

1. **Verificar** que el proyecto compila y ejecuta sin errores (`mvn clean package`).
2. **Comprimir** la carpeta del proyecto como `.zip`.
3. **Renombrar** el archivo a `pc211990u202014215.zip` (NRC `11990` + código `u202014215`).
4. **Subir** el archivo en la Actividad "PC2 (Práctica calificada 2)".

---

## 7. Checklist rápido contra la rúbrica

- [ ] El proyecto inicia sin errores en el puerto `8096`.
- [ ] `POST /api/v1/devices` responde `201` con el `DeviceResource` y el `id` generado.
- [ ] `400` ante datos inválidos (mac/version con formato incorrecto, campos vacíos, fecha futura).
- [ ] `409` si ya existe un device con la misma `macAddress`.
- [ ] `serialNumber` e `id` nunca se reciben en el request: siempre autogenerados.
- [ ] `deviceType` se valida contra el `Name` del enum `DeviceTypes`.
- [ ] Persistencia en PostgreSQL, esquema `sensibo`, tabla `devices` (snake_case, plural).
- [ ] `Device` es un aggregate root auditable (`createdAt`/`updatedAt` vía `shared`).
- [ ] Mensajes de error localizados según `Accept-Language` (`en`, `en-US`, `es`, `es-PE`).
- [ ] Paquetes con raíz `com.sensibo.platform.u202014215`, bounded context `registry`.
- [ ] JavaDoc con `@author Rafael Agustin Pacheco Lavado`.
- [ ] Swagger UI documentado en inglés (`@Operation` + `@ApiResponses`).
- [ ] `README.md` en inglés con descripción y autor.
- [ ] Solución empaquetada como `pc211990u202014215.zip`.
