# Migraciones de bases de datos con Alembic y PostgreSQL

Guía práctica y reproducible sobre el versionado y la evolución controlada del esquema de una base de datos PostgreSQL usando [Alembic](https://alembic.sqlalchemy.org/).

Trabajo final de **Ingeniería de Software II** — Universidad Nacional de Tucumán, 2026.

## Requisitos

- [Docker](https://docs.docker.com/get-docker/) (con Docker Compose)
- [Python](https://www.python.org/downloads/) 3.11 o superior
- [Git](https://git-scm.com/)

## Índice

1. [Introducción](docs/01-introduccion.md)
2. [Entorno: PostgreSQL con Docker Compose y proyecto Alembic](docs/02-entorno.md)
3. [Primera migración manual](docs/03-primera-migracion.md)
4. [Autogenerate y sus límites](docs/04-autogenerate.md)
5. [Esquema más complejo: claves foráneas e índices](docs/05-esquema-complejo.md)
6. [Migración de datos](docs/06-migracion-de-datos.md)
7. [Reversión e historial](docs/07-reversion-e-historial.md)
8. [Trabajo en equipo: conflictos entre migraciones](docs/08-trabajo-en-equipo.md)
9. [Integración continua con GitHub Actions](docs/09-integracion-continua.md)
10. [Beneficios y buenas prácticas](docs/10-beneficios-y-buenas-practicas.md)
11. [Desafíos y consideraciones](docs/11-desafios-y-consideraciones.md)
12. [Conclusión](docs/12-conclusion.md)
13. [Referencias](docs/13-referencias.md)

## Estructura del repositorio

- `docs/`: secciones de la guía.
- `ejemplo/`: proyecto de ejemplo con Alembic que acompaña la guía.
- `.github/workflows/`: pipeline de integración continua.
