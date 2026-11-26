# 2. Entorno: PostgreSQL con Docker Compose y proyecto Alembic

En esta sección se arma un entorno **reproducible**: una base PostgreSQL que corre en un contenedor de Docker y un proyecto Python, aislado en un entorno virtual, con Alembic inicializado y conectado a esa base.

La idea es que cualquier integrante del equipo (y más adelante el pipeline de integración continua) obtenga exactamente el mismo entorno con unos pocos comandos, sin instalar PostgreSQL en su sistema ni mezclar dependencias con otros proyectos.

Todo el código de esta sección está en la carpeta [`ejemplo/`](../ejemplo/). Los comandos se ejecutan desde esa carpeta:

```bash
cd ejemplo
```

## 2.1. PostgreSQL con Docker Compose

**Por qué Docker:** si cada uno instala PostgreSQL a mano, termina con versiones y configuraciones distintas, y los errores de "en mi máquina anda" aparecen enseguida. Con Docker la versión de la base queda escrita en un archivo versionado junto al código, y levantarla o borrarla es un solo comando.

El archivo [`ejemplo/docker-compose.yml`](../ejemplo/docker-compose.yml) define un único servicio:

```yaml
services:
  postgres:
    image: postgres:16.4
    container_name: alembic-postgres
    environment:
      POSTGRES_USER: alembic
      POSTGRES_PASSWORD: alembic
      POSTGRES_DB: tienda
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U alembic -d tienda"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pgdata:
```

Qué hace cada parte:

- **`image: postgres:16.4`**: la versión está fijada. Usar `postgres:latest` haría que el entorno cambie solo el día que salga una versión nueva, que es justo lo que queremos evitar.
- **`environment`**: la imagen oficial usa estas variables para crear, en el primer arranque, el usuario `alembic` con clave `alembic` y la base `tienda`. Son credenciales de desarrollo local; nunca se usan en producción.
- **`ports: "5432:5432"`**: expone el puerto del contenedor en `localhost:5432`, para que Alembic (que corre fuera del contenedor) pueda conectarse. Si ya tenés un PostgreSQL local usando ese puerto, cambiá el de la izquierda (por ejemplo `"5433:5432"`) y ajustá la URL de conexión.
- **`volumes`**: los datos se guardan en el volumen con nombre `pgdata`, así que sobreviven a reinicios del contenedor.
- **`healthcheck`**: Docker consulta periódicamente `pg_isready` y marca el contenedor como `healthy` cuando la base ya acepta conexiones. Sirve para saber cuándo se puede empezar a usar (y en CI, para no correr migraciones antes de tiempo).

Levantar la base en segundo plano:

```bash
docker compose up -d
```

Verificar que está corriendo y sana (la columna `STATUS` debe decir `healthy` tras unos segundos):

```bash
docker compose ps
```

Verificar la conexión ejecutando `psql` **dentro** del contenedor:

```bash
docker compose exec postgres psql -U alembic -d tienda -c '\conninfo'
```

La salida esperada es algo como:

```text
You are connected to database "tienda" as user "alembic" via socket in "/var/run/postgresql" at port "5432".
```

Otros comandos útiles:

| Comando | Qué hace |
|---|---|
| `docker compose stop` | Detiene el contenedor, conserva los datos. |
| `docker compose down` | Elimina el contenedor, conserva el volumen con los datos. |
| `docker compose down -v` | Elimina contenedor **y** volumen: la base vuelve a estar vacía. Útil para probar las migraciones desde cero. |

## 2.2. Entorno virtual de Python

**Por qué un venv:** instalar paquetes con `pip` directamente en el Python del sistema mezcla las versiones de todos los proyectos. Un entorno virtual (`venv`) es una carpeta con su propio intérprete y sus propios paquetes; lo que se instala ahí no afecta a nada más y se puede borrar sin consecuencias.

Crear el entorno en la carpeta `.venv` (ya está en el `.gitignore`, porque no se versiona: cada uno lo genera en su máquina):

```bash
python -m venv .venv
```

> En algunas distribuciones de Linux el comando es `python3` en lugar de `python`.

Activarlo. En Linux y macOS:

```bash
source .venv/bin/activate
```

En Windows (PowerShell):

```powershell
.venv\Scripts\Activate.ps1
```

En Windows (cmd):

```bat
.venv\Scripts\activate.bat
```

Con el entorno activo, el prompt muestra `(.venv)` y los comandos `python`, `pip` y `alembic` pasan a ser los del entorno. Para salir se usa `deactivate`.

## 2.3. Dependencias

El archivo [`ejemplo/requirements.txt`](../ejemplo/requirements.txt) lista las dependencias con versiones fijas:

```text
sqlalchemy==2.1.9
alembic==1.20.1
psycopg[binary]==3.3.9
```

- **SQLAlchemy**: la biblioteca de acceso a bases de datos sobre la que está construido Alembic. Alembic la usa para conectarse y para describir las operaciones sobre el esquema.
- **Alembic**: la herramienta de migraciones.
- **psycopg** (versión 3): el *driver* que habla el protocolo de PostgreSQL. El extra `[binary]` instala una versión precompilada, así no hace falta tener instalada `libpq` ni un compilador.

Fijar las versiones (`==`) garantiza que todos instalen exactamente lo mismo, igual que con la imagen de Docker.

```bash
pip install -r requirements.txt
```

Comprobar que Alembic quedó instalado:

```bash
alembic --version
```

## 2.4. Inicializar el proyecto Alembic

```bash
alembic init migrations
```

Este comando crea el "entorno de migraciones": la estructura de archivos donde Alembic guarda su configuración y los scripts de migración. El argumento `migrations` es el nombre de la carpeta (por defecto la documentación usa `alembic`, pero `migrations` deja más claro qué contiene).

```text
ejemplo/
├── alembic.ini
└── migrations/
    ├── README
    ├── env.py
    ├── script.py.mako
    └── versions/
```

Qué es cada archivo:

- **`alembic.ini`**: configuración general. Indica dónde está la carpeta de migraciones (`script_location`), el formato de los nombres de archivo y la configuración de *logging*. Es el archivo que busca el comando `alembic` en el directorio actual.
- **`migrations/env.py`**: un script de Python que Alembic ejecuta **cada vez** que se corre un comando. Es quien crea la conexión a la base y lanza las migraciones. Como es código Python normal, se puede adaptar: acá es donde se configura la URL de conexión (siguiente apartado) y, más adelante, donde se enlazan los modelos para `autogenerate`.
- **`migrations/script.py.mako`**: la plantilla con la que Alembic genera cada nuevo archivo de migración (con las funciones `upgrade()` y `downgrade()` vacías). Si el equipo quiere que todas las migraciones tengan, por ejemplo, un encabezado común, se modifica acá.
- **`migrations/versions/`**: la carpeta donde van a quedar los scripts de migración, uno por cambio de esquema. Ahora está vacía; se le agrega un `.gitkeep` para que Git la versione igual.
- **`migrations/README`**: una línea que describe el tipo de plantilla usada. No tiene efecto.

## 2.5. Configurar la URL de conexión

Alembic se conecta mediante una URL de SQLAlchemy con este formato:

```text
postgresql+psycopg://USUARIO:CLAVE@HOST:PUERTO/BASE
```

La parte `postgresql+psycopg` indica el dialecto (PostgreSQL) y el *driver* (psycopg 3). Para la base del `docker-compose.yml` queda:

```text
postgresql+psycopg://alembic:alembic@localhost:5432/tienda
```

Por defecto, `alembic init` deja esa URL en `alembic.ini` (`sqlalchemy.url = driver://user:pass@localhost/dbname`). **No conviene dejarla ahí**: `alembic.ini` se versiona, así que cualquier credencial que se escriba en él termina en el repositorio, y además cada entorno (local, CI, producción) necesita una URL distinta.

Por eso la línea queda comentada en `alembic.ini`:

```ini
# La URL de conexión NO se define acá: migrations/env.py la lee de la
# variable de entorno DATABASE_URL (ver docs/02-entorno.md).
# sqlalchemy.url =
```

y en `migrations/env.py`, justo después de obtener el objeto `config`, se toma de la variable de entorno `DATABASE_URL`:

```python
import os
...
config = context.config

# URL de conexión: se toma de la variable de entorno DATABASE_URL para no
# dejar credenciales fijas en alembic.ini. Si no está definida, se usa la base
# local que levanta docker-compose.yml (solo para desarrollo).
DEFAULT_DATABASE_URL = "postgresql+psycopg://alembic:alembic@localhost:5432/tienda"
database_url = os.environ.get("DATABASE_URL", DEFAULT_DATABASE_URL)
# configparser interpreta "%" como interpolación, por eso se escapa.
config.set_main_option("sqlalchemy.url", database_url.replace("%", "%%"))
```

Algunos detalles:

- `config.set_main_option` carga la URL en la configuración en memoria, así el resto de `env.py` generado por Alembic (modo *online* y *offline*) funciona sin más cambios.
- El valor por defecto apunta a la base de desarrollo del `docker-compose.yml`, para que el ejemplo funcione sin configurar nada. Esas credenciales son públicas a propósito; en cualquier entorno real se define `DATABASE_URL`.
- El `replace("%", "%%")` evita un error si la clave contiene `%` (por ejemplo, caracteres codificados en la URL), porque `alembic.ini` se lee con `configparser`, que usa `%` para interpolar valores.

Para apuntar a otra base basta con definir la variable antes de correr Alembic. En Linux y macOS:

```bash
export DATABASE_URL="postgresql+psycopg://alembic:alembic@localhost:5432/tienda"
```

En Windows (PowerShell):

```powershell
$env:DATABASE_URL = "postgresql+psycopg://alembic:alembic@localhost:5432/tienda"
```

## 2.6. Verificar que todo funciona

Con la base levantada y el venv activo:

```bash
alembic current
```

La salida esperada es:

```text
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

`alembic current` muestra en qué revisión está la base. Todavía no hay migraciones, así que no imprime ninguna revisión, pero que no haya errores confirma que Alembic leyó la configuración, encontró el *driver* y se pudo conectar a PostgreSQL. Si la conexión falla (por ejemplo, la base no está levantada o la clave es incorrecta), el comando termina con un error de `psycopg.OperationalError`.

## 2.7. Resumen: desde cero

Para quien clona el repositorio, todo el entorno se arma con estos pasos (en Linux/macOS):

```bash
cd ejemplo
```

```bash
docker compose up -d
```

```bash
python -m venv .venv
```

```bash
source .venv/bin/activate
```

```bash
pip install -r requirements.txt
```

```bash
alembic current
```

`alembic init` **no** se vuelve a ejecutar: el proyecto ya inicializado (`alembic.ini` y `migrations/`) está versionado en el repositorio.

En la siguiente sección se escribe la primera migración a mano.
