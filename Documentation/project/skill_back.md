# 🔧 Skill — Backend WIN24 (Spring Boot)

> **Proyecto:** Bolirana WIN24 — Plataforma de apuestas deportivas  
> **Stack:** Spring Boot 3.3.0 · Java 17 · PostgreSQL · Maven  
> **Paquete raíz:** `com.bolirana.backend`

---

## 1. Arquitectura

El backend sigue una **arquitectura en capas estricta** (Layered Architecture), donde cada capa tiene una responsabilidad única y la dependencia siempre fluye hacia abajo:

```
HTTP Request
     │
     ▼
┌─────────────┐
│ Controller  │  Recibe HTTP, delega, traduce a ResponseEntity
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Service   │  Lógica de negocio, orquesta repositorios
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Repository  │  Acceso a datos (Spring Data JPA)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Domain    │  Entidades JPA → tablas PostgreSQL
└─────────────┘
```

### Paquetes y responsabilidades

| Paquete | Responsabilidad | Clases |
|---|---|---|
| `controller/` | REST: recibe HTTP, delega a Service | 6 controllers |
| `service/` | Lógica de negocio, orquesta repositorios | 6 services |
| `repository/` | Acceso a datos (Spring Data JPA) | 8 repositories |
| `domain/` | Entidades JPA → tablas PostgreSQL | 8 entidades |
| `config/` | Configuración transversal (CORS) | `WebConfig.java` |

### Modelo de relaciones

```
Usuario ←(N:1)── Apuesta ──(N:1)→ OpcionApuesta ──(N:1)→ Mercado ──(N:1)→ Evento
Usuario ←(N:1)── MovimientoSaldo
OpcionApuesta ←(N:1)── HistorialCuota
Mercado ←(1:1)── ConfiguracionRiesgo
```

---

## 2. Endpoints (17 rutas, todas bajo `/api/...`)

| Controller | Verbo | Ruta | Método Java |
|---|---|---|---|
| Apuesta | GET | `/api/apuestas` | `listar()` |
| Apuesta | GET | `/api/apuestas/{id}` | `buscarPorId()` |
| Apuesta | GET | `/api/apuestas/usuario/{id}` | `listarPorApostador()` |
| Apuesta | POST | `/api/apuestas` | `crear()` → 201 |
| ConfiguracionRiesgo | GET | `/api/riesgo/mercado/{id}` | `buscarPorMercado()` |
| ConfiguracionRiesgo | POST | `/api/riesgo` | `crearOActualizar()` → 200 upsert |
| Evento | GET | `/api/eventos` | `listar()` |
| Evento | GET | `/api/eventos/{id}` | `buscarPorId()` |
| Evento | POST | `/api/eventos` | `crear()` → 201 |
| Mercado | GET | `/api/mercados` | `listar()` |
| Mercado | GET | `/api/mercados/{id}` | `buscarPorId()` |
| Mercado | POST | `/api/mercados` | `crear()` → 201 |
| MovimientoSaldo | GET | `/api/movimientos` | `listar()` |
| MovimientoSaldo | GET | `/api/movimientos/usuario/{id}` | `listarPorUsuario()` |
| MovimientoSaldo | POST | `/api/movimientos` | `crear()` → 201 |
| Usuario | GET | `/api/usuarios` | `listar()` |
| Usuario | GET | `/api/usuarios/{id}` | `buscarPorId()` |
| Usuario | POST | `/api/usuarios` | `crear()` → 201 |

---

## 3. Patrones de diseño aplicados

### ✅ Repository Pattern
Acceso a datos mediante `JpaRepository<Entidad, Long>` con *derived query methods*:
```java
// Spring Data genera la query automáticamente por el nombre del método
List<Apuesta> findByApostadorId(Long apostadorId);
Optional<Usuario> findByCorreo(String correo);
```

### ✅ Dependency Injection por constructor (Lombok)
Todos los `@Service` y `@RestController` usan `@RequiredArgsConstructor` — sin `@Autowired` en campos:
```java
@Service
@RequiredArgsConstructor
public class ApuestaService {
    private final ApuestaRepository apuestaRepository;
    private final OpcionApuestaRepository opcionApuestaRepository;
}
```

### ✅ Optional<T> para entidades no encontradas
Patrón uniforme en todos los `buscarPorId`:
```java
@GetMapping("/{id}")
public ResponseEntity<Evento> buscarPorId(@PathVariable Long id) {
    return eventoService.buscarPorId(id)
        .map(ResponseEntity::ok)
        .orElseGet(() -> ResponseEntity.notFound().build());
}
```

### ✅ Snapshot / Frozen-value (Apuesta)
Al crear una apuesta se congela la cuota vigente — valor histórico inmutable:
```java
public Apuesta crear(Apuesta apuesta) {
    OpcionApuesta opcion = opcionApuestaRepository.findById(...)
        .orElseThrow(() -> new IllegalArgumentException("Opcion no encontrada"));
    apuesta.setCuotaCongelada(opcion.getCuotaActual()); // snapshot
    apuesta.setEstado("PENDIENTE");
    return apuestaRepository.save(apuesta);
}
```

### ✅ Upsert (ConfiguracionRiesgo)
```java
public ConfiguracionRiesgo crearOActualizar(ConfiguracionRiesgo config) {
    return configuracionRiesgoRepository
        .findByMercadoId(config.getMercado().getId())
        .map(existing -> {
            existing.setLimiteAlerta(config.getLimiteAlerta());
            return configuracionRiesgoRepository.save(existing);
        })
        .orElse(configuracionRiesgoRepository.save(config));
}
```

### ✅ CORS centralizado
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*");
    }
}
```

---

## 4. Convenciones de código

### Naming
| Elemento | Convención | Ejemplo |
|---|---|---|
| Clases | PascalCase | `ApuestaService`, `EventoController` |
| Métodos de servicio | camelCase en español | `listar()`, `buscarPorId()`, `crearOActualizar()` |
| Métodos de repositorio | Spring Data (`findBy<Campo>`) | `findByApostadorId()`, `findByCorreo()` |
| Campos Java | camelCase | `cuotaCongelada`, `fechaCreacion` |
| Columnas DB | snake_case español | `cuota_congelada`, `nombre_completo` |
| Constantes | UPPER_SNAKE_CASE | — |

### Lombok en entidades
```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Evento {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    private String deporte;
    private String estado;
    @CreationTimestamp
    private LocalDateTime fechaCreacion;
}
```

### Javadoc obligatorio (Checkstyle)
Todo método público debe documentarse con `@param`, `@return` y `@throws` donde aplique:
```java
/**
 * Registra una nueva apuesta en el sistema.
 * Congela la cuota actual de la opción seleccionada (snapshot inmutable)
 * e inicializa el estado en PENDIENTE.
 *
 * @param apuesta objeto con los datos de la apuesta a registrar
 * @return la apuesta persistida con cuotaCongelada y estado asignados
 * @throws IllegalArgumentException si la opción de apuesta no existe
 */
public Apuesta crear(Apuesta apuesta) { ... }
```

### Código de respuesta HTTP
| Operación | Código |
|---|---|
| GET exitoso | 200 OK |
| GET no encontrado | 404 Not Found |
| POST creación | 201 Created |
| POST upsert | 200 OK |

---

## 5. Herramientas de calidad

### Checkstyle (maven-checkstyle-plugin 3.3.1)
Reglas activas:

| Regla | Descripción |
|---|---|
| `MissingJavadocMethod` | Javadoc obligatorio en métodos `public` |
| `TypeName` | Clases en PascalCase |
| `MethodName` / `ParameterName` / `LocalVariableName` | camelCase |
| `ConstantName` | UPPER_SNAKE_CASE |
| `AvoidStarImport` | Sin `import java.util.*` |
| `LineLength` (max=120) | Máximo 120 caracteres por línea |
| `LeftCurly` / `RightCurly` | Llaves en la misma línea |
| `ModifierOrder` | Orden: `public static final` |
| `NewlineAtEndOfFile` | Archivos terminan con nueva línea |

```bash
# Verificar sin bloquear build:
./mvnw checkstyle:check

# Resultado actual: ✅ 0 violaciones — BUILD SUCCESS
```

### JaCoCo (cobertura de tests)
```bash
./mvnw test jacoco:report
# Reporte: target/site/jacoco/index.html
```

---

## 6. MUST have ✅ / SHOULD have 🔶

### MUST have — Reglas obligatorias

- [x] Arquitectura en capas estricta: Controller → Service → Repository (ningún Controller accede a Repository directamente)
- [x] Inyección de dependencias por constructor (`@RequiredArgsConstructor`)
- [x] Métodos públicos documentados con Javadoc (`@param`, `@return`, `@throws`)
- [x] Nombres en español para dominio de negocio (`listar`, `crear`, `buscarPorId`)
- [x] `Optional<T>` para búsquedas que pueden retornar vacío
- [x] Códigos HTTP correctos: 201 en POST de creación, 404 en no encontrado
- [x] Checkstyle con 0 violaciones (`./mvnw checkstyle:check`)
- [x] Tests unitarios con JUnit 5 + Mockito (sin acceso a BD)
- [x] CORS configurado en capa de configuración centralizada (`WebConfig`)
- [x] Sin `@Autowired` en campos — solo inyección por constructor

### SHOULD have — Buenas prácticas recomendadas

- [ ] DTOs separados de entidades JPA (evitar exposición directa del modelo)
- [ ] `@ControllerAdvice` para manejo global de excepciones (actualmente `IllegalArgumentException` → 500)
- [ ] `@Valid` + anotaciones de validación (`@NotBlank`, `@Email`) en request bodies
- [ ] Enums Java para `estado` (`PENDIENTE`, `GANADA`, `PERDIDA`) en lugar de `String` libre
- [ ] Flyway o Liquibase para versionado de esquema (actualmente `ddl-auto: update`)
- [ ] Variables de entorno para credenciales (no hardcodeadas en `application.yml`)
- [ ] Exponer endpoints huérfanos: `GET /api/mercados/evento/{id}` y `GET /api/usuarios/correo/{correo}`

---

## 7. Cómo ejecutar

```bash
# Requisitos: Java 17, Maven, PostgreSQL corriendo

cd project-backend

# Levantar backend:
./mvnw spring-boot:run
# → Tomcat en http://localhost:8080

# Verificar calidad:
./mvnw checkstyle:check        # 0 violaciones
./mvnw test jacoco:report      # Tests + cobertura

# Probar endpoint:
curl http://localhost:8080/api/eventos
```
