# Flujo de Creación de una Visita Veterinaria

## Explicación completa desde el Controller hasta la Persistencia

Este documento describe el flujo completo que sigue la aplicación Spring PetClinic cuando un usuario crea una nueva visita veterinaria para una mascota.

---

## 1. Capa de Presentación (Vista JSP)

**Archivo:** `src/main/webapp/WEB-INF/jsp/pets/createOrUpdateVisitForm.jsp`

El formulario de creación de visitas utiliza las etiquetas de Spring MVC para el binding de datos:

```jsp
<form:form modelAttribute="visit" class="form-horizontal">
    <petclinic:inputField label="Date" name="date"/>
    <petclinic:inputField label="Description" name="description"/>
    <input type="hidden" name="petId" value="${visit.pet.id}"/>
    <button class="btn btn-default" type="submit">Add Visit</button>
</form:form>
```

- El formulario está vinculado al atributo de modelo `visit`.
- Contiene dos campos: **fecha** y **descripción**.
- El `petId` se envía como campo oculto.
- Además, muestra una tabla con las visitas anteriores de la mascota.

---

## 2. Capa del Controlador

**Archivo:** `src/main/java/org/springframework/samples/petclinic/web/VisitController.java`

### 2.1 Carga previa del modelo (`@ModelAttribute`)

Antes de que se ejecute cualquier handler (GET o POST), Spring invoca el método anotado con `@ModelAttribute`:

```java
@ModelAttribute("visit")
public Visit loadPetWithVisit(@PathVariable("petId") int petId) {
    Pet pet = this.clinicService.findPetById(petId);
    Visit visit = new Visit();
    pet.addVisit(visit);
    return visit;
}
```

Este método:
1. Busca la mascota (Pet) por su ID a través del `ClinicService`.
2. Crea una nueva instancia de `Visit`.
3. Asocia la visita con la mascota mediante `pet.addVisit(visit)`, lo que establece la relación bidireccional.
4. Retorna el objeto `Visit`, que queda disponible en el modelo bajo el nombre `"visit"`.

### 2.2 Mostrar el formulario (GET)

```
GET /owners/{ownerId}/pets/{petId}/visits/new
```

```java
@RequestMapping(value = "/owners/*/pets/{petId}/visits/new", method = RequestMethod.GET)
public String initNewVisitForm() {
    return "pets/createOrUpdateVisitForm";
}
```

Simplemente retorna la vista JSP del formulario. El modelo ya contiene el objeto `Visit` gracias al `@ModelAttribute`.

### 2.3 Procesar el formulario (POST)

```
POST /owners/{ownerId}/pets/{petId}/visits/new
```

```java
@RequestMapping(value = "/owners/{ownerId}/pets/{petId}/visits/new", method = RequestMethod.POST)
public String processNewVisitForm(@Valid Visit visit, BindingResult result) {
    if (result.hasErrors()) {
        return "pets/createOrUpdateVisitForm";
    } else {
        this.clinicService.saveVisit(visit);
        return "redirect:/owners/{ownerId}";
    }
}
```

Este método:
1. Recibe el objeto `Visit` ya vinculado (data binding) y validado (`@Valid`).
2. Si hay errores de validación, retorna al formulario mostrando los errores.
3. Si la validación es exitosa, llama a `clinicService.saveVisit(visit)`.
4. Redirige al detalle del propietario (`/owners/{ownerId}`).

### 2.4 Seguridad del binding (`@InitBinder`)

```java
@InitBinder
public void setAllowedFields(WebDataBinder dataBinder) {
    dataBinder.setDisallowedFields("id");
}
```

Impide que el campo `id` pueda ser manipulado desde el formulario, evitando inyección de IDs.

---

## 3. Capa del Modelo / Entidad

### 3.1 Visit

**Archivo:** `src/main/java/org/springframework/samples/petclinic/model/Visit.java`

```java
@Entity
@Table(name = "visits")
public class Visit extends BaseEntity {

    @Column(name = "visit_date")
    @DateTimeFormat(pattern = "yyyy/MM/dd")
    private LocalDate date;

    @NotEmpty
    @Column(name = "description")
    private String description;

    @ManyToOne
    @JoinColumn(name = "pet_id")
    private Pet pet;

    public Visit() {
        this.date = LocalDate.now(); // Por defecto, la fecha actual
    }
}
```

Características clave:
- **Herencia:** Extiende `BaseEntity`, que proporciona el campo `id` con `@GeneratedValue(strategy = GenerationType.IDENTITY)`.
- **Validación:** `@NotEmpty` en `description` garantiza que no se acepten descripciones vacías.
- **Formato de fecha:** `@DateTimeFormat(pattern = "yyyy/MM/dd")` para la conversión automática desde String.
- **Relación:** `@ManyToOne` con `Pet` a través de la columna `pet_id`.
- **Valor por defecto:** La fecha se inicializa con `LocalDate.now()`.

### 3.2 BaseEntity

**Archivo:** `src/main/java/org/springframework/samples/petclinic/model/BaseEntity.java`

```java
@MappedSuperclass
public class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    protected Integer id;

    public boolean isNew() {
        return this.id == null;
    }
}
```

El método `isNew()` es clave para determinar si se debe hacer un **INSERT** (id es null) o un **UPDATE** (id existe).

### 3.3 Pet (relación bidireccional)

**Archivo:** `src/main/java/org/springframework/samples/petclinic/model/Pet.java`

```java
@Entity
@Table(name = "pets")
public class Pet extends NamedEntity {

    @OneToMany(cascade = CascadeType.ALL, mappedBy = "pet", fetch = FetchType.EAGER)
    private Set<Visit> visits;

    public void addVisit(Visit visit) {
        getVisitsInternal().add(visit);
        visit.setPet(this);
    }
}
```

- La relación es **bidireccional**: Pet conoce sus Visits y cada Visit conoce su Pet.
- `CascadeType.ALL` propaga todas las operaciones (persist, merge, remove) desde Pet hacia Visit.
- `addVisit()` establece ambos lados de la relación.

---

## 4. Capa de Servicio

### 4.1 Interfaz ClinicService

**Archivo:** `src/main/java/org/springframework/samples/petclinic/service/ClinicService.java`

```java
public interface ClinicService {
    void saveVisit(Visit visit);
    Collection<Visit> findVisitsByPetId(int petId);
    Pet findPetById(int id);
    // ... otros métodos
}
```

### 4.2 Implementación ClinicServiceImpl

**Archivo:** `src/main/java/org/springframework/samples/petclinic/service/ClinicServiceImpl.java`

```java
@Service
public class ClinicServiceImpl implements ClinicService {

    private VisitRepository visitRepository;

    @Autowired
    public ClinicServiceImpl(PetRepository petRepository, VetRepository vetRepository,
                             OwnerRepository ownerRepository, VisitRepository visitRepository) {
        this.visitRepository = visitRepository;
        // ... otras inyecciones
    }

    @Override
    @Transactional
    public void saveVisit(Visit visit) {
        visitRepository.save(visit);
    }

    @Override
    @Transactional(readOnly = true)
    public Pet findPetById(int id) {
        return petRepository.findById(id);
    }
}
```

Características clave:
- **`@Service`**: Registrado como bean de Spring.
- **`@Transactional`**: `saveVisit()` se ejecuta dentro de una transacción de escritura.
- **`@Transactional(readOnly = true)`**: Las operaciones de lectura se marcan como solo lectura para optimización.
- **Inyección por constructor**: Todos los repositorios se inyectan vía constructor.

---

## 5. Capa de Repositorio (Persistencia)

La aplicación ofrece **tres implementaciones** del repositorio, seleccionables mediante perfiles de Spring.

### 5.1 Interfaz VisitRepository

**Archivo:** `src/main/java/org/springframework/samples/petclinic/repository/VisitRepository.java`

```java
public interface VisitRepository {
    void save(Visit visit);
    List<Visit> findByPetId(Integer petId);
}
```

### 5.2 Implementación JPA

**Archivo:** `src/main/java/org/springframework/samples/petclinic/repository/jpa/JpaVisitRepositoryImpl.java`  
**Perfil activo:** `jpa` (perfil por defecto de la aplicación)

```java
@Repository
public class JpaVisitRepositoryImpl implements VisitRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public void save(Visit visit) {
        if (visit.getId() == null) {
            this.em.persist(visit);   // INSERT — entidad nueva
        } else {
            this.em.merge(visit);     // UPDATE — entidad existente
        }
    }
}
```

- Usa el `EntityManager` de JPA directamente.
- Distingue entre **persist** (INSERT) y **merge** (UPDATE) según `visit.getId()`.
- Hibernate genera el SQL y gestiona la identidad.

### 5.3 Implementación JDBC

**Archivo:** `src/main/java/org/springframework/samples/petclinic/repository/jdbc/JdbcVisitRepositoryImpl.java`  
**Perfil activo:** `jdbc`

```java
@Repository
public class JdbcVisitRepositoryImpl implements VisitRepository {

    private NamedParameterJdbcTemplate jdbcTemplate;
    private SimpleJdbcInsert insertVisit;

    @Autowired
    public JdbcVisitRepositoryImpl(DataSource dataSource) {
        this.jdbcTemplate = new NamedParameterJdbcTemplate(dataSource);
        this.insertVisit = new SimpleJdbcInsert(dataSource)
            .withTableName("visits")
            .usingGeneratedKeyColumns("id");
    }

    @Override
    public void save(Visit visit) {
        if (visit.isNew()) {
            Number newKey = this.insertVisit.executeAndReturnKey(
                createVisitParameterSource(visit));
            visit.setId(newKey.intValue());
        } else {
            throw new UnsupportedOperationException("Visit update not supported");
        }
    }

    private MapSqlParameterSource createVisitParameterSource(Visit visit) {
        return new MapSqlParameterSource()
            .addValue("id", visit.getId())
            .addValue("visit_date", visit.getDate())
            .addValue("description", visit.getDescription())
            .addValue("pet_id", visit.getPet().getId());
    }
}
```

- Usa `SimpleJdbcInsert` para generar el INSERT con auto-generación de clave.
- **Solo soporta creación**, no actualización (lanza `UnsupportedOperationException`).
- Los resultados se mapean con `JdbcVisitRowMapper`.

### 5.4 Implementación Spring Data JPA

**Archivo:** `src/main/java/org/springframework/samples/petclinic/repository/springdatajpa/SpringDataVisitRepository.java`  
**Perfil activo:** `spring-data-jpa`

```java
public interface SpringDataVisitRepository extends VisitRepository, Repository<Visit, Integer> {
}
```

- Es la implementación más minimalista: solo declara una interfaz.
- Spring Data JPA genera automáticamente las implementaciones de `save()` y `findByPetId()`.
- Internamente usa Hibernate como proveedor JPA.

---

## 6. Esquema de Base de Datos

### Tabla `visits`

**Archivo (HSQLDB/H2):** `src/main/resources/db/h2/schema.sql`

```sql
CREATE TABLE visits (
    id          INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    pet_id      INTEGER NOT NULL,
    visit_date  DATE,
    description VARCHAR(255)
);
ALTER TABLE visits ADD CONSTRAINT fk_visits_pets FOREIGN KEY (pet_id) REFERENCES pets (id);
CREATE INDEX visits_pet_id ON visits (pet_id);
```

**Archivo (MySQL):** `src/main/resources/db/mysql/schema.sql`

```sql
CREATE TABLE IF NOT EXISTS visits (
    id INT(4) UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    pet_id INT(4) UNSIGNED NOT NULL,
    visit_date DATE,
    description VARCHAR(255),
    FOREIGN KEY (pet_id) REFERENCES pets(id)
) engine=InnoDB;
```

**Archivo (PostgreSQL):** `src/main/resources/db/postgresql/schema.sql`

```sql
CREATE TABLE IF NOT EXISTS visits (
    id SERIAL,
    pet_id INT NOT NULL,
    visit_date DATE,
    description VARCHAR(255),
    FOREIGN KEY (pet_id) REFERENCES pets(id),
    CONSTRAINT pk_visits PRIMARY KEY (id)
);
```

Todas las variantes comparten:
- **`id`**: Clave primaria auto-generada.
- **`pet_id`**: Clave foránea a la tabla `pets`.
- **`visit_date`**: Fecha de la visita.
- **`description`**: Descripción de la visita (max 255 caracteres).

---

## 7. Configuración de Spring

### 7.1 Perfil por defecto

**Archivo:** `src/main/java/org/springframework/samples/petclinic/PetclinicInitializer.java`

```java
private static final String SPRING_PROFILE = "jpa";
```

El perfil por defecto es **`jpa`**, lo que significa que se usa `JpaVisitRepositoryImpl` con EntityManager.

### 7.2 Configuración de negocio

**Archivo:** `src/main/resources/spring/business-config.xml`

Define:
- **Perfil `jpa` y `spring-data-jpa`**: Configura `EntityManagerFactory` con Hibernate y `JpaTransactionManager`.
- **Perfil `jdbc`**: Configura `DataSourceTransactionManager`.
- **Perfil `spring-data-jpa`**: Activa el escaneo de repositorios de Spring Data.

### 7.3 Configuración del DataSource

**Archivo:** `src/main/resources/spring/datasource-config.xml`

Configura la fuente de datos (HSQLDB por defecto para desarrollo) e inicializa las tablas con los scripts SQL (`schema.sql` y `data.sql`).

---

## 8. Diagrama del Flujo Completo

```
┌──────────────────────────────────────────────────────────────────────┐
│                         NAVEGADOR (Usuario)                          │
│                                                                      │
│  1. GET /owners/{ownerId}/pets/{petId}/visits/new                    │
│  6. Llena el formulario y hace POST                                  │
│ 17. Ve la página del propietario con la nueva visita                 │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      CONTROLLER (VisitController)                    │
│                                                                      │
│  2. @ModelAttribute: clinicService.findPetById(petId)                │
│  3. Crea Visit, asocia con Pet vía pet.addVisit(visit)               │
│  4. GET  → Retorna vista "pets/createOrUpdateVisitForm"              │
│  7. POST → Spring hace data binding del formulario al objeto Visit   │
│  8. @Valid → Ejecuta validación Bean Validation (@NotEmpty)          │
│  9. Si errores → Retorna al formulario                               │
│ 10. Si OK → clinicService.saveVisit(visit)                           │
│ 16. Redirect → /owners/{ownerId}                                     │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     SERVICE (ClinicServiceImpl)                       │
│                                                                      │
│ 11. @Transactional → Abre transacción                                │
│ 12. visitRepository.save(visit)                                      │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  REPOSITORY (según perfil activo)                     │
│                                                                      │
│  Perfil "jpa" (por defecto):                                         │
│  13a. JpaVisitRepositoryImpl → EntityManager.persist(visit)          │
│                                                                      │
│  Perfil "jdbc":                                                      │
│  13b. JdbcVisitRepositoryImpl → SimpleJdbcInsert.executeAndReturnKey │
│                                                                      │
│  Perfil "spring-data-jpa":                                           │
│  13c. SpringDataVisitRepository → Auto-implementado por Spring Data  │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        BASE DE DATOS                                 │
│                                                                      │
│ 14. INSERT INTO visits (pet_id, visit_date, description)             │
│     VALUES (?, ?, ?)                                                 │
│ 15. Genera ID automáticamente y confirma transacción                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 9. Resumen de Archivos Involucrados

| Capa | Archivo | Responsabilidad |
|------|---------|-----------------|
| Vista | `WEB-INF/jsp/pets/createOrUpdateVisitForm.jsp` | Formulario de creación de visita |
| Controller | `web/VisitController.java` | Manejo de peticiones HTTP GET/POST |
| Modelo | `model/Visit.java` | Entidad JPA, validación, mapeo a tabla `visits` |
| Modelo | `model/BaseEntity.java` | ID auto-generado, método `isNew()` |
| Modelo | `model/Pet.java` | Relación bidireccional con Visit |
| Servicio | `service/ClinicService.java` | Interfaz del servicio |
| Servicio | `service/ClinicServiceImpl.java` | Implementación con gestión transaccional |
| Repositorio | `repository/VisitRepository.java` | Interfaz del repositorio |
| Repositorio | `repository/jpa/JpaVisitRepositoryImpl.java` | Persistencia vía JPA/EntityManager |
| Repositorio | `repository/jdbc/JdbcVisitRepositoryImpl.java` | Persistencia vía JDBC directo |
| Repositorio | `repository/springdatajpa/SpringDataVisitRepository.java` | Persistencia vía Spring Data JPA |
| Configuración | `spring/business-config.xml` | Perfiles, transacciones, EntityManager |
| Configuración | `spring/datasource-config.xml` | DataSource e inicialización de BD |
| Configuración | `PetclinicInitializer.java` | Perfil por defecto (`jpa`) |
| Base de datos | `db/h2/schema.sql` | Esquema DDL de la tabla `visits` |
