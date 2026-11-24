# @safari/db: la capa de datos

`@safari/db` es la única puerta de la API a PostgreSQL. Expone un cliente de
Prisma 7 (con `@prisma/adapter-pg`) y un repositorio de **funciones planas**
por agregado: `listTags()`, `createCoupon()`, `findOrderByIdOrTracking()`…
La API NestJS de `apps/api/rest` lo importa como cualquier dependencia.

**Contenido**

1. [Qué es y por qué existe](#qué-es-y-por-qué-existe)
2. [Arranque rápido](#arranque-rápido)
3. [Estructura del paquete](#estructura-del-paquete)
4. [Cómo lo usa la API](#cómo-lo-usa-la-api)
5. [El esquema: `db/schema.sql` manda, Prisma no migra](#el-esquema-dbschemasql-manda-prisma-no-migra)
6. [Convenciones](#convenciones)
7. [Errores de dominio](#errores-de-dominio)
8. [Cómo agregar una tabla o un repositorio](#cómo-agregar-una-tabla-o-un-repositorio)
9. [Cómo se prueba](#cómo-se-prueba)
10. [Referencia de repositorios](#referencia-de-repositorios)
11. [Por qué el paquete se construye](#por-qué-el-paquete-se-construye)
12. [Problemas conocidos](#problemas-conocidos)

---

## Qué es y por qué existe

- **Un solo lugar para el SQL.** Los servicios de Nest no hablan con Prisma directamente: llaman funciones de este paquete. Así las reglas de la base (CHECK, índices, transacciones) viven junto a las consultas que las usan.
- **Frontera JSON-safe.** Los repositorios no devuelven filas crudas de Prisma, sino *records* (`src/records.ts`) en camelCase, sin `BigInt` ni `Decimal`. La API los traduce al snake_case que publica.
- **Errores con significado.** Los fallos de escritura salen como errores de dominio con un código cerrado (400, 404 o 409), no como códigos `P2xxx` de Prisma.
- **Autónomo.** Usa npm, no forma parte del workspace de yarn de `apps/`. Se construye con tsup a `dist/` y la API lo enlaza con `link:`.

El scraper Python (`services/scraper-worker`) **no** usa este paquete: escribe con SQL directo (psycopg) en la misma tabla `products`.

---

## Arranque rápido

**Requisitos:** la base corriendo y sembrada (`just db-up` en la raíz) y
`DATABASE_URL` definida. Los tests la leen del `.env` de `packages/db`; cópialo
de `.env.example` (`just setup` no lo crea) o exporta la variable.

Desde la raíz del repositorio:

```bash
just db-build      # npm install + npm run build (prisma generate + tsup -> dist/)
just db-check      # npm run typecheck + npm test (requiere just db-up)
```

O dentro de `packages/db`:

```bash
npm install         # dependencias propias (no forma parte de apps/)
npm run build       # prisma generate + tsup -> dist/ (CJS + .d.ts)
npm run dev         # tsup --watch: reconstruye dist/ al guardar
npm run typecheck   # tsc --noEmit
npm run lint        # biome check .
npm run format      # biome check --write . (reescribe archivos)
npm test            # vitest run: integración contra el Postgres local
npm run generate    # solo prisma generate -> generated/
```

`dist/` y `generated/` están en `.gitignore`: tras clonar hay que construir.
No confíes en el script `prepare` para eso; ver
[Problemas conocidos](#problemas-conocidos).

---

## Estructura del paquete

```text
packages/db/
├── index.ts                 barrel explícito: el único punto de entrada
├── prisma/schema.prisma     espejo de la base (prisma db pull), revisado a mano
├── prisma.config.ts         ruta del schema y DATABASE_URL (con dotenv)
├── generated/               cliente de Prisma (gitignored; npm run generate)
├── dist/                    build de tsup (gitignored; npm run build)
├── tsup.config.ts           build CJS + .d.ts, con shims (ver más abajo)
├── vitest.config.ts         tests en serie (fileParallelism: false)
├── biome.json               lint y formato
└── src/
    ├── client.ts            singleton lazy de PrismaClient + setPrismaLogSink()
    ├── clock.ts             now() y _setNowProvider() (reloj de los timestamps)
    ├── domain-errors.ts     errores de dominio y translateCatalogWriteError()
    ├── errors.ts            helpers para clasificar errores de Prisma
    ├── health.ts            pingDatabase() para /health
    ├── pagination.ts        buildPaginator(): el envoltorio de paginación
    ├── records.ts           tipos *Record y sus mappers _to*Record
    ├── slug.ts              generateSlug() y normalizeSlug()
    └── repositories/        un *.repository.ts por agregado (34)
                             y sus *.integration.test.ts
```

Piezas de `src/` que conviene conocer:

| Archivo | Qué hace |
|---|---|
| `client.ts` | Crea el `PrismaClient` en el **primer acceso** a `prisma`, no al importar (un `Proxy`). Si falta `DATABASE_URL`, lanza un error claro en ese momento. En `NODE_ENV=development` imprime las consultas. `setPrismaLogSink()` manda errores y warnings de Prisma a otro logger (la API los manda al `Logger` de Nest en `main.ts`). |
| `health.ts` | `pingDatabase(timeoutMs = 2000)` hace `SELECT 1` con timeout y nunca lanza: devuelve `{ ok, latencyMs, error? }`. Lo usa `apps/api/rest/src/health/health.service.ts`. |
| `slug.ts` | `generateSlug()` normaliza con `SELECT slugify($1)` de Postgres (una sola fuente de la regla) y añade sufijos `-2`, `-3`… si el slug ya existe. Cada repositorio le pasa su propia función de búsqueda (`ExistingSlugLookup`), así el nombre de la tabla nunca llega al SQL. Un texto que normaliza a vacío da `EmptySlugError`. |
| `errors.ts` | `parsePrismaError`, `isPrismaConnectionError`, `getUserFriendlyMessage`… Son helpers de lectura. Para escrituras se usa `translateCatalogWriteError` de `domain-errors.ts`, que sí distingue P2002, P2003 y P2025. |

---

## Cómo lo usa la API

`apps/api/rest/package.json` declara:

```json
"@safari/db": "link:../../../packages/db"
```

`link:` crea un enlace simbólico: `apps/api/rest/node_modules/@safari/db`
apunta a `packages/db`. La API carga **`dist/`**, no el código fuente. Por eso:

- Tras cambiar algo en `packages/db`, corre `npm run build` (o deja
  `npm run dev` abierto). No hace falta reinstalar la API.
- `DATABASE_URL` debe estar en el `.env` de la API. `just setup` la añade si
  falta.

Un servicio de Nest típico importa las funciones del barrel, traduce
camelCase a snake_case y convierte los errores de dominio en HTTP:

```ts
import { createTag, listTags, type TagRecord } from '@safari/db';
import { toWriteHttpException } from '#src/common/errors/domain-error.mapper.js';

async create(dto: CreateTagDto) {
  try {
    const tag = await createTag({ name: dto.name, typeId: dto.type_id });
    return toTagResponse(tag); // camelCase -> snake_case
  } catch (error) {
    throw toWriteHttpException(error); // 400 / 404 / 409 / 503 / 500
  }
}
```

`toWriteHttpException` vive en
`apps/api/rest/src/common/errors/domain-error.mapper.ts`. Es el único sitio
que traduce los códigos de dominio a HTTP, y un agregado nuevo no lo modifica.

En los tests de la API, 37 de los 44 specs mockean `@safari/db` con
`vi.mock('@safari/db', …)`, así que no necesitan Postgres.

---

## El esquema: `db/schema.sql` manda, Prisma no migra

La base la crea y la siembra `just db-migrate` (lo llama `just db-up`)
aplicando `db/schema.sql` y `db/seed.sql`. Ese SQL es el único dueño del DDL.

- Este paquete **no tiene `prisma/migrations/`, a propósito.** Dos dueños del
  DDL terminan en drift.
- `schema.prisma` es un **espejo introspectado** con `prisma db pull` y luego
  revisado a mano: modelos en PascalCase, campos en camelCase con `@map`,
  relaciones con nombre.
- **No uses `prisma migrate dev` ni `prisma db push`** contra esta base.

Si cambia `db/schema.sql`:

```bash
just db-reset                  # recrea la base con el DDL nuevo
cd packages/db
npx prisma db pull             # OJO: pisa los renombres manuales
# revisa el diff y vuelve a aplicar los renombres y los @map, luego:
npm run generate && npm test
```

### Lo que Prisma no modela

La cabecera de `prisma/schema.prisma` lista el detalle. En resumen:

| Qué | Cómo se cubre |
|---|---|
| **CHECK constraints** (`products_rebaja_valida`, `coupons_vigencia_valida`, `reviews_rating_check`, los de `orders`…; la lista completa está en la cabecera de `schema.prisma`) | Prisma no los conoce. El repositorio valida **antes de escribir** y lanza un error de dominio (ver [Errores de dominio](#errores-de-dominio)). |
| **Índice único parcial** `products_procedencia_key` (`(source_store, source_product_id) WHERE source_store IS NOT NULL`) | Sí está en el schema, gracias al preview `partialIndexes`. **No quites ese preview**: sin él, Prisma trataría el unique como total y `prisma validate` acusaría drift. `upsertScrapedProduct` opera sobre esa clave. |
| **Índices únicos de expresión** `users_email_lower_idx` (`lower(email)`) y `reviews_autor_producto_orden_key` (`COALESCE(order_id, 0)`) | Invisibles para Prisma. Por eso `User.email` no lleva `@unique`, la búsqueda por email usa `$queryRaw` (ver «Identidad y acceso» en la [Referencia de repositorios](#referencia-de-repositorios)) y los P2002 que disparan se traducen a mano. |
| **Índices trigram** `products_nombre_ilike_trgm_idx` y `questions_question_ilike_trgm_idx` | Sí están modelados, como `@@index(..., type: Gin)` sobre la columna. Así el `ILIKE` que genera `contains` + `mode: 'insensitive'` los usa. |

---

## Convenciones

### Records: camelCase, JSON-safe

Los repositorios devuelven tipos `*Record` de `src/records.ts`, construidos
por mappers internos `_to*Record`:

| Tipo en Postgres | En el record | Por qué |
|---|---|---|
| `bigserial` (ids) | `number` | `JSON.stringify` falla con `BigInt`, y los ids reales están lejos de `MAX_SAFE_INTEGER`. |
| `numeric(12,2)` (precios, importes) | `number` | El frontend espera `price: 40.5`, como lo servía el mock. Con 12 dígitos y 2 decimales un double no pierde precisión práctica. Si algún día hace falta aritmética exacta, el único punto de cambio es `_dec()` en `records.ts`. |
| `timestamptz` | `Date` | Nest lo serializa a ISO-8601 solo. |
| `json` / `jsonb` | `Prisma.JsonValue` | Con columnas `json`, las claves salen en el orden del texto guardado. |

**snake_case solo en la API.** La capa de datos habla camelCase; la
traducción al contrato HTTP (snake_case) vive en los servicios de Nest.

**Excepción de escritura: dinero exacto.** Las escrituras de órdenes reciben
el dinero como **texto decimal** (`"0.044"`) para que llegue exacto a
`numeric`. Para ese cálculo el barrel exporta `Decimal`, el de Prisma.

### Listados: `{ items, total }`

Los listados paginados devuelven `{ items, total }`; el que llama arma el
envoltorio. Son paginación por offset, con `page` 1-based y un `limit` por
defecto que depende del repositorio (30 en la mayoría).

```ts
const { items, total } = await listProducts({ typeSlug: 'gadget', page, limit });
return buildPaginator({ data: items, total, page, limit, baseUrl });
```

`buildPaginator()` reproduce exactamente la aritmética y las claves de
`apps/api/rest/src/common/pagination/paginate.ts` (`total`, `current_page`,
`count`, `last_page`, `per_page`, `firstItem`, `lastItem`, `*_page_url`),
incluida su rareza: `prev_page_url` apunta a la página actual. Las
`*_page_url` son `null` si no se pasa `baseUrl` (`APP_URL` es configuración
de la app, no de la capa de datos). Hoy la mayoría de los servicios usan el
`paginate()` de la API y dos (`shops` y `users`) usan `buildPaginator`; dan el
mismo contrato.

Algunos listados no paginan y devuelven un array: `listTaxes`,
`listShippings`, `listAttributes`, `listTopAuthors`, `listStaffShops` y las
listas del dashboard.

### Guardas de id

Antes de escribir, los repositorios validan las referencias con
**`Number.isSafeInteger`**, no `Number.isInteger`:

- `Number.isInteger(1e21)` es `true`, pero ese valor llega a Postgres como
  `bigint` inválido y el error no trae `.code` de Prisma: acabaría en un 500.
- Con la guarda, un id mal formado (`"abc"`, `NaN`, `1.5`, `1e21`) da
  `InvalidReferenceError` (400) con el nombre del campo, sin tocar la base.
- `null` sigue siendo legal donde la FK lo admite (limpia la referencia).
- Un id cero o negativo **no** se rechaza en la guarda: llega a Postgres, la
  FK falla (P2003) y se traduce al mismo 400.

Ejemplo: `_assertValidTypeId` en `src/repositories/tags.repository.ts`.

### Reloj de los timestamps

Spec viva: [`data-layer-clock-policy`](../../openspec/specs/data-layer-clock-policy/spec.md).

- **Ninguna tabla tiene trigger de `updated_at`.** Lo fija la capa de datos
  con `now()` de `src/clock.ts`: cada `update`/`upsert` escribe
  `updatedAt: now()`. El final de `db/schema.sql` lo explica.
- **`created_at` sale de `@default(now())` de Prisma**, que usa el mismo
  reloj de Node. Una fila, un reloj: así `updated_at >= created_at` siempre.
  Con triggers, el reloj de Postgres podía ir cientos de milisegundos por
  detrás del de Node y una fila salía con `updated_at` anterior a
  `created_at`.
- Otros instantes de dominio también salen de `now()`: `consumedAt` de los
  tokens, `scrapedAt` de productos, los `tracking_number` de órdenes, los
  `transaction_identifier` de transferencias.
- En tests, `_setNowProvider(() => fecha)` fija `now()`. No afecta a
  `created_at`.
- **Tabla nueva = misma regla.** No añadas triggers; fija `updatedAt: now()`.

### SQL crudo

Se usa `$queryRaw` (tagged template: cada `${valor}` viaja como parámetro,
nunca concatenado) solo donde Prisma no alcanza:

| Repositorio | Para qué |
|---|---|
| `users` | `WHERE lower(email) = lower($1)`, el único que usa `users_email_lower_idx`. |
| `conversations` | `DISTINCT ON` para el último mensaje de cada conversación de la página. |
| `analytics` | `count(*) FILTER (...)` y agrupación por mes UTC. |
| `ownership-transfers`, `flash-sale-requests` | `SELECT … FOR UPDATE` para bloquear filas dentro de una transacción. |
| `slug.ts`, `health.ts` | `SELECT slugify($1)` y `SELECT 1`. |

`$queryRawUnsafe` no se usa en el código del paquete (solo en dos tests).

### Transacciones

`$transaction` aparece en 12 repositorios: `attributes`, `auth-tokens`,
`conversations`, `flash-sale`, `flash-sale-requests`, `orders`,
`ownership-transfers`, `questions`, `reviews`, `staff`, `store-notices` y
`users`. Cuando basta con un **nested write** (padre e hijos en un solo
`create`), se prefiere ese: Prisma lo ejecuta en una transacción y una FK
inválida no deja filas sueltas.

---

## Errores de dominio

Spec viva: [`catalog-write-foundations`](../../openspec/specs/catalog-write-foundations/spec.md).

`src/domain-errors.ts` define un conjunto **cerrado** de cinco códigos. Todas
las clases extienden `CatalogWriteError` y reciben el agregado y el campo por
parámetro, así que un agregado nuevo no añade clases ni toca este archivo.

| Clase | `code` | HTTP | Cuándo |
|---|---|---|---|
| `EmptySlugError` | `CATALOG_EMPTY_SLUG` | 400 | El nombre (o el slug) normaliza a vacío. |
| `InvalidReferenceError` | `CATALOG_INVALID_REFERENCE` | 400 | Una FK apunta a algo que no existe (P2003) o el id está mal formado. |
| `InvalidFieldValueError` | `CATALOG_INVALID_REFERENCE` | 400 | Un valor viola un CHECK de valor simple. El repositorio lo lanza antes de escribir; el mensaje nombra el CHECK real. |
| `RecordNotFoundError` | `CATALOG_RECORD_NOT_FOUND` | 404 | Actualizar o borrar algo que no existe (P2025). |
| `DependentRowsError` | `CATALOG_DEPENDENT_ROWS` | 409 | Borrar dejaría filas huérfanas. |
| `SlugConflictError` | `CATALOG_SLUG_CONFLICT` | 409 | **Cualquier** clave única repetida (P2002), sea slug o no: el `code` de un cupón, el único de tres columnas de reseñas… El mensaje nombra el campo, no el valor. |

Cómo se usan:

- **`translateCatalogWriteError(error, { aggregate, id?, uniqueField? })`**
  convierte P2002, P2003 y P2025 en la clase que toca y devuelve intacto
  cualquier otro error. El repositorio lo llama en su `catch` y lo lanza.
- Con adapter-pg, `meta.target` del P2002 no llega. Por eso el campo sale de
  `uniqueField` o, si falta, se asume `'slug'`.
- **`isCatalogWriteError(error)`** compara por `code`, no por `instanceof`: un
  mock del barrel o dos copias del build crean clases distintas para el mismo
  error.

Errores propios de un repositorio, para reglas que cruzan varios campos o
estados:

| Error | Dónde | Código que reutiliza |
|---|---|---|
| `InvalidSalePriceError`, `MissingPriceError`, `IncompleteProvenanceError` | `products` | Traducen los CHECK multicampo de `products`. |
| `DuplicateEmailError` | `users` | Email repetido (el índice de expresión). |
| `InsufficientStockError` | `orders` | 409. |
| `SameOwnerError` | `ownership-transfers` | 409. |
| `InvalidFlashSaleRequestError` | `flash-sale-requests` | 400. |
| `FlashSaleRequestStateError` | `flash-sale-requests` | 409. |

Los que reutilizan un código de la tabla de arriba pasan por
`toWriteHttpException` sin cambios en la API.

---

## Cómo agregar una tabla o un repositorio

Toma como plantilla `src/repositories/tags.repository.ts` y su test: es corto
y tiene lectura, escritura, slug, guarda de id y reloj.

1. **DDL en `db/schema.sql`.** Con `IF NOT EXISTS`, sin trigger de
   `updated_at`, y CHECK con nombre (lo cita el error de dominio). Si la tabla
   tiene datos de referencia, van en el seed (`db/README.md` explica cómo se
   genera).
2. **Recrea la base:** `just db-reset`. `db-migrate` no altera tablas
   existentes.
3. **Espejo de Prisma:** `npx prisma db pull`, y revisa el diff. Renombra el
   modelo a PascalCase y los campos a camelCase con `@map`; vuelve a aplicar
   los renombres que el pull haya pisado en otros modelos. Añade los CHECK
   nuevos a la cabecera de `schema.prisma`. Luego `npm run generate`.
4. **Record en `src/records.ts`:** el tipo `XxxRecord` y su mapper
   `_toXxxRecord` (ids y `numeric` con `Number`/`_dec`).
5. **Repositorio en `src/repositories/xxx.repository.ts`:** funciones planas
   que usan `prisma` de `../client`.
   - Listados: `{ items, total }` y un `orderBy` determinista (con desempate
     por `id`).
   - Escrituras: guardas (`Number.isSafeInteger`, CHECK) antes de escribir,
     `updatedAt: now()` en los updates y `translateCatalogWriteError` en el
     `catch`.
   - Si exportas los valores admitidos de un CHECK, hazlo como constante
     (`SHIPPING_TYPES`, `COUPON_TYPES`…).
6. **Exporta en `index.ts`.** El barrel es explícito: lo que no esté ahí no
   existe para la API.
7. **Test de integración** `xxx.integration.test.ts` (ver
   [Cómo se prueba](#cómo-se-prueba)).
8. **Build y gate:** `npm run build`, `npm run lint`, `npm run typecheck` y
   `npm test`. Después, en la API, el servicio que lo consuma.

---

## Cómo se prueba

- **38 archivos de test, 499 tests** (contados con `npx vitest list` el
  2026-11-21). Todos son de integración contra el Postgres real salvo
  `products.repository.test.ts`, que prueba `_toProductRecord` sin base.
- **Requisitos:** `just db-up` (base sembrada) y `DATABASE_URL`. Cada test
  carga `dotenv/config`, que lee el `.env` del directorio donde corres.
- **En serie.** `vitest.config.ts` pone `fileParallelism: false`: los archivos
  comparten una sola base y algunos tests cuentan filas absolutas, que otro
  archivo en paralelo alteraría.
- **Filas centinela.** Las escrituras de cada archivo usan un prefijo propio
  (`zz-tags-`, `zz-types-`…) y se borran en `beforeAll` (por si una corrida
  anterior se cortó) y en `afterAll`.
- **Reloj fijo.** Los tests de `updatedAt` usan `_setNowProvider`.
- En el gate del repo (`just check`) van tres pasos de este paquete: build,
  lint y typecheck al principio, y los tests casi al final. En CI, la variable
  `VITEST_JSON_OUT` hace que vitest escriba además un JSON con los conteos.

---

## Referencia de repositorios

Todas las funciones se importan de `@safari/db`. Esta tabla es el mapa; el
detalle de cada uno está en los bloques de abajo y en la cabecera de su
archivo.

| Área | Repositorios |
|---|---|
| Catálogo | `products`, `categories`, `types`, `tags`, `manufacturers`, `attributes`, `authors` |
| Tiendas | `shops`, `staff`, `ownership-transfers`, `withdraws`, `become-seller` |
| Configuración | `settings`, `taxes`, `shippings`, `order-statuses` |
| Contenido | `faqs`, `terms-and-conditions`, `refund-policies`, `refund-reasons`, `store-notices` |
| Ventas | `coupons`, `flash-sale`, `flash-sale-requests`, `orders`, `order-files`, `wishlists` |
| Comunidad | `reviews`, `abusive-reports`, `questions`, `conversations` |
| Identidad y acceso | `users`, `auth-tokens` |
| Dashboard | `analytics` |

<details>
<summary><strong>Catálogo</strong>: products, categories, types, tags, manufacturers, attributes, authors</summary>

**`products.repository.ts`**

- `listProducts` (filtros, paginado, `limit` por defecto `DEFAULT_PAGE_SIZE`
  = 30), `findProductBySlug`, `findProductShopId`.
- `createProduct`, `updateProduct`, `deleteProduct`.
- `upsertScrapedProduct` y `deleteScrapedProduct`: escritura idempotente
  sobre la clave de procedencia (`products_procedencia_key`). Hoy el scraper
  Python no los usa; existen para una ingesta desde Node o para herramientas
  de administración.
- Valida los CHECK de varios campos y los traduce a `InvalidSalePriceError`
  (`salePrice < price`), `MissingPriceError` (un producto simple necesita
  precio) e `IncompleteProvenanceError` (`source_store` y
  `source_product_id` van juntos).
- `PRODUCT_INCLUDE` y `_toProductRecord` se exportan del archivo, **no** del
  barrel: los reutilizan otros repositorios (wishlists, reviews, flash-sale…)
  para devolver el mismo `ProductRecord`.
- `ProductRecord.inFlashSale` es `true` si el producto tiene alguna fila en
  `flash_sale_products` (un `_count` en `PRODUCT_INCLUDE`). No mira la
  vigencia de la venta.

**`categories.repository.ts`**: `getCategoryTree`, `listCategories`,
`findCategoryByIdOrSlug`, `createCategory`, `updateCategory`,
`deleteCategory`. Las escrituras validan la arista madre-hija: no se permiten
ciclos, y una categoría con hijas no puede cambiar a un `type` distinto del de
ellas (en la práctica, el `type` solo cambia en una hoja).

**`types`, `tags`, `manufacturers`**: `list*`, `find*BySlug`, `create*`,
`update*`, `delete*`. `manufacturers` añade
`findOrCreateManufacturerBySlug`. En todos, el slug es inmutable al
actualizar. `deleteTag` no cuenta dependientes: `product_tag` se borra en
cascada.

**`attributes.repository.ts`**: `AttributeRecord` trae `values` en `id ASC`.

- `listAttributes`: array sin paginar en `id DESC`, filtros por `shopId` y
  nombre.
- `findAttributeByIdOrSlug`: por id si el parámetro es un entero seguro, por
  slug si no.
- `createAttribute`: un único nested write (padre y valores). Una tienda
  inexistente da `InvalidReferenceError` en `shop_id` sin dejar valores
  sueltos; el campo se nombra a mano porque en un nested write el adaptador
  no lo informa.
- `updateAttribute`: reconcilia los valores con el body en un
  `$transaction`. Un id del atributo se actualiza; un id nulo, ausente o no
  entero se inserta; un valor que no aparece en el body se borra. Un id de
  **otro** atributo da `InvalidReferenceError`. Los slugs existentes son
  inmutables; los nuevos salen de `generateSlug` con un lookup en memoria,
  calculado antes de la transacción.
- `deleteAttribute`: devuelve el atributo con sus valores; Postgres los borra
  en cascada.

**`authors.repository.ts`**: `AuthorRecord` lleva `socials`, `image` y
`coverImage` como `JsonValue`, `born`/`death` como `Date` y `productsCount`
como dato sembrado (no hay `products.author_id`).

- `listAuthors({ name, isApproved, page, limit })`: `id DESC`, `limit` 30;
  `name` es ILIKE sobre `name` o `slug`.
- `listTopAuthors(limit = 10)`: `productsCount DESC, id DESC`, sin paginar.
- `findAuthorBySlug`, `createAuthor` (`isApproved` por defecto `true`),
  `updateAuthor` (slug inmutable; un `name` que normaliza a vacío da
  `EmptySlugError`), `deleteAuthor` (nada referencia a `authors`).

</details>

<details>
<summary><strong>Tiendas</strong>: shops, staff, ownership-transfers, withdraws, become-seller</summary>

**`shops.repository.ts`**: `listShops`, `listShopsNear`, `findShopBySlug`,
`findShopOwnerById`, `findOrCreateShopBySlug`, `createShop`, `updateShop`,
`setShopActive` e `isShopStaff(userId, shopId)`. Esta última dice si hay fila
en `shop_staff`; ser dueño no cuenta, y con ids no válidos devuelve `false`
sin lanzar.

**`staff.repository.ts`**

- Los listados no tienen función propia: `listUsersWithRelations` acepta
  `staffOf: { shopId?, ownerId? }`. Pagina usuarios, no filas: quien es staff
  de dos tiendas sale una vez.
- `createShopStaff({ name, email, passwordHash, shopId })`: en un
  `$transaction` crea el usuario (permisos `customer` y `staff`) y su fila del
  pivote.
  - Tienda inexistente: `InvalidReferenceError`.
  - El email (sin distinguir mayúsculas) ya es staff de esa tienda:
    idempotente, `{ created: false }`; garantiza el permiso `staff`.
  - El email es de otro usuario: `DuplicateEmailError`. No se ata un usuario
    existente en silencio.
- `removeShopStaff({ userId, shopId?, ownerId? })`: borra las filas del
  pivote que pasan el filtro y devuelve cuántas. Si el usuario se queda sin
  tiendas, pierde el permiso `staff`. El usuario no se borra.
- `listStaffShops(userId)`: las tiendas de las que es staff, `shop_id ASC`.

El permiso global `staff` refleja si el usuario es staff de alguna tienda. El
seed sigue la misma regla.

**`ownership-transfers.repository.ts`**: los `*Info` son `JsonValue`;
`OwnershipTransferDetailRecord` añade `previousOwner` y `currentOwner`. Quién
ve qué lo decide la API y llega como el filtro `participantId`.

- `listOwnershipTransfers`: `created_at DESC, id DESC`, `limit` 10.
  `transactionIdentifier` es exacto.
- `findOwnershipTransferByIdentifier`, `deleteOwnershipTransfer`.
- `createOwnershipTransfer`: solicitud `pending`; el dueño anterior se lee
  con `FOR UPDATE`. No toca `shops`.
- `updateOwnershipTransfer(id, { status, message })`: `status` fuera de
  `OWNERSHIP_TRANSFER_STATUSES` da `InvalidFieldValueError`.
- `transferShopOwnership`: en una transacción bloquea la tienda, valida al
  vendedor (existe y tiene `store_owner`), cambia `shops.owner_id` e inserta
  la fila `approved`. Devuelve `{ shop, owner, transfer }`. Si el vendedor ya
  es el dueño: `SameOwnerError` (409).
- `transaction_identifier` es `YYYY-MM-DD-NNNN` (día UTC de `now()`,
  secuencial por día). Si otra transacción toma el mismo número, el UNIQUE da
  P2002 y se reintenta (hasta 5 veces).

**`withdraws.repository.ts`**: `amount` con `_dec` (`10`, no `"10.00"`).
`listWithdraws` (`id DESC`, `limit` 10, filtro `ownerId` para la
visibilidad), `findWithdrawById`, `createWithdraw` (nace `pending`; `amount`
fuera de `(0, 1e10)` da 400), `updateWithdrawStatus` (valida contra
`WITHDRAW_STATUSES`) y `deleteWithdraw`.

**`become-seller.repository.ts`**: `getBecomeSeller` y `updateBecomeSeller`
sobre la fila única de la página «vender con nosotros».

</details>

<details>
<summary><strong>Configuración</strong>: settings, taxes, shippings, order-statuses</summary>

- **`settings`**: `getSettings()`, la fila única.
- **`taxes`** y **`shippings`**: listados **sin paginar** (`id ASC`, filtro
  opcional por nombre), porque es lo que espera el admin. `find*ById`,
  `create*`, `update*` y `delete*`. Validan `rate` (finito, ≥ 0), `amount`
  (≥ 0) y `type` (`SHIPPING_TYPES`).
- **`order-statuses`**: `listOrderStatuses` (`limit` 30, búsqueda por
  `name`, `orderBy` en `ORDER_STATUS_SORTABLE`; por defecto `serial ASC`,
  desempate `id ASC`), `findOrderStatusBySlug`, `createOrderStatus`,
  `updateOrderStatus` y `deleteOrderStatus`.
  - Serial o slug repetido: `SlugConflictError`. Serial no entero positivo o
    color que no sea `#rrggbb`: `InvalidFieldValueError`.
  - Si llega `name`, **recalcula el slug** sin contar la propia fila.
  - `orders` no tiene FK hacia esta tabla: borrar no cuenta dependientes.

</details>

<details>
<summary><strong>Contenido</strong>: faqs, terms-and-conditions, refund-policies, refund-reasons, store-notices</summary>

**`faqs`** y **`terms-and-conditions`**: `list*` (paginado, `limit` 10,
`id ASC`), `findFaqById` / `findTermsAndConditionsBySlug`, `create*`,
`update*`, `delete*`. Términos añade `setTermsApproval` (moderación). Los
filtros se combinan con AND: `shopId`, tipo e `issuedBy` exactos, título con
`contains` sin distinguir mayúsculas, e `isApproved` en los términos.

El tipo y el emisor se derivan de la tienda, así los CHECK
`faqs_faq_type_check` y `terms_and_conditions_type_check` no pueden
violarse:

- sin `shopId`: tipo `global`, emisor `GLOBAL_CONTENT_ISSUER`
  (`'Super Admin'`);
- con `shopId`: tipo `shop`, emisor el nombre de la tienda;
- tienda inexistente o id no entero seguro: `InvalidReferenceError`.

**`refund-policies`** y **`refund-reasons`**: paginados (`limit` 10,
`id ASC`). Toman el slug del body si llega no vacío y lo mantienen inmutable.
`listRefundPolicies` filtra `target` y `status` por igualdad y el título con
`contains`; valida los dos CHECK con `REFUND_POLICY_TARGETS` y
`REFUND_POLICY_STATUSES`. `listRefundReasons` filtra por nombre.

**`store-notices.repository.ts`**: `StoreNoticeDetailRecord` añade el
`creator` con sus `permissions`, los `recipients`, las `shops` y los `reads`
(lectura por usuario), cada uno en un orden fijo.

- `listStoreNotices`: `limit` 12; filtros por texto, `userId`, `shopId` o
  `shopSlug`; `orderBy` en `STORE_NOTICE_SORTABLE` con desempate `id DESC`.
- `findStoreNoticeById`, `deleteStoreNotice` (los pivotes caen por cascada).
- `createStoreNotice`: el `type` decide los destinatarios (ver
  `db/README.md`) y cada usuario destinatario recibe una lectura en `false`.
  Es un único nested write.
- `updateStoreNotice`: si llegan `type` o `receivedBy`, reconcilia los
  pivotes en un `$transaction` y conserva la lectura de quien sigue siendo
  destinatario.
- `markStoreNoticeRead`: upsert de la lectura del usuario.
- Valida los tres CHECK; exporta `STORE_NOTICE_PRIORITIES` y
  `STORE_NOTICE_TYPES`.

</details>

<details>
<summary><strong>Ventas</strong>: coupons, flash-sale, flash-sale-requests, orders, order-files, wishlists</summary>

**`coupons.repository.ts`**: `CouponRecord` no trae `isValid`, porque depende
del reloj: lo calcula la API con `now()`. `image: null` se trata como
ausente.

- `listCoupons`: `id DESC`, `limit` 12; `shopId` y `code` (parcial, sin
  distinguir mayúsculas) con AND.
- `findCouponByCode` (exacto, distingue mayúsculas) y `findCouponById`.
- `createCoupon`: nace con `isApprove` en `false`. `code` repetido:
  `SlugConflictError`; tienda inexistente: `InvalidReferenceError` en
  `shop_id`.
- `updateCoupon`: `shopId` y `userId` no cambian; la vigencia se valida
  contra la fila, así que basta con que cambie una fecha.
- `setCouponApproval` y `deleteCoupon`. Valida los cuatro CHECK (`NaN` y
  fechas inválidas incluidos); exporta `COUPON_TYPES`.

**`flash-sale.repository.ts`**: `FlashSaleRecord` trae `products`
(hidratados con `PRODUCT_INCLUDE`, en orden de `product_id`).
`startDate`/`endDate` se leen como UTC de columnas `timestamp` sin zona.

- `listFlashSales` (`id DESC`, `limit` 10, `title` ILIKE, sin filtro de
  vigencia), `findFlashSaleBySlug`.
- `listFlashSaleProducts({ slug, page, limit })`: `id ASC`; sin `slug`, los de
  todas las ventas sin repetir; slug inexistente: `null`.
- `listProductFlashSales(productId)`: las ventas que contienen el producto;
  producto inexistente: `null`.
- `createFlashSale` / `updateFlashSale`: validan los CHECK y que existan
  todos los `productIds` antes de escribir; venta y tabla puente van en una
  transacción. `saleBuilder.product_ids` se reescribe con los `productIds`
  deduplicados. En el update, `productIds` presente **reemplaza** el
  conjunto; ausente, no lo toca.
- `deleteFlashSale`: devuelve la venta con los productos que tenía.

**`flash-sale-requests.repository.ts`**: solicitudes de un vendedor para
entrar en una venta. `status` es `pending | approved | rejected`. La tienda
se deduce de los productos y no cambia al editar.

- **Visibilidad:** con `viewerId`, las lecturas solo ven solicitudes de
  tiendas de las que ese usuario es dueño o staff; sin `viewerId` (super
  admin), todas. La autorización de las escrituras vive en el servicio de
  Nest.
- `listFlashSaleRequests`, `findFlashSaleRequestById`,
  `listFlashSaleRequestProducts` (`null` si no existe o no es visible).
- `resolveFlashSaleRequestShop(productIds)`: devuelve `{ shopId, productIds }`;
  sin productos o de dos tiendas: `InvalidFlashSaleRequestError` (400).
- `createFlashSaleRequest` (`pending`, `note` no vacía) y
  `updateFlashSaleRequest` (solo si está `pending`, si no
  `FlashSaleRequestStateError` 409).
- `deleteFlashSaleRequest`: no se puede borrar una aprobada (409); hay que
  rechazarla antes.
- `approveFlashSaleRequest`: mete en la venta los productos que faltan y los
  marca con `addedToSale`. `rejectFlashSaleRequest`: si estaba aprobada,
  retira de la venta **solo** los marcados. Así rechazar no quita productos
  que ya estaban en la venta por otra vía.
- Aprobar y rechazar bloquean solicitud y venta (`SELECT … FOR UPDATE`),
  reescriben `sale_builder.product_ids` desde la tabla puente y fijan
  `updatedAt: now()` en ambas.

**`orders.repository.ts`**: el dinero sale como `number`; snapshots y
direcciones como `JsonValue`. `OrderDetailRecord` añade `lines` (por
`lineNo`), `shop` (solo en las hijas) y `children`.

- `listOrders`: `id DESC`, `limit` 15. Sin `shopId` ni `trackingNumber`, solo
  padres; con `shopId`, las hijas de esa tienda; con `trackingNumber`, la
  orden exacta. `customerId` filtra si llega; el aislamiento por cliente lo
  decide la API.
- `findOrderByIdOrTracking(number | string)`: un `number` busca por `id`, un
  `string` por `tracking_number`.
- `findProductsForOrder(ids)`: los productos vivos de un carrito.
- `createOrderWithChildren`: en un `$transaction`, el padre, una hija por
  tienda con sus líneas y el descuento de stock (`updateMany … where
  quantity >= n`). Sin stock: `InsufficientStockError` y rollback. Genera los
  `tracking_number` (`YYYYMMDD` UTC + 6 dígitos, hasta 5 intentos).
- `setOrderStatus(id, { orderStatus?, paymentStatus? })`: valida contra
  `ORDER_STATUSES` / `PAYMENT_STATUSES` y propaga a las hijas. No se llama
  `updateOrderStatus` porque ese nombre es del catálogo de estados.
- `markOrderPaid(trackingNumber)`: `order-processing` + `payment-success` en
  la orden y sus hijas.
- `deleteOrder`: hijas y líneas por cascada. Si el `SET NULL` de
  `reviews.order_id` chocaría con otra reseña del autor sin orden, da
  `SlugConflictError` y no borra nada.

**`order-files.repository.ts`**: descargas digitales, siempre de un cliente.
`listOrderFilesByCustomer` (`id ASC`, `limit` 30) y
`findFirstOrderFileByDigitalFileId(digitalFileId, customerId)`.

**`wishlists.repository.ts`**: no hay `WishlistRecord`; las lecturas
devuelven `ProductRecord`. Todo va por usuario.

- `listWishlistProducts(userId, { page, limit })`: `created_at DESC`,
  desempate `product_id DESC`, `limit` 30.
- `isInWishlist`, `findWishlistProduct`.
- `addToWishlist` (idempotente), `removeFromWishlist`
  (`RecordNotFoundError('wishlists', id)` si no estaba) y `toggleWishlist`
  (devuelve `true` si ahora está).
- Un producto inexistente da `RecordNotFoundError('products', id)`.

</details>

<details>
<summary><strong>Comunidad</strong>: reviews, abusive-reports, questions, conversations</summary>

**`reviews.repository.ts`**: `ReviewDetailRecord` añade el autor, el
producto hidratado y `abusiveReportsCount`. El conteo sale de **un**
`groupBy` sobre `abusive_reports` por página: la tabla es polimórfica y no
hay relación para `_count`.

- `listReviews`: `id DESC`, `limit` 30; filtros `productId`, `shopId`,
  `rating` y `typeSlug` con AND.
- `findReviewById`.
- `createReview`: el producto debe existir y `shopId` debe ser su tienda.
  Mismo usuario, producto y orden: `SlugConflictError`.
- `updateReview`: solo `rating` (1..5), `comment` y `photos` (array).
- `deleteReview`: borra la reseña y sus reportes en un `$transaction`.

**`abusive-reports.repository.ts`**

- `listAbusiveReports` (`id DESC`, moderación) y `listAbusiveReportsByUser`
  (`created_at DESC`).
- `findAbusiveReportById`, `updateAbusiveReport` (solo `message`, no vacío),
  `deleteAbusiveReport`.
- `createAbusiveReport`: valida el objeto con `assertReportableExists`. Tipos
  `'Review'` y `'Question'` se buscan en su tabla (`RecordNotFoundError` si no
  existen); cualquier otro tipo da `InvalidFieldValueError`. Reporte
  repetido: `SlugConflictError`. Tipos admitidos: `REPORTABLE_MODEL_TYPES`.
- `rejectAbusiveReports(modelType, modelId)`: rechazar es borrar **todos** los
  reportes del objeto y dejar el objeto intacto. Devuelve cuántos borró. Como
  las filas desaparecen, el objeto se puede volver a reportar.

**`questions.repository.ts`**: `answer` en `null` significa sin responder.
`QuestionDetailRecord` añade autor, producto y `abusiveReportsCount`, como
las reseñas.

- `listQuestions` (`id DESC`, `limit` 10; filtros `productId`, `shopId`,
  `question` parcial y `typeSlug`) y `listUserQuestions(userId)`
  (`created_at DESC, id DESC`, `limit` 8).
- `findQuestionById`, `createQuestion` (el producto debe existir y `shopId`
  ser su tienda), `answerQuestion`, `updateQuestionText` (el autor; no toca
  la respuesta) y `deleteQuestion` (con sus reportes, en un
  `$transaction`).
- No hay UNIQUE: un usuario puede preguntar varias veces. Quién responde lo
  decide la API (`super_admin`, el dueño o el staff vía `isShopStaff`).

**`conversations.repository.ts`**: `ConversationDetailRecord` trae usuario,
tienda, `latestMessage` y `lastSeenAt` (la última lectura de quien consulta).

- `latestMessage` sale de **una** consulta `DISTINCT ON (conversation_id)`
  sobre los ids de la página. Un `include` con `take: 1` traería todos los
  mensajes y recortaría en memoria.
- **Visibilidad** (`ConversationViewer = { userId, seeAll }`): con `seeAll`,
  todo; si no, las propias y las de las tiendas que el usuario posee.
- `listConversations` (`limit` 15, búsqueda por `shopName`, `orderBy` en
  `CONVERSATION_SORTABLE`; por defecto `id ASC`), `findConversationById`,
  `findOrCreateConversation(userId, shopId)`.
- `listMessages(conversationId, { page, limit, readerId })`: `id DESC`.
- `createMessage`: inserta y fija `conversations.updated_at = now()` en un
  `$transaction`. Cuerpo vacío: `InvalidFieldValueError`.
- `markConversationSeen(id, viewer)`: upsert de `last_seen_at = now()`.
- `unseen` no vive aquí: depende de quién consulta y lo calcula la API.

</details>

<details>
<summary><strong>Identidad y acceso</strong>: users, auth-tokens</summary>

**`users.repository.ts`**: `findUserById`, `findUserWithRelations`,
`findUserCredentialsByEmail`, `listUsers`, `listUsersWithRelations`,
`createUser`, `updateUserPasswordHash`, `setUserActive` y `grantPermission`.

- **El hash no cruza la frontera de serialización.** `passwordHash` solo
  viaja en el tipo `UserCredentials`, que vive en `users.repository.ts` y no
  en `records.ts`. La única función que lo devuelve es
  `findUserCredentialsByEmail`; `createUser` y `updateUserPasswordHash` lo
  reciben ya hasheado. Hashear y verificar es trabajo de la API: `bcryptjs`
  no entra en este paquete.
- **Por qué `findUserCredentialsByEmail` usa `$queryRaw`.** La unicidad del
  email sin distinguir mayúsculas vive en `users_email_lower_idx`, un índice
  de expresión sobre `lower(email)`. Ni `mode: 'insensitive'` (genera
  `ILIKE`) ni normalizar en JS antes de un `equals` usan ese índice; solo
  `WHERE lower(email) = lower($1)` lo hace. Todo consumidor que compare por
  email debe seguir esta regla (spec
  [`identity-schema`](../../openspec/specs/identity-schema/spec.md)).
- `createUser` traduce el P2002 del email a `DuplicateEmailError`: lo dispara
  un índice de expresión que Prisma no ve.
- `updateUserPasswordHash` y `setUserActive` devuelven `null` si el id no
  existe. `grantPermission` es idempotente.
- `listUsers` devuelve `{ items, total }`.

**`auth-tokens.repository.ts`**: tokens de recuperación de contraseña y
códigos OTP. `createPasswordResetToken`, `findLivePasswordResetTokens`,
`consumePasswordResetToken`, `createOtpCode`, `findLiveOtpCodeById`,
`consumeOtpCode`, `purgeExpiredAuthTokens` y `findUserIdByProfileContact`.
`consumedAt` sale de `now()`.

</details>

<details>
<summary><strong>Dashboard</strong>: analytics</summary>

Solo lectura y sin tabla propia. Todo en UTC; «ahora» es el `Date` que
recibe `getDashboardAnalytics` (la API le pasa `now()`). Solo cuentan
órdenes padre (`parent_id IS NULL`). Usa `$queryRaw` parametrizado; los
importes salen de `numeric` como texto y se convierten con `Number`.

- `getDashboardAnalytics(at)` → `DashboardAnalyticsRecord`:
  - `totalRevenue`: suma de `total` de padres `order-completed` con
    `created_at <= at`. `todaysRevenue`: lo mismo en el día UTC de `at`.
  - `totalShops`: tiendas; `totalVendors`: dueños distintos.
  - `totalOrders`: padres con `created_at <= at`.
  - `newCustomers`: usuarios con `customer`, sin `super_admin` ni
    `store_owner`, creados en `(at - 30d, at]`.
  - `ordersByStatus.{today, weekly, monthly, yearly}`: conteo por
    `order_status`. Solo trae estados con órdenes; la API rellena el resto.
  - `salesByMonth`: 12 importes (enero en `[0]`) del año UTC de `at`.
- `listCategoryWiseProductCounts({ limit })`: productos por categoría y
  tienda. Con el seed da `[]` (`category_product` está vacía).
- `listLowStockProducts({ limit, shopId })`: `quantity < 10`, `id ASC`.
- `listTopRatedProducts({ limit, shopId })`: producto + `totalRating`,
  `ratingCount` y `actualRating` (`round(avg, 2)`).

`limit` es 10 por defecto en las tres listas.

</details>

---

## Por qué el paquete se construye

La API compila con `nest build` (`tsc`), y **`tsc` no transpila archivos que
resuelve dentro de `node_modules`**: los revisa y los deja tal cual. Si el
paquete publicara su `index.ts`, el typecheck pasaría y Node fallaría al
arrancar, porque no ejecuta TypeScript de `node_modules`. Por eso tsup genera
`dist/` y `package.json` apunta ahí:

- **`exports`, y también `main` y `types` en el nivel superior.** La API
  (TypeScript 6, `moduleResolution: nodenext`) lee `exports`; `main` y
  `types` cubren a las herramientas que lo ignoran. Sin ellos, alguna caería
  al `.ts` fuente.
- **Solo CJS.** El paquete exporta un singleton con estado (el cliente de
  Prisma). Emitir CJS y ESM a la vez abre el *dual package hazard*: dos
  copias del módulo, dos clientes y dos pools de conexiones. La API es ESM y
  aun así importa funciones por nombre, porque tsup deja al final de
  `dist/index.js` la anotación `0 && (module.exports = { … })` que Node usa
  para detectar los exports de un CJS. Si algún día hace falta ESM, es añadir
  `'esm'` a `format` en `tsup.config.ts`.
- **`shims: true` es obligatorio.** El cliente generado por Prisma 7 arranca
  con `globalThis['__dirname'] = path.dirname(fileURLToPath(import.meta.url))`.
  En CJS, esbuild cambia `import.meta` por un objeto vacío y
  `fileURLToPath(undefined)` lanza un `TypeError` **al cargar el módulo**: el
  build sale en verde y el artefacto no arranca. El shim lo evita.
- **`generated/` va dentro del bundle.** Se importa por ruta relativa; solo
  `@prisma/client`, `@prisma/adapter-pg`, `prisma`, `pg` y `dotenv` quedan
  como `external`.

---

## Problemas conocidos

### La API no ve mi cambio

La API usa `dist/`, no `src/`. Corre `npm run build` en `packages/db` (o
deja `npm run dev` abierto) y reinicia la API si no recarga sola.

### `prepare` no siempre construye

`prepare` corre `npm run build`, pero npm se salta los scripts del ciclo de
vida si el árbol de dependencias ya está al día: un `npm install` repetido no
construye nada. Y con `link:`, yarn no lo corre al instalar la API. Usa
siempre `npm run build` o `just db-build`. En CI se instala con
`npm ci --ignore-scripts` porque el primer paso de `just check` ya construye.

### `DATABASE_URL no está configurada`

`client.ts` lanza ese error en el primer acceso a la base, no al importar.
Revisa el `.env` de quien ejecuta: el de `apps/api/rest` para la API, el de
`packages/db` para los tests.

### vitest corre 0 tests en Windows

Si el directorio actual tiene la letra de unidad en minúscula (`c:\…`),
vitest carga el módulo dos veces y falla con «Vitest failed to find the
current suite» o 0 tests. `just db-check` lo evita con `cd "$(pwd)"`; a mano,
entra con `C:\…`.

### `prisma db pull` pisa los renombres

El pull regenera nombres desde la base. Revisa el diff y vuelve a aplicar los
renombres y los `@map` antes de generar. No quites el preview
`partialIndexes`.
