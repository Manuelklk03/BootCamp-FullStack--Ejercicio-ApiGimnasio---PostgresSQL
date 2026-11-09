# Entrega: API del gimnasio FitVal con NestJS y PostgreSQL

## Objetivo

En la entrega de SQL diseñaste la base de datos del gimnasio **FitVal** y la consultabas a mano. Ahora vas a construir la **API** que usaría la web del gimnasio: un servidor **NestJS** que lee y escribe en esa base de datos, con **registro, login y permisos**.

Al terminar deberías poder:

- Crear un proyecto NestJS y conectarlo a PostgreSQL con `pg`.
- Convertir las consultas de tu entrega de SQL (`JOIN`, `INSERT`, `DELETE`…) en rutas de una API.
- Escribir consultas con `$1`, `$2`… y explicar por qué evitan la **inyección SQL**.
- Registrar usuarios, hacer login con **JWT** y proteger rutas con **guards**.
- Devolver el código de estado correcto: `200`, `201`, `400`, `401`, `403`, `404`, `409`.
- Trabajar como en una empresa: `main` protegida, una **rama `feature/` y una pull request** por funcionalidad, y un **CI** que revisa tu código.

**Tiempo orientativo:** 4–6 horas.

**Qué vas a entregar:** un repositorio `api-fitval` en GitHub con el código, una PR por cada parte y un archivo `respuestas.md` (detalles en el apartado [Entrega](#entrega)).

**Cómo trabajarla:**

- Haz las partes **en orden**. Cada parte termina con una PR que fusionas antes de empezar la siguiente.
- Ten a mano los apuntes [03a. NestJS](../sesiones/backend/03a_nestjs_fundamentos.md), [03b. NestJS + PostgreSQL](../sesiones/backend/03b_nestjs_postgresql.md) y [04. Autenticación](../sesiones/backend/04_nestjs_autenticacion.md). Esta entrega usa lo mismo, aplicado al gimnasio.
- Una ruta no está terminada hasta que pruebas también **lo que falla**. Cada parte tiene su tabla de pruebas. Usa Postman o `curl`.
- Las preguntas son pocas y cortas: respóndelas en `respuestas.md` con una o dos frases.

### Todo en inglés

El código se escribe en **inglés**: tablas, columnas, rutas, variables y tipos. Los datos de FitVal son los mismos que en la entrega de SQL, con los nombres traducidos:

| Entrega de SQL | Esta entrega |
|---|---|
| `planes` (`nombre`, `precio_mensual`) | `plans` (`name`, `monthly_price`) |
| `socios` (`fecha_alta`) | `members` (`joined_at`) |
| `tarjetas` (`codigo`, `activa`) | `cards` (`code`, `active`) |
| `monitores` | `trainers` |
| `clases` (`actividad`, `sala`, `fecha_hora`, `plazas`) | `classes` (`activity`, `room`, `starts_at`, `spots`) |
| `reservas` (`fecha_reserva`) | `bookings` (`booked_at`) |
| `especialidades`, `monitor_especialidades` | `specialties`, `trainer_specialties` |

Lo que lee una persona (el nombre de un plan, los mensajes de error, los comentarios del código) puede seguir en español.

---

## Las reglas de la API

Vuelve aquí cada vez que dudes qué código devolver.

| Código | Cuándo |
|---|---|
| `200` | Todo bien. Devuelve los datos. |
| `201` | Se ha creado algo. Devuelve lo creado, con su `id`. |
| `400` | El cliente ha enviado algo mal: un id que no es un número, un campo que falta, un `trainerId` que no existe… |
| `401` | No sabemos quién eres: falta el token o no es válido. |
| `403` | Sabemos quién eres, pero no tienes permiso. |
| `404` | El id que viene **en la ruta** (`/api/classes/:id`) no existe. |
| `409` | La petición es correcta, pero choca con lo que hay. Ejemplo: borrar una clase que tiene reservas. |
| `500` | Error nuestro (por ejemplo, PostgreSQL parado). El cliente solo ve un mensaje genérico. |

Una regla más: si un id llega **en el body** (por ejemplo `trainerId`) y no existe, es `400`: la petición está mal hecha.

NestJS responde los errores en JSON él solo: `{ "message": "...", "error": "Not Found", "statusCode": 404 }`.

---

# PARTE 0 — Prepara el repositorio con CI

Antes de escribir la API vas a dejar el repositorio listo para trabajar con PR:

1. Un fichero `ci.yml` que revisa tu código con **ESLint** cada vez que subes algo.
2. La rama `main` protegida: solo se puede cambiar con una PR, y solo si los checks salen en verde ✅.

Es lo mismo que en la entrega de Express. La guía completa, con capturas, está en [Instrucciones del CI mínimo](https://github.com/TheBridge-FullStackDeveloper/temario_fullstack_FT_LABORA_sept26/tree/master/sesiones/devops/instrucciones_ci).

## 0.1 Crea el proyecto

```bash
npx @nestjs/cli@11 new api-fitval
```

Cuando pregunte por el gestor de paquetes, elige **npm**.

> Escribe `@11` siempre. La versión 12 de NestJS cambia muchas cosas y el código de los apuntes no funciona con ella.

NestJS ya te deja preparado:

- La carpeta `.git`: el repositorio está creado. **No** hagas `git init`.
- El `.gitignore`, con `node_modules`, `dist` y `.env`.
- ESLint y su script `npm run lint`.

```bash
cd api-fitval
npm run lint     # no debe mostrar ningún error (algún "warning" amarillo no importa)
```

## 0.2 Añade el `ci.yml`

Crea la carpeta `.github` (con punto), dentro `workflows` (con **s**), y dentro el fichero `ci.yml`:

```yaml
# CI mínimo: revisa el código en cada push a main y en cada PR hacia main.
# Cada job es un "check" que sale con ✅ o ❌ en GitHub.
name: ci

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  # Check 1: revisa los .html. Esta API no tiene: dirá "Scanned 0 files" y pasará.
  htmlhint:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar el código
        uses: actions/checkout@v7
      - name: Preparar Node.js
        uses: actions/setup-node@v7
        with:
          node-version: 24
      - name: Revisar el HTML con HTMLHint
        run: npx --yes htmlhint@1 "**/*.html" --ignore "dist/**,build/**,coverage/**"

  # Check 2: revisa el código TypeScript con ESLint (tu script "lint")
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar el código
        uses: actions/checkout@v7
      - name: Preparar Node.js
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: npm
      - name: Instalar dependencias
        run: npm ci
      - name: Lint
        run: npm run lint
```

> En YAML los espacios importan. Copia el fichero entero y no muevas nada.

## 0.3 Súbelo a GitHub

1. En GitHub crea un repositorio **público** llamado `api-fitval`, **vacío** (sin README ni `.gitignore`).
2. En tu terminal:

```bash
git add .
git commit -m "Proyecto inicial con CI mínimo"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/api-fitval.git
git push -u origin main
```

3. En la pestaña **Actions** espera a que `htmlhint` y `lint` salgan en verde ✅.

**No sigas hasta verlos en verde.** Si nunca se han ejecutado, no podrás elegirlos en el paso siguiente.

## 0.4 Protege la rama `main`

En tu repositorio:

1. **Settings** → **Rulesets** → **New ruleset** → **New branch ruleset**.
2. **Ruleset Name**: `proteger-main`. **Enforcement status**: `Active`.
3. **Target branches**: **Add target** → **Include default branch**.
4. En **Branch rules**, marca solo estas cuatro casillas:
    - ☑ **Restrict deletions**
    - ☑ **Require a pull request before merging** (no toques sus opciones)
    - ☑ **Require status checks to pass**
    - ☑ **Block force pushes**
5. Debajo de **Require status checks to pass**: **Show additional settings** → **+ Add checks**. Añade `htmlhint` y `lint`.
6. Baja del todo y pulsa **Create**.

### Ejercicio 0

1. Cambia cualquier cosa, haz commit e intenta `git push` directamente a `main`. Copia en `respuestas.md` la línea importante del error.
2. Deshaz el commit local con `git reset --hard origin/main`.

## 0.5 Cómo trabajarás a partir de ahora

Cada parte es **una funcionalidad**, con su propia rama `feature/...` y su propia PR. Cada parte te dice el nombre de la rama.

```bash
git switch main
git pull
git switch -c feature/database-connection    # la rama de la funcionalidad
# ...programas y pruebas...
git add .
git commit -m "Conectar con PostgreSQL"
git push -u origin feature/database-connection
```

En GitHub: **Compare & pull request** → **Create pull request**. Espera a los dos checks ✅ y pulsa **Merge pull request** → **Confirm merge**.

En la **descripción de cada PR** escribe qué rutas has hecho y la **tabla de pruebas** de esa parte con el código que te ha salido a ti.

Si un check sale en rojo ❌: pulsa **Details**, lee el error, arréglalo y haz `git push` en la **misma rama**.

---

# PARTE 1 — Conecta con PostgreSQL

**Rama:** `feature/database-connection`

## 1.1 Crea la base de datos `fitval`

Para esta entrega usarás un contenedor **nuevo**, solo para FitVal. Así no se mezcla con lo que tengas de la clase de SQL.

En la raíz del proyecto (al lado de `package.json`), crea `compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:17                 # PostgreSQL 17
    container_name: postgres-fitval    # nombre del contenedor de esta entrega
    environment:
      POSTGRES_USER: alumno            # usuario
      POSTGRES_PASSWORD: alumno_dev_123
      POSTGRES_DB: fitval              # crea la base de datos fitval al arrancar
    ports:
      - "5433:5432"                    # en tu ordenador, el puerto 5433
    volumes:
      - fitval-data:/var/lib/postgresql/data   # los datos no se pierden al parar

volumes:
  fitval-data:
```

- `container_name: postgres-fitval`: con este nombre lo usarás en los comandos `docker exec`.
- `POSTGRES_DB: fitval`: la base de datos se crea sola la primera vez. No hace falta `CREATE DATABASE`.
- `"5433:5432"`: dentro del contenedor PostgreSQL escucha en el 5432, pero en tu ordenador lo verás en el **5433**. Así no choca con el contenedor `postgres-sql` de la clase, aunque lo tengas arrancado.

Arráncalo desde la raíz del proyecto:

```bash
docker compose up -d
```

Comprueba que está en marcha con `docker ps`: tiene que salir `postgres-fitval`.

En el proyecto, crea la carpeta `database` y dentro `schema.sql`. Copia **todo** este bloque. Son tus tablas y tus datos de la entrega de SQL, con los nombres en inglés:

```sql
-- Base de datos del gimnasio FitVal (la de la entrega de SQL, en inglés)
-- Se puede ejecutar todas las veces que quieras: borra y vuelve a crear todo.

DROP TABLE IF EXISTS users, trainer_specialties, specialties, bookings, classes, trainers, cards, members, plans;

CREATE TABLE plans (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL,
  monthly_price NUMERIC(6,2) NOT NULL CHECK (monthly_price >= 0)
);

CREATE TABLE members (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  dni VARCHAR(9) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(150) UNIQUE NOT NULL,
  joined_at DATE NOT NULL,
  plan_id INTEGER NOT NULL REFERENCES plans(id)
);

CREATE TABLE cards (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code VARCHAR(20) UNIQUE NOT NULL,
  active BOOLEAN NOT NULL,
  member_id INTEGER UNIQUE NOT NULL REFERENCES members(id)
);

CREATE TABLE trainers (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  dni VARCHAR(9) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(150) UNIQUE NOT NULL
);

CREATE TABLE classes (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  activity VARCHAR(50) NOT NULL,
  room VARCHAR(30) NOT NULL,
  starts_at TIMESTAMPTZ NOT NULL,
  spots INTEGER NOT NULL CHECK (spots > 0),
  trainer_id INTEGER NOT NULL REFERENCES trainers(id)
);

CREATE TABLE bookings (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  member_id INTEGER NOT NULL REFERENCES members(id),
  class_id INTEGER NOT NULL REFERENCES classes(id),
  booked_at DATE NOT NULL,
  UNIQUE (member_id, class_id)
);

CREATE TABLE specialties (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE trainer_specialties (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  trainer_id INTEGER NOT NULL REFERENCES trainers(id),
  specialty_id INTEGER NOT NULL REFERENCES specialties(id),
  UNIQUE (trainer_id, specialty_id)
);

-- Datos de prueba (los mismos de la entrega de SQL)

INSERT INTO plans (name, monthly_price) VALUES
  ('Básico', 29.90),
  ('Completo', 44.90),
  ('Estudiante', 24.90),
  ('Mañanas', 19.90);

INSERT INTO members (dni, name, email, joined_at, plan_id) VALUES
  ('11111111A', 'Ana Pérez', 'ana@gym.com', '2026-01-10', 2),
  ('22222222B', 'Luis García', 'luis@gym.com', '2026-02-03', 1),
  ('33333333C', 'Marta López', 'marta@gym.com', '2026-03-15', 3),
  ('44444444D', 'Hugo Sanz', 'hugo@gym.com', '2026-03-20', 2),
  ('55555555E', 'Lucía Ruiz', 'lucia@gym.com', '2026-05-02', 3),
  ('66666666F', 'Pablo Martín', 'pablo@gym.com', '2026-06-11', 1),
  ('77777777G', 'Sara Gil', 'sara@gym.com', '2026-09-01', 2),
  ('88888888H', 'Diego Vidal', 'diego@gym.com', '2026-10-01', 1);

INSERT INTO cards (code, active, member_id) VALUES
  ('TJ-0001', TRUE, 1),
  ('TJ-0002', TRUE, 2),
  ('TJ-0003', TRUE, 3),
  ('TJ-0004', FALSE, 4),
  ('TJ-0005', TRUE, 5),
  ('TJ-0006', TRUE, 6),
  ('TJ-0007', TRUE, 7);

INSERT INTO trainers (dni, name, email) VALUES
  ('90000001J', 'Carla Navarro', 'carla@gym.com'),
  ('90000002K', 'Javier Ortega', 'javier@gym.com'),
  ('90000003L', 'Elena Romero', 'elena@gym.com'),
  ('90000004M', 'Rubén Castro', 'ruben@gym.com');

INSERT INTO classes (activity, room, starts_at, spots, trainer_id) VALUES
  ('Spinning', 'Sala 1', '2026-11-02 09:00', 12, 1),
  ('Yoga', 'Sala 2', '2026-11-02 18:00', 10, 2),
  ('Pilates', 'Sala 2', '2026-11-03 10:00', 8, 3),
  ('Spinning', 'Sala 1', '2026-11-03 19:00', 12, 1),
  ('Zumba', 'Sala 3', '2026-11-04 18:30', 20, 3),
  ('Yoga', 'Sala 2', '2026-11-05 09:30', 10, 2),
  ('Crossfit', 'Sala 3', '2026-11-05 20:00', 3, 1);

INSERT INTO bookings (member_id, class_id, booked_at) VALUES
  (1, 1, '2026-10-28'),
  (1, 3, '2026-10-28'),
  (1, 7, '2026-10-29'),
  (2, 1, '2026-10-29'),
  (2, 2, '2026-10-29'),
  (3, 2, '2026-10-30'),
  (3, 5, '2026-10-30'),
  (4, 1, '2026-10-30'),
  (4, 7, '2026-10-31'),
  (5, 3, '2026-10-31'),
  (5, 5, '2026-10-31'),
  (6, 4, '2026-11-01'),
  (7, 5, '2026-11-01'),
  (7, 7, '2026-11-01');

INSERT INTO specialties (name) VALUES
  ('Ciclo indoor'), ('Yoga'), ('Pilates'), ('Baile'), ('Fuerza');

INSERT INTO trainer_specialties (trainer_id, specialty_id) VALUES
  (1, 1), (1, 5), (2, 2), (2, 3), (3, 3), (3, 4), (4, 5);
```

Ejecútalo. En Git Bash, macOS o Linux:

```bash
docker exec -i postgres-fitval psql -U alumno -d fitval < database/schema.sql
```

En PowerShell:

```powershell
Get-Content database/schema.sql | docker exec -i postgres-fitval psql -U alumno -d fitval
```

La primera vez verás varios `NOTICE: table "..." does not exist, skipping`. Es normal: el `DROP TABLE IF EXISTS` no encuentra nada que borrar.

Comprueba que hay 7 clases y 14 reservas:

```bash
docker exec -it postgres-fitval psql -U alumno -d fitval -c "SELECT COUNT(*) FROM classes;" -c "SELECT COUNT(*) FROM bookings;"
```

**Cuando quieras volver a empezar** (porque tus pruebas han cambiado los datos), ejecuta otra vez `schema.sql`: borra todo y lo vuelve a crear.

## 1.2 Instala las librerías y crea el `.env`

```bash
npm install pg class-validator class-transformer
npm install -D @types/pg
```

Crea `.env` en la raíz:

```env
DATABASE_URL=postgres://alumno:alumno_dev_123@localhost:5433/fitval
```

Y `.env.example`, que **sí** se sube, sin la contraseña real:

```env
DATABASE_URL=postgres://alumno:CAMBIAME@localhost:5433/fitval
```

## 1.3 `src/main.ts`

Sustitúyelo entero. Es el de los apuntes 03a y 03b:

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

process.loadEnvFile(); // lee el .env y guarda sus valores en process.env

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Todas las rutas empiezan por /api
  app.setGlobalPrefix('api');

  // Comprueba los DTO en todas las rutas
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // quita los campos que no estén en el DTO...
      forbidNonWhitelisted: true, // ...o mejor: responde 400 si llegan
      transform: true, // aplica las transformaciones del DTO (como el trim)
    }),
  );

  await app.listen(process.env.PORT ?? 3000);
}
void bootstrap();
```

## 1.4 La conexión: `src/database/database.module.ts`

Es el del apunte 03b con un añadido: si PostgreSQL se para, la API **no se cae**.

```ts
import { Module } from '@nestjs/common';
import { Pool } from 'pg';

// Crea el Pool y avisa (sin caerse) si PostgreSQL corta una conexión
function createPool() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });

  // Si PostgreSQL se para, cierra las conexiones que no se estaban usando.
  // Sin esta línea, ese aviso tumbaría la API entera.
  pool.on('error', (error) => console.error('PostgreSQL:', error.message));

  return pool;
}

@Module({
  providers: [
    {
      provide: Pool, // cuando alguien pida un Pool...
      useFactory: createPool, // ...NestJS ejecuta esta función una vez
    },
  ],
  exports: [Pool], // otros módulos pueden pedir el Pool
})
export class DatabaseModule {}
```

## 1.5 Una ruta para comprobar la conexión

Borra `src/app.service.ts` y `src/app.controller.spec.ts`: no los vamos a usar.

Sustituye `src/app.controller.ts`:

```ts
import { Controller, Get } from '@nestjs/common';
import { Pool } from 'pg';

@Controller()
export class AppController {
  constructor(private readonly pool: Pool) {} // NestJS nos da el Pool

  // GET /api/health: ¿está viva la API? ¿y la base de datos?
  @Get('health')
  async health() {
    const result = await this.pool.query<{ database: string }>(
      'SELECT current_database() AS database',
    );
    return { status: 'ok', database: result.rows[0].database };
  }
}
```

Y `src/app.module.ts`:

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { DatabaseModule } from './database/database.module';

@Module({
  imports: [DatabaseModule], // para que AppController pueda pedir el Pool
  controllers: [AppController],
})
export class AppModule {}
```

Arranca con `npm run start:dev` y pide `GET http://localhost:3000/api/health`:

```json
{ "status": "ok", "database": "fitval" }
```

### Ejercicio 1

Con la API arrancada, para PostgreSQL (`docker compose stop`) y vuelve a pedir `/api/health`. Mira la respuesta y la terminal de NestJS.

**Pregunta 1:** ¿qué código recibe el cliente con PostgreSQL parado? ¿Qué ves en la terminal que el cliente no ve? ¿Por qué es mejor así?

Vuelve a arrancarlo (`docker compose start`). Después: `npm run lint`, commit, push, PR, checks en verde y merge.

---

# PARTE 2 — Leer planes y clases

**Rama:** `feature/read-classes`

## 2.1 Ejemplo resuelto: los planes

La web del gimnasio enseña los planes y su precio. Genera el módulo:

```bash
npx nest g module plans
npx nest g controller plans --no-spec
npx nest g service plans --no-spec
```

`src/plans/plan.ts`:

```ts
// Así llega un plan desde PostgreSQL (una fila)
export interface PlanRow {
  id: number;
  name: string;
  monthly_price: string; // ¡ojo! NUMERIC llega como texto: "29.90"
}

// Así lo devuelve nuestra API
export interface Plan {
  id: number;
  name: string;
  monthlyPrice: number;
}

// Convierte una fila en un plan de la API
export function toPlan(row: PlanRow): Plan {
  return {
    id: row.id,
    name: row.name,
    monthlyPrice: Number(row.monthly_price), // "29.90" → 29.9
  };
}
```

`pg` devuelve las columnas `NUMERIC` como **texto**, para no perder decimales. Por eso `toPlan` lo convierte con `Number(...)`.

`src/plans/plans.service.ts`:

```ts
import { Injectable } from '@nestjs/common';
import { Pool } from 'pg';
import { Plan, PlanRow, toPlan } from './plan';

@Injectable()
export class PlansService {
  constructor(private readonly pool: Pool) {}

  // GET /api/plans: del más barato al más caro
  async findAll(): Promise<Plan[]> {
    const result = await this.pool.query<PlanRow>(
      'SELECT id, name, monthly_price FROM plans ORDER BY monthly_price',
    );
    return result.rows.map(toPlan); // cada fila → un plan de la API
  }
}
```

`src/plans/plans.controller.ts`:

```ts
import { Controller, Get } from '@nestjs/common';
import { PlansService } from './plans.service';

@Controller('plans') // rutas /api/plans
export class PlansController {
  constructor(private readonly plansService: PlansService) {}

  @Get() // GET /api/plans
  findAll() {
    return this.plansService.findAll();
  }
}
```

Y en `src/plans/plans.module.ts`, añade `imports: [DatabaseModule]` (como en el apunte 03b, apartado 5).

Pide `GET /api/plans`. Tienen que salir 4 planes; el primero, `Mañanas` con `"monthlyPrice": 19.9`.

## 2.2 Ahora tú: las clases

La web enseña el horario de clases. Cada clase lleva el **nombre del monitor**, que está en otra tabla: necesitas un `JOIN`. Es tu consulta B4.2 de la entrega de SQL.

Genera el módulo `classes` (como con `plans`) y crea `src/classes/gym-class.ts` con este contenido. Se llama `GymClass` porque `class` es una palabra reservada.

```ts
// Así llega una clase desde PostgreSQL (con el nombre del monitor, del JOIN)
export interface GymClassRow {
  id: number;
  activity: string;
  room: string;
  starts_at: Date;
  spots: number;
  trainer_name: string;
}

// Así la devuelve nuestra API
export interface GymClass {
  id: number;
  activity: string;
  room: string;
  startsAt: Date;
  spots: number;
  trainerName: string;
}

// Convierte una fila en una clase de la API
export function toGymClass(row: GymClassRow): GymClass {
  return {
    id: row.id,
    activity: row.activity,
    room: row.room,
    startsAt: row.starts_at,
    spots: row.spots,
    trainerName: row.trainer_name,
  };
}
```

### Ejercicio 2

Programa en el servicio y el controlador de `classes`:

| Ruta | Qué hace |
|---|---|
| `GET /api/classes` | Todas las clases, ordenadas por fecha (`starts_at`). |
| `GET /api/classes?activity=yoga` | Solo las clases cuya actividad **contiene** ese texto, sin mirar mayúsculas. |
| `GET /api/classes/:id` | Una clase. `400` si el id no es un número, `404` si no existe. |

Pistas:

- La consulta empieza así: `SELECT c.id, c.activity, c.room, c.starts_at, c.spots, t.name AS trainer_name FROM classes c JOIN trainers t ON t.id = c.trainer_id`. El `AS trainer_name` es lo que hace que la columna se llame como en `GymClassRow`.
- El filtro es como el de `author` del apunte 03b (apartado 9): `ILIKE '%' || $1 || '%'`.
- `ParseIntPipe` para el `400`, `NotFoundException` para el `404`.

**Tabla de pruebas:**

| Petición | Resultado esperado |
|---|---|
| `GET /api/classes` | `200`, 7 clases; la primera, Spinning con `"trainerName": "Carla Navarro"` |
| `GET /api/classes?activity=yoga` | `200`, 2 clases |
| `GET /api/classes?activity=SPIN` | `200`, 2 clases |
| `GET /api/classes/7` | `200`, Crossfit con 3 plazas |
| `GET /api/classes/abc` | `400` |
| `GET /api/classes/99` | `404` |

> Las horas salen como `"2026-11-02T09:00:00.000Z"`. La `Z` significa hora UTC. No te preocupes por eso en esta entrega.

## 2.3 Rómpelo a propósito: la inyección SQL

### Ejercicio 3

Cambia **temporalmente** el filtro para pegar el texto dentro del SQL, con comillas invertidas:

```ts
// ❌ MAL, solo para probar
`... WHERE c.activity ILIKE '%${activity}%' ORDER BY c.starts_at`
```

En Postman, pestaña **Params**, pide `GET /api/classes` con la clave `activity` y este valor:

```text
zzz' OR 1=1 --
```

Después vuelve a dejarlo con `$1` y repite la misma petición.

**Pregunta 2:** ¿cuántas clases salen con cada versión? Explica qué ha pasado con la consulta en la versión mala. Pista: escribe la consulta tal y como le llega a PostgreSQL. `--` empieza un comentario en SQL.

---

# PARTE 3 — Crear y borrar clases

**Rama:** `feature/create-delete-class`

De momento cualquiera puede crear y borrar. Lo cerraremos en la Parte 5.

### Ejercicio 4.1: crear una clase

Programa `POST /api/classes`. El cliente envía:

```json
{
  "activity": "Fuerza",
  "room": "Sala 3",
  "startsAt": "2026-11-10T18:00:00Z",
  "spots": 15,
  "trainerId": 4
}
```

Y la API responde `201` con la clase creada, **con el nombre del monitor**:

```json
{
  "id": 8,
  "activity": "Fuerza",
  "room": "Sala 3",
  "startsAt": "2026-11-10T18:00:00.000Z",
  "spots": 15,
  "trainerName": "Rubén Castro"
}
```

Las reglas van en un DTO, `src/classes/dto/create-class.dto.ts` (apunte 03a, apartado 10):

| Campo | Regla | Decoradores |
|---|---|---|
| `activity` | Texto, de 1 a 50 caracteres, sin espacios a los lados | `@Transform` (trim), `@IsString`, `@MinLength`, `@MaxLength` |
| `room` | Texto, de 1 a 30 caracteres, sin espacios a los lados | Los mismos |
| `startsAt` | Fecha y hora en formato ISO | `@IsDateString()` |
| `spots` | Entero, mayor que 0 | `@IsInt`, `@Min` |
| `trainerId` | Entero, y tiene que existir en `trainers` | `@IsInt`, `@Min(1)` y… lo de abajo |

¿Te suenan los números? Son los mismos que pusiste en las tablas: `VARCHAR(50)`, `VARCHAR(30)`, `CHECK (spots > 0)`.

### Ejemplo resuelto: comprobar que el monitor existe

En la entrega de SQL, guardar una clase con un monitor que no existe daba un error de clave ajena. Si dejamos que ese error llegue a la API, el cliente recibe un `500`, como si fuera culpa nuestra. Así que lo comprobamos antes y respondemos `400`:

```ts
// En ClassesService. Añade BadRequestException al import de @nestjs/common
async create(dto: CreateClassDto): Promise<GymClass> {
  // 1. ¿Existe el monitor?
  const trainer = await this.pool.query('SELECT id FROM trainers WHERE id = $1', [
    dto.trainerId,
  ]);
  if (trainer.rows.length === 0) {
    throw new BadRequestException('No existe ningún monitor con ese trainerId'); // → 400
  }

  // 2. Guardamos la clase. RETURNING nos da su id nuevo
  const result = await this.pool.query<{ id: number }>(
    `INSERT INTO classes (activity, room, starts_at, spots, trainer_id)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING id`,
    [dto.activity, dto.room, dto.startsAt, dto.spots, dto.trainerId],
  );

  // 3. Devolvemos la clase completa, con el nombre del monitor
  return this.findOne(result.rows[0].id);
}
```

Fíjate en el paso 3: en vez de escribir otro `JOIN`, **reutilizamos** `findOne`.

Tú escribes el DTO y la ruta en el controlador.

### Ejercicio 4.2: borrar una clase

Programa `DELETE /api/classes/:id`. Devuelve `200` con la clase borrada.

- `404` si la clase no existe.
- `409` si la clase **tiene reservas**: el gimnasio no quiere perder a quién se había apuntado.

Pista: cuenta las reservas con `SELECT COUNT(*)::int AS total FROM bookings WHERE class_id = $1`. El `::int` hace falta porque `COUNT` llega como texto, igual que `NUMERIC`. Para el `409`, `ConflictException`.

**Tabla de pruebas** (en este orden):

| Petición | Resultado esperado |
|---|---|
| `POST` con el body de arriba | `201`, la clase con `"trainerName": "Rubén Castro"` |
| El mismo body con `"spots": 0` | `400` |
| El mismo body con `"trainerId": 99` | `400` |
| El mismo body con `"startsAt": "mañana"` | `400` |
| El mismo body con `"activity": "   "` | `400` |
| El mismo body con un campo de más, `"id": 1` | `400` |
| `DELETE /api/classes/8` (la que has creado) | `200` |
| El mismo `DELETE` otra vez | `404` |
| `DELETE /api/classes/1` | `409` (tiene 3 reservas) |

**Pregunta 3:** PostgreSQL ya tiene una clave ajena en `trainer_id`. ¿Para qué comprobamos en la API que el monitor existe? ¿Qué recibiría el cliente si no lo hiciéramos?

**Pregunta 4:** ¿por qué borrar una clase con reservas es `409` y no `400`?

---

# PARTE 4 — Registro y login

**Rama:** `feature/auth`

Ahora los usuarios de la API. Sigue el apunte [04. Autenticación](../sesiones/backend/04_nestjs_autenticacion.md), apartados 3 a 10, **con estos cambios**:

| En el apunte | En esta entrega |
|---|---|
| Base de datos `bootcamp_api` | `fitval` |
| Rol `customer` (cliente) | `member` (socio) |
| Rol `employee` (empleado) | `staff` (personal del gimnasio) |

## 4.1 La tabla `users`

Añade esto al **final** de `database/schema.sql`:

```sql
-- Usuarios de la API
CREATE TABLE users (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,      -- no puede haber dos iguales
  password_hash TEXT NOT NULL,             -- el hash, nunca la contraseña
  role VARCHAR(20) NOT NULL DEFAULT 'member'
    CHECK (role IN ('member', 'staff'))    -- solo estos dos roles
);
```

Y ejecuta `schema.sql` entero otra vez (apartado 1.1). Así vuelves a tener los datos limpios y la tabla nueva. La línea `DROP TABLE IF EXISTS` de arriba ya incluye `users`.

## 4.2 El código

Del apunte 04 necesitas:

```bash
npm install argon2 @nestjs/jwt@11
```

- `src/users/`: `user.ts`, `users.service.ts` y `users.module.ts` (apartado 5).
- `src/auth/`: los DTO, `jwt-payload.ts`, `auth.module.ts`, `auth.service.ts` y `auth.controller.ts` (apartados 7 a 9).
- `src/auth/auth.guard.ts` y la ruta `GET /api/auth/me` (apartado 10).
- `JWT_SECRET` en el `.env`, y también en el `.env.example` con el valor `CAMBIAME`.

Recuerda: en el token va **solo** `{ sub: id }`.

**Tabla de pruebas:**

| Petición | Resultado esperado |
|---|---|
| `POST /api/auth/register` con `{ "email": "ana@gym.com", "password": "12345678" }` | `201`, con `"role": "member"` y **sin** contraseña ni hash |
| La misma otra vez | `409` |
| `"email": "ana"` | `400` |
| `"password": "123"` | `400` |
| `POST /api/auth/login` con los datos de Ana | `200`, con `accessToken` |
| Login con una contraseña mal | `401` |
| Login con un correo que no existe | `401`, con el **mismo** mensaje |
| `GET /api/auth/me` con el token de Ana | `200`, sus datos |
| `GET /api/auth/me` sin token | `401` |
| `GET /api/auth/me` cambiando la última letra del token | `401` |

**Pregunta 5:** pega el token de Ana en [jwt.io](https://jwt.io). ¿Qué datos lleva dentro? ¿Por qué no metemos también el rol?

---

# PARTE 5 — Solo el personal crea y borra clases

**Rama:** `feature/staff-guard`

Los socios pueden **ver** las clases. Solo el **personal** (`staff`) puede **crearlas y borrarlas**.

### Ejercicio 5

1. Crea `src/auth/staff.guard.ts`. Es el `EmployeeGuard` del apunte 04 (apartado 11), cambiando `'employee'` por `'staff'` y el mensaje por `'Solo para el personal del gimnasio'`.
2. Pon `@UseGuards(AuthGuard, StaffGuard)` en el `POST` y el `DELETE` de `classes.controller.ts`. Los `GET` se quedan sin guard.
3. `ClassesModule` tiene que importar `UsersModule` (el `StaffGuard` necesita el `UsersService`).

Para las pruebas necesitas un usuario de recepción. Regístralo con la API (`recepcion@fitval.com`, contraseña `recepcion123`) y haz login para tener su token. Todavía es `member`.

**Tabla de pruebas** (en este orden). El body es el del Ejercicio 4.1:

| Petición | Resultado esperado |
|---|---|
| `GET /api/classes`, sin token | `200` |
| `POST /api/classes`, sin token | `401` |
| `POST /api/classes`, token de Ana | `403` |
| `POST /api/classes`, token de recepción | `403` (todavía es `member`) |

Ahora hazla `staff` desde `psql`, **sin volver a hacer login**:

```bash
docker exec -it postgres-fitval psql -U alumno -d fitval -c "UPDATE users SET role = 'staff' WHERE email = 'recepcion@fitval.com';"
```

| Petición | Resultado esperado |
|---|---|
| `POST /api/classes`, el **mismo** token de recepción | `201` |
| `DELETE /api/classes/1`, token de recepción | `409` (tiene reservas) |
| `DELETE /api/classes/6`, token de Ana | `403` |
| `DELETE /api/classes/6`, token de recepción | `200` (el Yoga del jueves no tiene reservas) |

**Pregunta 6:** el token de recepción se creó cuando todavía era `member`. ¿Por qué funciona ahora sin volver a hacer login? ¿Qué pasaría si el rol fuera dentro del token?

---

## Entrega

Entrega en Moodle la **URL del repositorio** `api-fitval`. En el repositorio tiene que haber:

- [ ] `.github/workflows/ci.yml` y la rama `main` protegida (Parte 0).
- [ ] Una rama `feature/...` y una PR **fusionada** por cada parte (1 a 5), con los checks en verde y la tabla de pruebas en la descripción.
- [ ] `compose.yaml` y `database/schema.sql` con la tabla `users`.
- [ ] `.env.example` subido y `.env` **no** subido.
- [ ] `respuestas.md` con la línea del Ejercicio 0 y las preguntas 1 a 6 (una o dos frases cada una).

Se valora como **Apto** si: la API arranca con `npm run start:dev`, las rutas responden los códigos de las tablas de pruebas, el CI está en verde y las respuestas están explicadas con tus palabras. Si algo no funciona, explica en la PR qué has probado.

## Si algo falla

- **`ECONNREFUSED 127.0.0.1:5433`**: PostgreSQL no está arrancado. Desde la raíz del proyecto, `docker compose up -d`.
- **`ECONNREFUSED 127.0.0.1:5432`**: tu `DATABASE_URL` usa el puerto 5432. El de esta entrega es el **5433**.
- **`Bind for 0.0.0.0:5433 failed: port is already allocated`**: otro programa usa el puerto 5433. Cambia el primer número de `ports` (por ejemplo `"5434:5432"`) y también el puerto de `DATABASE_URL`.
- **`password authentication failed`** o **`database "fitval" does not exist`**: revisa `DATABASE_URL` en `.env` y que hayas creado la base de datos (apartado 1.1).
- **`relation "classes" does not exist`**: no has ejecutado `schema.sql` en `fitval`.
- **`ENOENT: no such file or directory, open '.env'`**: falta el `.env`, o no está en la raíz del proyecto.
- **`Nest can't resolve dependencies of the ...Service (?)`**: a ese módulo le falta `imports: [DatabaseModule]`.
- **`Nest can't resolve dependencies of the StaffGuard (?)`**: `ClassesModule` no importa `UsersModule`, o `UsersModule` no tiene `exports: [UsersService]`.
- **`trainerName` sale `undefined`**: en la consulta falta `AS trainer_name`.
- **`monthlyPrice` sale entre comillas (`"29.90"`)**: falta el `Number(...)` en `toPlan`.
- **`secretOrPrivateKey must have a value`**: falta `JWT_SECRET` en el `.env`. Para y arranca otra vez.
- **Siempre `401` aunque el token es bueno**: la cabecera tiene que ser exactamente `Authorization: Bearer <token>`. En Postman, pestaña **Authorization → Bearer Token**.
- **El check `lint` sale en rojo**: pulsa **Details** y lee qué regla y qué línea. Pasa `npm run lint` en tu ordenador hasta que no salga ningún error.
- **`npm ci` falla en el CI**: no has subido `package-lock.json`, o instalaste algo y no subiste el nuevo.
- **Tus resultados no coinciden con las tablas**: tus pruebas han cambiado los datos. Ejecuta otra vez `schema.sql`. Después tendrás que registrar de nuevo a los usuarios.

## Retos opcionales

Para quien termine antes. Cada uno en su rama `feature/...` y con su PR.

1. **Reservar plaza** (`feature/bookings`). `POST /api/classes/:id/bookings`, con `AuthGuard`. El socio que reserva es el de la tabla `members` con **el mismo correo** que el usuario (Ana se registró con `ana@gym.com`, que es la socia 1). Responde:
    - `201` con la reserva (`booked_at` es la fecha de hoy: `CURRENT_DATE`).
    - `404` si la clase no existe.
    - `403` si el usuario no es socio (por ejemplo, recepción).
    - `409` si la clase está llena (prueba con el Crossfit, la 7).
    - `409` si ya tiene plaza. Pista: el `UNIQUE (member_id, class_id)` hace fallar el `INSERT` con el código `23505`. Captúralo con `try/catch`.
2. **Socios por plan** (`feature/plan-members`). Añade a cada plan de `GET /api/plans` el campo `memberCount`. Es tu consulta B4.5. Resultado: Básico 3, Completo 3, Estudiante 2, Mañanas 0.
3. **Compilar en el CI** (`feature/ci-build`). Añade al job `lint` del `ci.yml` un paso que ejecute `npm run build`. Rompe un tipo a propósito en una rama y mira cómo el check sale en rojo.

## Documentación para consultar

- [NestJS: controladores](https://docs.nestjs.com/controllers)
- [NestJS: guards](https://docs.nestjs.com/guards)
- [NestJS: validación](https://docs.nestjs.com/techniques/validation)
- [class-validator: lista de decoradores](https://github.com/typestack/class-validator#validation-decorators)
- [node-postgres (`pg`): consultas](https://node-postgres.com/features/queries)
- [PostgreSQL: códigos de error](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [jwt.io: introducción a JWT](https://jwt.io/introduction)
- [MDN: códigos de estado HTTP](https://developer.mozilla.org/es/docs/Web/HTTP/Reference/Status)
- [Instrucciones del CI mínimo (con capturas)](https://github.com/TheBridge-FullStackDeveloper/temario_fullstack_FT_LABORA_sept26/tree/master/sesiones/devops/instrucciones_ci)

Si usas IA para resolver dudas, pídele que te **explique** el error o te dé una **pista**, no el código entero. En la corrección tendrás que saber explicar cada comprobación que hace tu código.
