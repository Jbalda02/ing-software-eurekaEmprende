# ConectaEmprende

Plataforma web de apoyo a emprendedores: publicación de emprendimientos, blog, eventos, autoevaluaciones, métricas y generación de hojas de ruta asistidas por IA.

Proyecto de **Ingeniería de Software I** — Facultad de Ingenierías, Universidad de Especialidades Espíritu Santo (UEES). Sistema heredado del semestre anterior, actualmente en evolución.

| Componente | Tecnología | Repositorio |
|---|---|---|
| Frontend | Angular 17.3 · TypeScript 5.4 · SSR | [ConectaEmprende](https://github.com/Jbalda02/ConectaEmprende) |
| Backend | Spring Boot 3.4.4 · Java 21 | [eureka-emprende](https://github.com/Jbalda02/eureka-emprende) |
| Base de datos | PostgreSQL 17 | — |

---

## Estructura del repositorio

Este repositorio agrupa la documentación del proyecto e incluye el código como **submódulos de Git**:

```
ing-software-eurekaEmprende/
├── ConectaEmprende/     → submódulo: frontend Angular
├── eureka-emprende/     → submódulo: backend Spring Boot
└── README.md
```

Por ser submódulos, **un `git clone` normal deja esas carpetas vacías**. Usa el comando de la sección siguiente.

---

## 1. Clonar el proyecto

```bash
# Windows: habilita rutas largas antes de clonar (las rutas de Angular superan el límite de 260 caracteres)
git config --global core.longpaths true

git clone --recurse-submodules https://github.com/Jbalda02/ing-software-eurekaEmprende.git
cd ing-software-eurekaEmprende
```

> Si ya clonaste sin `--recurse-submodules`, ejecuta `git submodule update --init --recursive`.

---

## 2. Requisitos previos

| Herramienta | Versión | Necesaria para |
|---|---|---|
| **Docker Desktop** | 4.x o superior | Ruta recomendada |
| **JDK** | **21** (obligatorio) | Compilar el backend sin Docker |
| **PostgreSQL client** | 17 | El comando `pg_restore` |
| **Node.js** | 18.19+ o 20.11+ | Frontend (Angular 17) |

El backend **no compila con JDK 17 ni con JDK 24**. Verifica con `java -version`.

---

## 3. Configurar las variables de entorno

En la carpeta `eureka-emprende`, copia la plantilla y completa los valores:

```bash
cd eureka-emprende
cp .env.example .env
```

| Variable | Uso |
|---|---|
| `AWS_ACCESS_KEY_ID` | Almacenamiento de multimedia en S3 |
| `AWS_SECRET_ACCESS_KEY` | Almacenamiento de multimedia en S3 |
| `GPT_KEY` | Generación de roadmaps con IA |

Sin `GPT_KEY` la aplicación arranca, pero la generación de roadmaps falla.

> El archivo `.env` está excluido del control de versiones y **no debe subirse nunca** al repositorio.

---

## 4. Restaurar la base de datos — **paso obligatorio**

El proyecto **no crea las tablas automáticamente**: `spring.jpa.hibernate.ddl-auto` no está definido, así que Hibernate usa el valor por defecto `none`.

Si te saltas este paso, **la aplicación arranca sin errores y Swagger carga bien**, pero toda operación contra la base de datos falla. Es el error de levantamiento más común.

El respaldo `dump-copy-1-eurekaEmprende-202602170822.sql` está incluido en la raíz de este repositorio. Está en **formato personalizado de PostgreSQL** — se restaura con `pg_restore`, no con `psql -f`.

```bash
# 1. Levanta solo la base de datos
cd eureka-emprende
docker compose up -d postgres

# 2. Copia el respaldo al contenedor
docker compose cp ../dump-copy-1-eurekaEmprende-202602170822.sql postgres:/tmp/dump.sql

# 3. Restaura
docker compose exec postgres pg_restore -U postgres -d eurekaEmprende \
  --no-owner --clean --if-exists /tmp/dump.sql
```

Verifica que se hayan creado las 40 tablas:

```bash
docker compose exec postgres psql -U postgres -d eurekaEmprende -c "\dt"
```

Las advertencias sobre roles inexistentes son inofensivas gracias a `--no-owner`.

---

## 5. Levantar el backend

```bash
cd eureka-emprende
docker compose up -d --build
```

La primera compilación descarga las dependencias de Maven y tarda varios minutos. La API queda en **http://localhost:8080**.

Ver los logs de arranque:

```bash
docker compose logs -f backend
```

<details>
<summary>Alternativa: sin Docker</summary>

Requiere PostgreSQL 17 en `localhost:5432` con la base ya restaurada. Credenciales por defecto: `postgres` / `postgres`.

```bash
./mvnw spring-boot:run        # Linux / macOS
.\mvnw.cmd spring-boot:run    # Windows
```
</details>

---

## 6. Levantar el frontend

```bash
cd ConectaEmprende
npm install
npx ng serve
```

Queda en **http://localhost:4200** y consume la API en `http://localhost:8080`, según `src/environments/environment.ts`. Ese origen ya está autorizado en la configuración CORS del backend.

> **No uses `npm start` para desarrollo.** Ese script ejecuta `node server.js`, que sirve el build de producción desde `dist/` y falla si no compilaste antes. Para producción con SSR: `npm run build && npm start`.

---

## 7. Verificar que todo funciona

| Comprobación | URL | Resultado esperado |
|---|---|---|
| Salud del backend | http://localhost:8080/actuator/health | `{"status":"UP"}` |
| Documentación de la API | http://localhost:8080/swagger-ui.html | Swagger UI con los controladores |
| Base de datos restaurada | http://localhost:8080/v1/categorias | Lista de categorías |
| Frontend | http://localhost:4200 | Página de inicio con emprendimientos |

Si las tres primeras responden pero el frontend muestra listas vacías, el problema es CORS.

---

## 8. Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Las carpetas de los submódulos están vacías | Clonaste sin `--recurse-submodules` | `git submodule update --init --recursive` |
| `Filename too long` al clonar | Límite de rutas de Windows | `git config --global core.longpaths true` |
| La API responde pero todo sale vacío o da error 500 | La base de datos no se restauró | Ejecuta el paso 4 |
| `port 5432 already allocated` | PostgreSQL nativo ocupando el puerto | `services.msc` → `postgresql-x64-XX` → Detener |
| `input file appears to be a text format dump` | Usaste `psql` en vez de `pg_restore` | Usa `pg_restore` (paso 4) |
| El backend no compila | JDK distinto de 21 | Instala JDK 21 y ajusta `JAVA_HOME` |
| La generación de roadmap falla | Falta `GPT_KEY` | Revisa el `.env` |
| Los cambios del `.env` no se aplican | Compose cachea el entorno | `docker compose up -d --force-recreate backend` |

---

## 9. Detener el entorno

```bash
docker compose down       # detiene los contenedores, conserva los datos
docker compose down -v    # ELIMINA también la base de datos
```

`docker compose down -v` borra el volumen `postgres_data`. Después habría que repetir el paso 4.

---

## Documentación adicional

| Documento | Contenido |
|---|---|
| `eureka-emprende/DOCUMENTACION_TECNICA.md` | Arquitectura hexagonal, catálogo de los 13 módulos, modelo de datos, seguridad JWT |
| `eureka-emprende/ESTRUCTURA_PROYECTO.md` | Estructura de carpetas del backend |
| `ConectaEmprende/DOCUMENTACION_TECNICA.md` | Arquitectura del frontend, rutas, servicios, SSR |

## Trabajar con los submódulos

```bash
# Traer los últimos cambios de todos los submódulos
git submodule update --remote --merge

# Trabajar dentro de un submódulo (es un repositorio completo)
cd eureka-emprende
git checkout master
git pull

# Tras commitear en un submódulo, registra el nuevo puntero en este repositorio
cd ..
git add eureka-emprende
git commit -m "Actualizar puntero del submodulo eureka-emprende"
```
