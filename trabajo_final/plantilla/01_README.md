# Trabajo Final — Guillermo Castro Abarca
## Blueprint elegido
Escape Room

## Descripcion
Backend de simulación para un sistema de Escape Room digitalizado. La aplicación permite gestionar salas temáticas, jugadores y el estado de las partidas en tiempo real, registrando información como tiempos de juego, puzzles resueltos y resultados de escape.

## Entidades

### Entidades Principales

#### `Jugador`
Representa a un jugador registrado en el sistema.

| Campo     | Tipo         | Descripción                            |
|-----------|--------------|----------------------------------------|
| `id`      | `Long`       | Clave primaria autoincremental         |
| `nombre`  | `String`     | Nombre único (2–100 caracteres)        |
| `enJuego` | `boolean`    | Si el jugador está en partida activa   |
| `sala`    | `SalaEscape` | Sala a la que pertenece (`@ManyToOne`) |

#### `SalaEscape`
Representa una sala temática del escape room.

| Campo                 | Tipo            | Descripción                                     |
|-----------------------|-----------------|-------------------------------------------------|
| `id`                  | `Long`          | Clave primaria autoincremental                  |
| `nombre`              | `String`        | Nombre único de la sala                         |
| `tematica`            | `TipoTematica`  | `TERROR` o `AVENTURA` (asignada aleatoriamente) |
| `tiempoMaximoMinutos` | `int`           | Tiempo límite de la partida                     |
| `tiempoInicio`        | `LocalDateTime` | Marca temporal de inicio de partida             |
| `enJuego`             | `boolean`       | Si la partida está activa                       |
| `jugadoresConectados` | `List<Jugador>` | Jugadores en la sala (`@OneToMany`)             |

#### `Partida`
Registra el historial de cada juego jugado, persistiendo los resultados en base de datos.

| Campo                   | Tipo            | Descripción                                  |
|-------------------------|-----------------|----------------------------------------------|
| `id`                    | `Long`          | Clave primaria autoincremental               |
| `sala`                  | `SalaEscape`    | Sala donde se jugó (`@ManyToOne`)            |
| `fechaInicio`           | `LocalDateTime` | Cuándo empezó la partida                     |
| `fechaFin`              | `LocalDateTime` | Cuándo terminó la partida                    |
| `completada`            | `boolean`       | `true` = escaparon, `false` = tiempo agotado |
| `jugadores`             | `List<Jugador>` | Quiénes jugaron (`@ManyToMany`)              |

#### Relación entre entidades

```
SalaEscape  1 ──────────── * Jugador
            @OneToMany      @ManyToOne

SalaEscape  1 ──────────── * Partida
            @OneToMany      @ManyToOne

Partida     * ──────────── * Jugador
            @ManyToMany     (tabla: partida_jugador)
```

## Endpoints de la API
Secuencia completa de llamadas para probar el sistema de extremo a extremo:

```bash
# 1. Crear jugadores
POST /jugadores?nombre=Alice
POST /jugadores?nombre=Bob

# 2. Crear sala
POST /salas?nombre=SalaDemo

# 3. Unir jugadores (usar los IDs devueltos en los pasos anteriores)
POST /salas/1/unirse?jugadorId=1
POST /salas/1/unirse?jugadorId=2

# 4. Verificar estado
GET /salas → estado: LISTA_PARA_INICIAR

# 5. Iniciar partida
POST /salas/1/iniciar

# 6. Consultar estado de juego
GET /salas/1/estado → puzzlesDisponibles: ["terror"] o ["aventura"]

# 7. Resolver puzzles en cadena
POST /salas/responder
Body: { "salaId": 1, "jugador": "Alice", "puzzleId": "terror", "respuesta": "666" }

POST /salas/responder
Body: { "salaId": 1, "jugador": "Bob", "puzzleId": "puerta_sotano", "respuesta": "llave" }

POST /salas/responder
Body: { "salaId": 1, "jugador": "Alice", "puzzleId": "salida", "respuesta": "abrir" }

# 8. Verificar victoria
GET /salas/1/estado → puzzlesResueltos: ["terror", "puerta_sotano", "salida"], puzzlesDisponibles: []

# 9. Consultar historial de partidas
GET /partidas
GET /partidas/sala/1
```


## Como ejecutar

## 🐳 Despliegue con Docker

La forma más sencilla de ejecutar la aplicación es usando Docker Compose, que levanta automáticamente la app y PostgreSQL sin necesidad de instalar nada más que Docker.

### Prerrequisitos

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y en ejecución

> **Windows:** asegúrate de que la virtualización está activada en la BIOS (opción `SVM Mode` en AMD o `VT-x` en Intel) y de que Docker Desktop está corriendo antes de ejecutar los comandos.

### Estructura de ficheros necesaria

Asegúrate de tener estos ficheros en la raíz del proyecto junto al `pom.xml`:

```
EscapeRoom/
├── Dockerfile
├── docker-compose.yml
├── .env                  ← créalo a partir de .env.example
├── .dockerignore
├── pom.xml
└── src/
```

### Pasos para desplegar

**1. Clona el repositorio**
```bash
git clone <url-del-repositorio>
cd EscapeRoom
```

**2. Crea el fichero de variables de entorno**
```bash
# Linux / macOS
cp .env.example .env

# Windows (PowerShell)
cp .env.example .env
```

Edita `.env` si quieres cambiar las credenciales de la base de datos:
```env
DB_USERNAME=postgres
DB_PASSWORD=postgres
```

**3. Compila el JAR localmente**
```bash
# Linux / macOS
./mvnw clean package -DskipTests

# Windows
.\mvnw clean package -DskipTests
```

**4. Levanta los contenedores**
```bash
docker compose up --build
```

Este comando hace lo siguiente:
- Descarga la imagen de PostgreSQL 16
- Copia el JAR compilado a una imagen JRE ligera
- Levanta PostgreSQL y espera a que esté sano antes de arrancar la app

**5. Verifica que todo funciona**

Cuando veas esta línea en los logs, la app está lista:
```
escaperoom-app | Started EscapeRoomApplication in X seconds
```

Abre en el navegador:
- **Swagger UI:** http://localhost:8080/swagger-ui/index.html
- **API docs (JSON):** http://localhost:8080/v3/api-docs

### Comandos útiles

```bash
# Ver logs en tiempo real
docker compose logs -f

# Ver logs solo de la app
docker compose logs -f app

# Parar los contenedores (conserva los datos)
docker compose down

# Parar y borrar todos los datos (volumen de PostgreSQL)
docker compose down -v

# Reiniciar solo la app sin reconstruir
docker compose restart app

# Reconstruir la imagen tras cambios en el código
docker compose up --build
```

### Conexión a PostgreSQL desde un cliente externo

Puedes conectarte a la base de datos con cualquier cliente (DBeaver, pgAdmin, IntelliJ...):

| Campo      | Valor        |
|------------|--------------|
| Host       | `localhost`  |
| Puerto     | `5432`       |
| Base datos | `escaperoom` |
| Usuario    | `postgres`   |
| Contraseña | `postgres`   |

### Cómo funciona el Dockerfile

El `Dockerfile` copia el JAR precompilado localmente a una imagen JRE ligera:

```
Runtime  →  eclipse-temurin:21-jre-jammy
             - Copia el JAR desde target/
             - Corre con usuario no-root (appuser)
             - Expone el puerto 8080
```

> **Nota:** antes de hacer `docker compose up --build` hay que compilar el JAR localmente:
> ```bash
> # Linux / macOS
> ./mvnw clean package -DskipTests
>
> # Windows
> .\mvnw clean package -DskipTests
> ```
>
>
> <br>
Nota: asegurate de configurar <br>
$env:JAVA_HOME = "C:\Users\guill\.jdks\ms-21.0.10"<br>
$env:Path = "$env:JAVA_HOME\bin;" + $env:Path<br>
---

## ▶️ Ejecución local (sin Docker)

### Prerrequisitos

- Java 21+
- Maven 3.9+ (o usa el wrapper `./mvnw`)
- PostgreSQL 16+ en ejecución local

### Pasos

```bash
# 1. Clonar el repositorio
git clone <url-del-repositorio>
cd EscapeRoom

# 2. Crear la base de datos en PostgreSQL
psql -U postgres -c "CREATE DATABASE escaperoom;"

# 3. Activar configuración PostgreSQL en application.properties
# Comenta las líneas de H2 y descomenta las de PostgreSQL

# 4. Compilar y ejecutar
./mvnw spring-boot:run

# Windows
.\mvnw spring-boot:run

# 5. Acceder a Swagger UI
# http://localhost:8080/swagger-ui/index.html
```

---

## Base de Datos

El proyecto usa **PostgreSQL**. Hibernate gestiona el esquema automáticamente con `ddl-auto=update`, creando las tablas `jugador` y `sala_escape` en el primer arranque. No es necesario ejecutar ningún script SQL manualmente.

Con Docker Compose la base de datos se crea y configura de forma automática.

---
