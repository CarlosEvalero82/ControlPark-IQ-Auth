# ControlPark-IQ · Servicio web de registro e inicio de sesión

**Evidencia:** GA7-220501096-AA5-EV01 · Diseño y desarrollo de servicios web - caso
**Programa:** Tecnólogo en Análisis y Desarrollo de Software · SENA · Ficha 3235898
**Aprendiz:** Carlos Eduardo Valero
**Instructora:** Adriana Milena Rangel Márquez
**Proyecto:** ControlPark-IQ, sistema de gestión de parqueaderos para conjuntos residenciales

Servicio web REST (API) que permite **registrar** usuarios de ControlPark-IQ (administrador, portería, residente) e **iniciar sesión**. Recibe un usuario y una contraseña. Si la autenticación es correcta, responde **"Autenticación satisfactoria"**; si no, responde **"Error en la autenticación"**.

---

## 1. Diseño del servicio: tabla de endpoints

URL base: `http://localhost:8080`

| Endpoint | Recibe (JSON) | Si todo sale bien | Si algo falla |
|---|---|---|---|
| `POST /api/auth/registro` | `usuario`, `contrasena` (y `rol` opcional) | **201** · `Usuario registrado correctamente` | **400** `Datos inválidos` · **409** `El usuario ya existe` |
| `POST /api/auth/login` | `usuario`, `contrasena` | **200** · `Autenticación satisfactoria` | **400** `Datos inválidos` · **401** `Error en la autenticación` |

Ejemplo de petición:

```json
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
    "usuario": "porteria.torre1",
    "contrasena": "Parque2026"
}
```

Ejemplo de respuesta:

```json
HTTP/1.1 200 OK
{
    "exito": true,
    "mensaje": "Autenticación satisfactoria"
}
```

Cuando hay errores de validación, la respuesta trae además la lista `errores`:

```json
HTTP/1.1 400 Bad Request
{
    "exito": false,
    "mensaje": "Datos inválidos",
    "errores": ["El usuario debe tener entre 4 y 20 caracteres", "El usuario es obligatorio"]
}
```

### Reglas de seguridad

- **La contraseña se guarda protegida con BCrypt**, nunca en texto plano. En el login se compara la contraseña escrita con el hash guardado (`matches`).
- **El mensaje de error del login es genérico:** siempre `Error en la autenticación`, falle el usuario, la contraseña o el usuario esté inactivo. Así no se le dan pistas a un atacante.
- El repositorio **no tiene contraseñas reales de MySQL**: se leen de las variables de entorno `DB_USERNAME` y `DB_PASSWORD`.

### Validaciones

| Campo | Registro | Login |
|---|---|---|
| `usuario` | Obligatorio · 4 a 20 caracteres · letras, números, `.` o `_` · único (no distingue mayúsculas) | Obligatorio |
| `contrasena` | Obligatoria · 8 a 64 caracteres · al menos una letra y un número | Obligatoria |
| `rol` | Opcional: `ADMINISTRADOR`, `PORTERIA` o `RESIDENTE` (por defecto `RESIDENTE`) | No aplica |

## 2. Arquitectura en capas

```
Cliente (Postman / React) → Controller → Service → Repository → MySQL
```

| Capa | Clase | Responsabilidad |
|---|---|---|
| Controlador ("mesero") | `controller/AuthController` | Recibe la petición HTTP y responde con el código y el mensaje |
| Servicio ("chef") | `service/AuthService` | Lógica: valida si existe, protege la clave con BCrypt, compara |
| Repositorio ("bodega") | `repository/UsuarioRepository` | Consulta y guarda en la tabla `usuarios` |
| Modelo | `model/Usuario`, `model/Rol` | Entidad JPA de la tabla `usuarios` y roles del sistema |
| DTO | `dto/RegistroDTO`, `dto/CredencialesDTO`, `dto/RespuestaDTO` | Datos que entran (con validaciones) y respuesta JSON |
| Configuración | `config/SeguridadConfig` | Crea el codificador BCrypt |
| Errores | `exception/ManejadorErrores` | Convierte errores de validación en 400 y duplicados en 409 |

## 3. Estructura del proyecto

```
ControlPark-IQ-Auth/
├── pom.xml
├── README.md
├── database/
│   └── controlpark_auth.sql            # script de la BD (opcional: Spring la crea sola)
├── ENDPOINTS.txt                       # endpoints de la API (AA5-EV02)
├── postman/
│   ├── ControlPark-IQ-Auth.postman_collection.json
│   └── ControlPark-IQ-Local.postman_environment.json
└── src/
    ├── main/
    │   ├── java/co/edu/sena/controlparkiq/
    │   │   ├── ControlParkAuthApplication.java
    │   │   ├── config/SeguridadConfig.java
    │   │   ├── controller/AuthController.java
    │   │   ├── dto/CredencialesDTO.java · RegistroDTO.java · RespuestaDTO.java
    │   │   ├── exception/ManejadorErrores.java
    │   │   ├── model/Usuario.java · Rol.java
    │   │   ├── repository/UsuarioRepository.java
    │   │   └── service/AuthService.java
    │   └── resources/application.properties
    └── test/
        ├── java/co/edu/sena/controlparkiq/AuthControllerTest.java
        └── resources/application.properties   # H2 en memoria solo para pruebas
```

## 4. Tecnologías

| Tecnología | Uso |
|---|---|
| Java 17+ · Spring Boot 3.3.5 | Proyecto base y servidor en el puerto 8080 |
| Spring Web | Controlador REST y JSON |
| Spring Data JPA + MySQL 8 | Tabla `usuarios` en la base `controlpark_auth` |
| Validation (Jakarta) | `@NotBlank`, `@Size`, `@Pattern` y `@Valid` |
| spring-security-crypto | BCrypt (sin `spring-boot-starter-security`) |
| JUnit 5 + MockMvc + H2 | Pruebas automáticas |
| Postman | Pruebas manuales (colección incluida) |
| Git + GitHub | Control de versiones |

## 5. Cómo ejecutarlo

1. Tener instalado **Java 17 o superior**, **MySQL 8** (con MySQL Workbench) e **IntelliJ IDEA** o VS Code.
2. Clonar el repositorio:
   ```bash
   git clone https://github.com/CarlosEvalero82/ControlPark-IQ-Auth.git
   ```
3. Abrir la carpeta en IntelliJ (File → Open → `pom.xml` → Open as Project) y esperar a que Maven descargue las dependencias.
4. Configurar el usuario de MySQL. Si su usuario `root` **no tiene contraseña**, no hay que hacer nada. Si **tiene contraseña**, defínala como variable de entorno `DB_PASSWORD` en la configuración de ejecución (Run → Edit Configurations → Environment variables: `DB_PASSWORD=su_clave`).
5. Ejecutar `ControlParkAuthApplication` (botón **Run**) o `mvn spring-boot:run`.
   En consola debe aparecer `Tomcat started on port 8080` y `Started ControlParkAuthApplication`.
   La base `controlpark_auth` y la tabla `usuarios` se crean solas.
6. Probar con Postman: ver la sección 7 de este README.

## 6. Pruebas automáticas

```bash
mvn test
```

Ejecuta 7 pruebas (`AuthControllerTest`) sobre una base H2 en memoria, sin necesidad de MySQL: los 6 casos de prueba (registro correcto, repetido y vacío; login correcto, con contraseña incorrecta y con usuario inexistente) y la verificación de que la contraseña queda protegida con BCrypt.

## 7. Pruebas con Postman (AA5-EV02)

Las pruebas manuales de la API se hacen con **Postman**. En la carpeta `postman/` están:

| Archivo | Para qué sirve |
|---|---|
| `ControlPark-IQ-Auth.postman_collection.json` | Colección con **19 pruebas**: 11 de registro, 5 de inicio de sesión y 3 de errores generales (405, 415, 404). Cada petición trae sus verificaciones automáticas (código HTTP, mensaje, errores, que nunca devuelva la contraseña y tiempo de respuesta) |
| `ControlPark-IQ-Local.postman_environment.json` | Entorno con la variable `baseUrl = http://localhost:8080` |

Cómo usarlas:
1. Iniciar la API (`ControlParkAuthApplication`) y comprobar que escucha en el puerto 8080.
2. En Postman: **Import** → seleccionar los dos archivos de `postman/`.
3. Elegir el entorno **ControlPark-IQ Local** (arriba a la derecha).
4. Ejecutar toda la colección con **Run collection**. La primera petición genera un sufijo aleatorio, así la colección se puede repetir sin vaciar la tabla `usuarios`.

La lista completa de endpoints está en [`ENDPOINTS.txt`](ENDPOINTS.txt).

## 8. Control de versiones

El proyecto se creó con Git desde el primer archivo. Cada avance quedó en un commit con un mensaje claro (`feat:`, `test:`, `docs:`). Para ver el historial:

```bash
git log --oneline
```
