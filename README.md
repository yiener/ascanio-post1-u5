markdown

```
#  Sistema de Reservas de Laboratorios Universitarios

> **Proyecto:** Post-contenido — Unidad 5  
> **Autor:** Yeiner Ascanio  
> **Código estudiantil:** 02230132014  
> **Stack:** Spring Boot 3.2.3 · Java 17 · Spring Data JPA · H2 · Thymeleaf · JUnit 5 · Mockito  
> **Repositorio:** `Ascanio-post1-u5` (Parte 1 + Parte 2)

---

##  1. ¿Qué es este proyecto?

Es una aplicación empresarial de **gestión de reservas de laboratorios universitarios**, construida sobre **Spring Boot 3.2.3** y diseñada siguiendo una **arquitectura por capas limpia y desacoplada**.

El sistema expone dos superficies de interacción distintas que **comparten exactamente la misma lógica de negocio**:

| Superficie | Tecnología | Cliente objetivo |
|---|---|---|
| **API REST** | Spring Web + JSON | Integraciones externas, Postman, móviles |
| **Portal Web MVC** | Thymeleaf + HTML/CSS | Usuarios humanos desde el navegador |

Ambas superficies se apoyan en un único `ReservaService` transaccional que concentra las reglas del dominio: control de solapamientos, validación de horarios institucionales, restricciones de duración y cancelación controlada.

---

##  2. Arquitectura del proyecto

La estructura de paquetes respeta estrictamente la separación de responsabilidades:

```

svgsvg

com.universidad.reservaslabs
│
├── ReservasLabsApplication.java ← Punto de entrada Spring Boot
│
├── config/
│ └── DataInitializer.java ← Semilla de datos (CommandLineRunner)
│
├── model/ ← Entidades JPA (Dominio)
│ ├── EstadoReserva.java ← Enum: PENDIENTE | CONFIRMADA | CANCELADA
│ ├── Laboratorio.java ← Entidad Laboratorio
│ └── Reserva.java ← Entidad Reserva (@ManyToOne → Laboratorio)
│
├── repository/ ← Acceso a datos (Spring Data JPA)
│ ├── LaboratorioRepository.java
│ └── ReservaRepository.java ← JPQL de detección de solapamientos
│
├── service/ ← Lógica de negocio (transaccional)
│ └── ReservaService.java ← Reglas: horario, duración, solapamiento, cancelación
│
├── exception/ ← Excepciones de dominio + handler REST
│ ├── RecursoNoEncontradoException.java
│ ├── ReservaConflictException.java
│ ├── ErrorResponse.java
│ └── GlobalRestExceptionHandler.java ← @RestControllerAdvice
│
├── controller/ ← Controladores REST
│ ├── LaboratorioController.java
│ └── ReservaController.java
│
└── web/ ← Controladores MVC + handler web
├── ReservaWebController.java
└── ReservaWebExceptionHandler.java ← @ControllerAdvice (redirección)

text

````
---

##  3. Configuración del entorno

**Archivo:** `src/main/resources/application.properties`

| Parámetro | Valor |
|---|---|
| Puerto | `8080` |
| Base de datos | H2 en memoria (`jdbc:h2:mem:reservas_labs_db`) |
| Consola H2 | `/h2-console` · usuario `sa` · sin contraseña |
| Estrategia DDL | `create-drop` |
| SQL en consola | `show-sql=true` + `format_sql=true` |
| Thymeleaf | `cache=false` (modo desarrollo) |

---

##  4. Cómo ejecutar el proyecto

### 4.1 Requisitos previos
- **JDK 17** o superior configurado en el `PATH`
- **Apache Maven 3.8+** (o el wrapper `mvnw` incluido)
- IDE recomendado: IntelliJ IDEA / VS Code + Spring Boot Tools
- Cliente HTTP: **Postman**, **Bruno** o `curl`

### 4.2 Arranque en modo desarrollo

```bash
mvn spring-boot:run
````

svgsvg

### 4.3 Compilar y ejecutar como JAR

bash

```
mvn clean package
java -jar target/reservas-labs-api-1.0.0.jar
```

svgsvg

### 4.4 Ejecutar la suite de pruebas

bash

```
mvn test
```

svgsvg

Cuando la consola muestre `Started ReservasLabsApplication`, el sistema estará disponible en `http://localhost:8080`.

---

##  5. Endpoints y rutas

| **Componente**       | **URL**                                            | **Método**    | **Descripción**             |
| :------------------- | :------------------------------------------------- | :------------ | :-------------------------- |
| **Portal Web**       | `http://localhost:8080/reservas`                   | `GET`         | Listado general + métricas  |
| **Formulario Web**   | `http://localhost:8080/reservas/nueva`             | `GET`         | Formulario de nueva reserva |
| **Crear (Web)**      | `http://localhost:8080/reservas`                   | `POST`        | Procesa el formulario       |
| **Cancelar (Web)**   | `http://localhost:8080/reservas/{id}/cancelar`     | `POST`        | Cancela reserva desde la UI |
| **API Laboratorios** | `http://localhost:8080/api/laboratorios`           | `GET`, `POST` | Catálogo JSON               |
| **API Reservas**     | `http://localhost:8080/api/reservas`               | `GET`, `POST` | Listar / crear reservas     |
| **API Cancelar**     | `http://localhost:8080/api/reservas/{id}/cancelar` | `POST`, `PUT` | Cancelación controlada      |
| **Consola H2**       | `http://localhost:8080/h2-console`                 | `GET`         | Gestor web de la BD         |

---

##  6. Decisiones de diseño

Esta sección documenta **cuatro decisiones arquitectónicas clave**, justificando la alternativa elegida frente a la descartada.

---

###  Decisión 1 — ¿Dónde vive la detección de solapamientos: SQL/JPQL o Java puro en el Service?

**Elegido:** Consulta JPQL en `ReservaRepository.buscarSolapamientos()` + decisión final en `ReservaService`.

La condición matemática de solapamiento entre dos intervalos `[A, B)` y `[C, D)` es:

text

```
solapamiento ⟺ (r.inicio < fin) ∧ (r.fin > inicio)
```

svgsvg

Traer todas las reservas de un laboratorio a memoria (`findAll()` + streams) tendría un costo **O(N)** en transferencia de red y consumo de heap — inaceptable cuando la tabla crezca a miles de filas.

java

```
@Query("SELECT r FROM Reserva r WHERE r.laboratorio.id = :laboratorioId " +
       "AND r.estado <> com.universidad.reservaslabs.model.EstadoReserva.CANCELADA " +
       "AND r.inicio < :fin AND r.fin > :inicio")
List<Reserva> buscarSolapamientos(@Param("laboratorioId") Long labId,
                                  @Param("inicio") LocalDateTime inicio,
                                  @Param("fin") LocalDateTime fin);
```

svgsvg

**El Repositorio responde una pregunta de datos** (*"¿qué reservas chocan?"*). **El Service responde una pregunta de negocio** (*"¿se permite crear esta reserva?"*) lanzando `ReservaConflictException`. Si el Controller llamara directamente a `buscarSolapamientos()`, la regla de negocio se filtraría a la capa HTTP y quedaría duplicada entre el flujo REST y el flujo Web.

---

###  Decisión 2 — ¿Qué reglas necesitan al Repositorio y cuáles no?

Se aplicó el patrón **Fail-Fast**, dividiendo las validaciones en dos grupos:

**A) Reglas stateless (Java puro, sin BD) —** `validarHorarioYDuracion()`**:**

- Coherencia cronológica: `fin.isAfter(inicio)`
- Mismo día calendario: `inicio.toLocalDate().equals(fin.toLocalDate())`
- Duración entre 30 minutos y 3 horas
- Horario hábil: 07:00 a 21:00
- No permitir fechas pasadas

Se ejecutan en **microsegundos** antes de abrir cualquier transacción o consultar el pool JDBC. **Criterio general:** si la regla solo depende del objeto que se está validando, no hay motivo para involucrar la base de datos.

**B) Reglas stateful (requieren BD):**

- Existencia del laboratorio: `laboratorioRepository.findById(...)`
- Ausencia de solapamientos: `reservaRepository.buscarSolapamientos(...)`

**Criterio general:** si la regla necesita comparar contra otras filas que solo la base de datos conoce, la consulta pertenece al Repositorio y la decisión al Servicio.

---

###  Decisión 3 — ¿Cómo comparten el mismo Service la API REST y el Portal Web?

**Elegido:** Inyección de dependencias de **la misma instancia singleton** de `ReservaService` en ambos controladores.

java

```
@RestController
public class ReservaController {
    private final ReservaService reservaService;   // ← misma instancia
    ...
}

@Controller
public class ReservaWebController {
    private final ReservaService reservaService;   // ← misma instancia
    ...
}
```

svgsvg

**Alternativa descartada:** duplicar la lógica en un `ReservaWebService` o copiarla dentro del controlador MVC. Esto habría obligado a **modificar la regla de solapamiento en dos lugares** cada vez que cambiara, rompiendo el principio DRY y arriesgando inconsistencias silenciosas entre superficies.

Los controladores operan estrictamente como **adaptadores de entrada**: traducen HTTP a llamadas del dominio y el resultado a JSON o a vista Thymeleaf, sin decidir nada.

---

### 🧩 Decisión 4 — ¿Por qué dos manejadores de excepciones en vez de uno solo?

**Elegido:** Dos `@ControllerAdvice` desacoplados que parten del **mismo vocabulario de excepciones de dominio**.

text

```
                          ┌───────────────────────────────┐
                          │        ReservaService         │
                          │ lanza ReservaConflictException│
                          └───────────────┬───────────────┘
                                          │
                ┌─────────────────────────┴─────────────────────────┐
                ▼                                                   ▼
      ┌──────────────────────┐                        ┌──────────────────────┐
      │  ReservaController   │                        │ ReservaWebController │
      │   (@RestController)  │                        │     (@Controller)    │
      └──────────┬───────────┘                        └──────────┬───────────┘
                 │                                               │
                 ▼                                               ▼
      ┌──────────────────────┐                        ┌──────────────────────┐
      │ GlobalRestException  │                        │ ReservaWebException  │
      │   Handler            │                        │   Handler            │
      │ @RestControllerAdvice│                        │ @ControllerAdvice    │
      │ (annotations=RC)     │                        │ (assignableTypes=WC) │
      └──────────┬───────────┘                        └──────────┬───────────┘
                 │                                               │
                 ▼                                               ▼
        JSON estándar (ErrorResponse)              FlashAttribute "error" + redirect
        HTTP 409 / 404 / 400                       → /reservas/nueva o /reservas
```

svgsvg

Un `@RestControllerAdvice` **siempre serializa a JSON** — inútil para una vista HTML. Un manejador único tendría que inspeccionar el header `Accept` y ramificar por cada excepción, añadiendo complejidad condicional. Dos manejadores restringidos a su tipo de controlador mantienen la separación limpia de responsabilidades y garantizan que **el mismo error de negocio se presenta coherentemente en cada superficie**.

---

##  7. Pruebas automatizadas

### 7.1 `ReservaServiceTest` (JUnit 5 + Mockito)

- ✅ Creación exitosa con horario válido
- 🚫 Rechazo por solapamiento horario
- 🚫 Rechazo por duración < 30 min o > 3 h
- 🚫 Rechazo fuera de horario hábil (07:00–21:00)
- 🚫 Rechazo por fechas pasadas o invertidas
- ✅ Cancelación exitosa de reservas futuras
- 🚫 Rechazo de cancelación de reservas ya iniciadas

### 7.2 `ReservaControllerIntegrationTest` (Spring Boot Test + MockMvc)

- `POST /api/reservas` → **201 Created**
- `POST /api/reservas` (solapado) → **409 Conflict**
- `GET /api/reservas/{id}` inexistente → **404 Not Found**
- `GET /api/reservas` → **200 OK** con estructura JSON
- `POST /api/reservas/{id}/cancelar` → **200 OK**

---

##  8. Evidencias visuales de ejecución

Todas las capturas están almacenadas en la carpeta `evidencia/`.

###  8.1 Portal Web (Thymeleaf)

**A. Listado general de reservas —** `/reservas`

[https://./evidencia/reserva.png](./evidencia/reserva.png)

**B. Error de solapamiento en el formulario web —** `/reservas/nueva`

[https://./evidencia/solapamiento.png](./evidencia/solapamiento.png)

---

### 🔌 8.2 API REST (Postman)

**A. Creación exitosa de reserva —** `POST /api/reservas` **→** `201 Created`

[https://./evidencia/201created.png](./evidencia/201created.png)

**B. Error de solapamiento horario —** `POST /api/reservas` **→** `409 Conflict`

[https://./evidencia/409conflicted.png](./evidencia/409conflicted.png)

---

##  9. Estructura del repositorio

text

```
Ascanio-post1-u5/
├── evidencia/
│   ├── 201created.png
│   ├── 409conflicted.png
│   ├── reserva.png
│   └── solapamiento.png
├── src/
│   ├── main/
│   │   ├── java/com/universidad/reservaslabs/
│   │   │   ├── ReservasLabsApplication.java
│   │   │   ├── config/DataInitializer.java
│   │   │   ├── model/…
│   │   │   ├── repository/…
│   │   │   ├── service/…
│   │   │   ├── exception/…
│   │   │   ├── controller/…
│   │   │   └── web/…
│   │   └── resources/
│   │       ├── application.properties
│   │       └── templates/reservas/{lista,nueva}.html
│   └── test/java/com/universidad/reservaslabs/
│       ├── service/ReservaServiceTest.java
│       └── controller/ReservaControllerIntegrationTest.java
├── .gitignore
├── pom.xml
└── README.md
```

svgsvg

---

##  10. Convenciones de código aplicadas

- **Paquetes** en minúsculas y bien segmentados (`model`, `repository`, `service`, `controller`, `web`, `exception`, `config`).
- **Clases** en `PascalCase` (`ReservaService`, `GlobalRestExceptionHandler`).
- **Métodos y variables** en `camelCase` (`buscarSolapamientos`, `nombreSolicitante`).
- **Excepciones de dominio** con nombres que comunican intención (`ReservaConflictException`, `RecursoNoEncontradoException`).
- **Sin código comentado de depuración** ni `System.out.println` sueltos.
- **Inyección por constructor** en todos los beans.

---

##  11. Conclusiones

Diseñar correctamente una aplicación en capas exige tomar decisiones que van más allá de "seguir la receta". La parte más difícil de este laboratorio fue determinar **dónde ubicar cada regla de negocio**: las reglas puras (duración, horario institucional, coherencia cronológica) pertenecen al Service con Java puro, mientras que las reglas que comparan contra el estado global (existencia del laboratorio, solapamientos) exigen una consulta al Repositorio con el filtrado delegado al motor SQL. Comprobar que la API REST y el portal Thymeleaf pueden compartir la misma instancia de `ReservaService` sin duplicar lógica confirmó el valor real de la capa de servicio como punto único de gobierno del dominio. Finalmente, separar los dos `@ControllerAdvice` demostró que un mismo vocabulario de excepciones puede producir presentaciones radicalmente distintas (JSON semántico vs. redirección con flash) sin comprometer la coherencia del sistema.

---

##  12. Herramientas utilizadas

- Java 17 · Spring Boot 3.2.3 · Spring Data JPA · Hibernate · H2 · Thymeleaf
- JUnit 5 · Mockito · MockMvc
- Apache Maven · Postman · Git · GitHub · VS Code

---




