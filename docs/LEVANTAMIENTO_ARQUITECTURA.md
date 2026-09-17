# Levantamiento de arquitectura — KimünKo

**Fecha:** 2026-09-07
**Commit auditado:** `3708bbe676705504ece8bd00520278a60909cb83` (rama `main`, árbol limpio)
**Alcance:** solo lectura. Se leyó el código de la aplicación y se cruzó con el
proyecto Supabase `eaodsaiwzhbgehhfnegj` mediante consultas de lectura.

---

## 1. Resumen ejecutivo

El proyecto está mejor desacoplado de lo que se asumía y peor estructurado de lo
que se asumía. El navegador **no** habla con la base de datos: el único
consumidor del cliente de navegador es un hook muerto que nadie importa. Todo el
acceso a datos pasa por el servidor. Eso resuelve gratis la mitad del criterio de
desacoplamiento.

A cambio, no existe ninguna capa entre la ruta y el SQL. Las 92 Server Actions
son simultáneamente controlador, validador, repositorio y regla de negocio.
`submitQuizAction` son 317 líneas que hacen las siete cosas. No hay modelo de
dominio: no existe una entidad `Intento` ni una función que responda "¿aprobó?"
fuera del cuerpo de una action.

Tres hallazgos pesan por encima del resto:

1. **`src/lib/actions/sedes.ts` expone tres Server Actions con `service_role` y
   cero autenticación.** `toggleSedeAction` desactiva una sede y ejecuta
   `UPDATE profiles SET sede = null` sobre todos sus trabajadores. Es alcanzable
   por un visitante anónimo. Ver §8.1.
2. **Las reglas de negocio están repartidas entre trigger, constraint y
   TypeScript sin criterio.** El límite de intentos está duplicado en el trigger
   y en el código; la condición de aprobación de curso vive solo en JavaScript;
   el cupo de días administrativos vive solo en TypeScript. Ver §3.3 y §3.4.
3. **Tres carreras reales de escritura concurrente**, ninguna cubierta por
   constraint. Ver §4.4 y §6, criterio 8.

Sobre la hipótesis de diseño: certificación se sostiene como servicio separable,
notificaciones no, IA no existe, y eventos está más acoplado al shell que el
propio núcleo de acreditación. Detalle en §5.

---

## 2. Inventario

### 2.1 Stack real

El brief decía Next.js 14. Es **Next.js 16.1.7** con React 19.2.3. La diferencia
importa: en Next 16 el interceptor de request se llama `proxy.ts`, no
`middleware.ts` (`node_modules/next/dist/lib/constants.js:274`,
`PROXY_FILENAME = 'proxy'`). El archivo `src/proxy.ts` con export nombrado
`proxy` es la convención correcta y **sí se ejecuta**. La entrada BUG-25 del
`CLAUDE.md`, que afirma que hace falta un `src/middleware.ts`, está obsoleta: ese
archivo se creó y se volvió a borrar en `f0f5718`, correctamente.

| Capa | Tecnología |
| :--- | :--- |
| Framework | Next.js 16.1.7 (App Router) |
| UI | React 19.2.3, Tailwind v4, radix-ui, shadcn |
| BaaS | Supabase (`@supabase/ssr` 0.9, `supabase-js` 2.99) |
| Validación | Zod 4.3 |
| PDF | pdf-lib 1.17 |
| Push | web-push 3.6 (VAPID) |
| Gráficos | recharts 3.10 |
| Tests | vitest 4.1 |
| Deploy | Vercel + GitHub Actions |

No hay `vercel.json`. `next.config.ts` declara únicamente
`experimental.optimizePackageImports: ['lucide-react']` — sin regiones, sin
runtimes, sin cabeceras.

### 2.2 Métricas de tamaño

| Métrica | Valor |
| :--- | ---: |
| Archivos `.ts` / `.tsx` en `src/` | 224 |
| Líneas de código en `src/` | 33.096 |
| Páginas (`page.tsx`) | 36 |
| Rutas de API (`route.ts`) | **0** |
| Server Actions exportadas | 92 (en 19 módulos) |
| Client Components | 83 |
| Componentes en `components/alumco/` | 84 |

No existe `/api/bot/mcp` ni ningún otro endpoint HTTP propio. La superficie
pública de la aplicación son las páginas más los endpoints RPC que Next genera
implícitamente por cada Server Action.

Archivos más grandes:

| Líneas | Archivo |
| ---: | :--- |
| 1164 | `src/lib/actions/events.ts` |
| 966 | `src/app/(dashboard)/cursos/[id]/modulos/[moduleId]/quiz/QuizClient.tsx` |
| 806 | `src/lib/actions/courses.ts` |
| 640 | `src/lib/types/databases.ts` |
| 587 | `src/components/alumco/eventos/WizardEvento.tsx` |
| 539 | `src/app/admin/dashboard/page.tsx` |
| 486 | `src/lib/actions/admin-days.ts` |
| 470 | `src/lib/actions/quiz.ts` |

### 2.3 Cobertura de tests

Existen 4 archivos de test, 320 líneas en total, todos unitarios sobre funciones
puras:

| Archivo | Qué cubre |
| :--- | :--- |
| `tests/moduleGates.test.ts` | candado secuencial de módulos |
| `tests/rateLimit.test.ts` | ventana deslizante del rate limit |
| `tests/sanitizeHtml.test.ts` | saneador de HTML de módulos de texto |
| `tests/verification.test.ts` | normalización y validación del folio |

Están bien elegidos: son las cuatro funciones donde un error tiene consecuencia
directa. Pero **ninguna Server Action, ningún componente y ninguna política RLS
tiene test.** No hay tests de integración ni end-to-end, pese a que `playwright`
figura en `devDependencies` sin usarse.

### 2.4 CI/CD

Dos workflows en `.github/workflows/`:

- `auto-deploy.yml`: en push a `main`, hace `curl -X POST` al deploy hook de
  Vercel. **No corre `test`, ni `lint`, ni `typecheck`.** Los tres scripts
  existen en `package.json` y ninguno se ejecuta automáticamente. No hay gate
  alguno entre un commit y producción.
- `alias-preview-clienta.yml`: reasigna el alias `alumcotest.vercel.app` a la
  rama de la landing. Tiene reintento con backoff y timeout — irónicamente, la
  pieza de infraestructura mejor protegida contra fallos del proyecto.

---

## 3. Dónde vive la lógica de negocio

### 3.1 Mapa de Server Actions

92 funciones exportadas en 19 módulos bajo `src/lib/actions/`. Todas comparten la
misma forma: `'use server'`, autenticación inline, validación inline (cuando la
hay), consultas Supabase inline, reglas inline, `revalidatePath` y retorno de un
objeto `{ success | error }` pensado para la UI.

| Módulo | Acciones | Tablas que toca | Validación | Autorización |
| :--- | ---: | :--- | :--- | :--- |
| `events.ts` | 15 | `events`, `event_sections`, `event_tasks`, `event_section_members`, Storage | Zod (39 usos) | helper local `caller.role !== 'admin'` |
| `courses.ts` | 11 | `courses`, `modules`, `quizzes`, Storage | Zod (37 usos) | `requireAdmin` |
| `admin-days.ts` | 8 | `admin_day_requests`, `admin_day_config`, `admin_day_area_quotas` | **manual, sin Zod** | `requireAdmin` (solo las de revisión) |
| `trabajadores.ts` | 8 | `profiles`, Storage | Zod (6) | inline `callerProfile?.role !== 'admin'` |
| `support.ts` | 6 | `support_tickets`, `support_messages` | Zod (11) | `requireAdmin` + rate limit |
| `progress.ts` | 5 | `course_progress`, `modules` | ninguna | propia del usuario |
| `auth.ts` | 5 | `profiles`, `auth.users` | Zod (15) | `requireAdmin` en registro |
| `analytics.ts` | 5 | `certificates`, `profiles`, `courses`, `platform_settings` | ninguna | `requireAdmin` |
| `certificates.ts` | 4 | `certificates`, `quiz_attempts`, Storage | ninguna | propia + `requireAdmin` en listado |
| `quiz.ts` | 3 | `quiz_attempts`, `quizzes`, `questions`, `modules`, `courses` | Zod (8) | propia del usuario |
| `registro.ts` | 3 | `profiles`, `auth.users` | Zod (9) | inline |
| `admin-questions.ts` | 3 | `questions` | Zod (11) | `requireAdmin` |
| `push.ts` | 3 | `push_subscriptions` | ninguna | propia del usuario |
| `feedback.ts` | 3 | `course_feedback` | Zod (4) | `requireAdmin` en resumen |
| **`sedes.ts`** | **3** | **`sedes`, `profiles`** | **ninguna** | **NINGUNA** |
| `alerts.ts` | 2 | `courses`, `profiles`, `course_progress` | ninguna | `requireAdmin` |
| `preview.ts` | 2 | cookie | — | `requireAdmin` |
| `search.ts` | 1 | `courses`, `profiles` | ninguna | por rol, inline |
| `preferences.ts` | 1 | `user_preferences` | Zod (5) | propia del usuario |

Zod cubre 10 de 19 módulos. Los 9 sin Zod incluyen `admin-days.ts` (486 líneas
de reglas con validación escrita a mano) y `sedes.ts`.

**Caso representativo — `submitQuizAction`** (`src/lib/actions/quiz.ts:119-424`,
317 líneas). En un solo cuerpo hace:

1. autenticación (`:129`)
2. validación de forma de las respuestas con Zod (`:141`)
3. validación de la cadena quiz→módulo→curso (`:148-172`)
4. autorización por área del trabajador (`:174-198`)
5. lectura de configuración del quiz (`:200`)
6. regla del candado secuencial (`:228-248`)
7. regla del límite de intentos (`:250-291`) — duplicada del trigger
8. corrección del intento y cálculo del puntaje (`:324-334`)
9. escritura del intento (`:336`)
10. orquestación de efectos: marcar módulo, emitir certificado (`:366-398`)
11. invalidación de caché (`:401-402`)
12. formateo de la respuesta para la UI (12 retornos distintos)

No hay ningún punto en ese archivo donde se pueda preguntar "¿este intento
aprueba?" sin arrastrar Supabase, cookies y `next/cache`.

### 3.2 Acceso a datos

Este era el punto crítico del levantamiento. El resultado es favorable.

| Categoría | Cantidad | Detalle |
| :--- | ---: | :--- |
| Client Components que consultan Postgres vía PostgREST | **1, muerto** | `src/hooks/usePendingRequestsCount.ts` es el único importador de `@/lib/supabase/client`. Ningún archivo lo importa a él. Es código muerto. |
| Server Components / Server Actions | **53 archivos** | importan `@/lib/supabase/server` |
| Rutas de API | **0** | no existen |

**El navegador no habla con la base de datos.** El criterio de desacoplamiento no
falla por donde se temía.

Falla por otro lado: no hay capa de servicio. `@/lib/supabase/server` se importa
directamente desde 53 archivos, incluidos 15 `page.tsx` que arman consultas SQL
en el cuerpo del componente de página. `src/app/admin/dashboard/page.tsx` tiene 6
llamadas `.from()` dentro de la página.

**Uso de `service_role`:** `createAdminClient()` se invoca **94 veces** en 35
archivos. Ese cliente salta RLS por completo. Es decir: las 24 tablas tienen RLS
activo, pero el camino normal de la aplicación la esquiva en casi todas las
operaciones administrativas. La RLS protege contra acceso directo a PostgREST con
la clave anónima —que es real y vale— pero no participa de la autorización de la
app. La autorización efectiva es el `if` de TypeScript que precede a cada
`createAdminClient()`, y en `sedes.ts` ese `if` no existe.

### 3.3 Reglas de negocio atrapadas en Postgres

Cruce del código con las 9 RPC, 8 triggers y las políticas RLS. Dato relevante:
**el código de aplicación no llama a ninguna RPC.** `grep '\.rpc('` sobre `src/`
devuelve 0. Las 9 funciones existen exclusivamente para ser invocadas desde
dentro de las políticas RLS.

| Regla | Dónde vive | Clasificación | Comentario |
| :--- | :--- | :--- | :--- |
| `is_admin`, `is_staff`, `user_sede`, `is_event_member`, `is_section_encargado`, `viewer_is_demo` | RPC dentro de políticas RLS | **Guardia de seguridad** | Correcto. Defensa en profundidad, no se invoca desde la app. |
| `handle_new_user` | trigger sobre `auth.users` | **Guardia** aceptable | Garantiza que todo usuario tenga perfil `pendiente`. Que el rol y la sede por defecto estén hardcodeados en plpgsql es una decisión de negocio enterrada, pero de bajo riesgo. |
| `gen_verification_code` + `set_certificate_verification_code` | RPC + trigger | **Guardia** | El folio se genera en la base con reintento sobre colisión. Correcto: garantiza unicidad donde está el índice único. |
| **Límite de intentos de quiz** | trigger `check_and_set_attempt` **y** `quiz.ts:250-291` | **Regla de negocio, duplicada** | Ver §3.4. |
| **Asignación de `attempt_number`** | solo trigger `check_and_set_attempt` | **Regla de negocio atrapada** | La numeración de intentos es dominio puro y solo existe en plpgsql. |
| **Condición de aprobación de curso** | solo `progress.ts:152` | **Regla de negocio en el código** | `allModules.every(m => completedModules.includes(m.id))`. Sin equivalente ni respaldo en la base. |
| **Nota de aprobación del quiz** | solo `quiz.ts:334` | **Regla en el código** | `score >= quiz.passing_score`. Correcto que esté en el servidor de aplicación; no hay defensa en profundidad. |
| **Emisión de certificado** | `certificates.ts:22` y `progress.ts:172` | **Regla duplicada en dos rutas** | Dos caminos de emisión: por quiz aprobado y por curso sin quiz. La unicidad sí está protegida por `UNIQUE (user_id, course_id)`. |
| **Cupo de días administrativos** | solo `admin-days.ts:220-290` | **Regla de negocio en el código** | Cupo, solapamiento, anticipación y bloqueo por cursos vencidos: 70 líneas de TypeScript. La base solo aporta `CHECK (days_count BETWEEN 1 AND 5)` y `CHECK (end_date >= start_date)`. |
| **Scoping por sede** | RLS vía `user_sede()` | **Guardia** | Correcto. Pero los 94 usos de `service_role` lo puentean en el panel admin. |
| **Aislamiento demo/real** | `is_demo` en 99 sitios del código + `SEDE_DEMO` en RLS | **Mixto, frágil** | Es una dimensión de tenencia implementada por convención. Cada nueva action debe acordarse de llamar `courseInScope`/`profileInScope`. Nada lo obliga. |

**La vista `reporte_avance` no se usa.** Aparece tipada en
`src/lib/types/databases.ts:627` y en ningún otro lugar del código. El fix de
BUG-14 la reemplazó por consultas directas en `src/app/admin/reportes/page.tsx` y
nunca se eliminó. Es decir: la vista que el advisor de Supabase marca como ERROR
por `SECURITY DEFINER` es código muerto. La proto-proyección CQRS que se
esperaba encontrar existe en el esquema pero no está conectada a nada.

### 3.4 Duplicación y divergencia

Cuatro casos confirmados:

**a) Límite de intentos — duplicado con semántica distinta.**
El trigger `check_and_set_attempt` cuenta intentos posteriores a
`last_quiz_reset_at` y lanza `P0001` si alcanzó `max_attempts`. `quiz.ts:281`
hace exactamente la misma cuenta en TypeScript antes de insertar. Divergen en un
punto: **el código además rechaza reintentar un quiz ya aprobado**
(`quiz.ts:266-278`, `hasPassedBefore`); el trigger no conoce esa regla. Si
alguien invoca la action saltándose ese camino, o si se agrega un segundo punto
de inserción, un usuario aprobado puede volver a rendir.

**b) Máximo de días por solicitud — mismo número escrito dos veces.**
`ADMIN_DAY_MAX_PER_REQUEST = 5` en `src/lib/types/databases.ts:310` y
`CHECK (days_count >= 1 AND days_count <= 5)` en `admin_day_requests`. Cambiar la
constante sin migrar la constraint produce un error de base sin mensaje útil;
cambiar la constraint sin tocar la constante deja la regla laxa en la base y
estricta en la app.

**c) Candado secuencial — implementado tres veces.**
`computeModuleGates`/`isModuleUnlocked` en `src/lib/utils.ts` se aplica en la UI,
otra vez en `progress.ts:100` y otra vez en `quiz.ts:228`. Aquí la duplicación es
**correcta y deliberada** (la UI no es una defensa), y además es la única regla
con test. Se menciona para contraste: así se ve una duplicación sana.

**d) Emisión de certificado — dos rutas.**
`submitQuizAction` la dispara al completar curso con quiz (`quiz.ts:385`);
`markModuleCompleteAction` la dispara para cursos sin quiz (`progress.ts:172`).
La condición "el curso tiene quiz" se evalúa en `progress.ts:169`. Si un curso
cambia de tener quiz a no tenerlo, la ruta de emisión cambia sin que nada lo
verifique.

---

## 4. Acoplamiento y fronteras

### 4.1 Grafo por capacidad de negocio

Se agrupó por capacidad, no por carpeta. Las flechas son importaciones reales.

```
                    ┌──────────────────────────────────┐
                    │  SHELL (layouts, /inicio, nav)   │
                    └───┬──────────┬───────────┬───────┘
                        │          │           │
        ┌───────────────┘          │           └──────────────┐
        ▼                          ▼                          ▼
 ┌────────────┐            ┌──────────────┐           ┌──────────────┐
 │CAPACITACIÓN│            │   EVENTOS    │           │ DÍAS ADMIN.  │
 │courses     │            │events (1164) │           │admin-days    │
 │progress    │            │lib/eventos   │           │              │
 │quiz        │            └──────┬───────┘           └──────────────┘
 │admin-quest.│                   │                  (usa cursos vencidos
 └─────┬──────┘                   ▼                   por consulta, no import)
       │                   ┌──────────────┐
       ▼                   │NOTIFICACIONES│
 ┌──────────────┐          │lib/push/send │
 │ CERTIFICACIÓN│          │actions/push  │
 │certificates  │          └──────────────┘
 │lib/certific. │
 └──────────────┘

 Sin dependencias entrantes ni salientes entre módulos de acción:
 soporte · reportes (analytics/alerts) · trabajadores · registro · sedes · search
```

Importaciones cruzadas entre los 19 módulos de `lib/actions/`, medidas
exhaustivamente (incluyendo el `import()` dinámico de `quiz.ts:386`):

```
progress  -> certificates
quiz      -> progress, certificates
(los otros 17 módulos: ninguna)
```

**El acoplamiento entre capacidades es casi nulo.** La única cadena es
`quiz → progress → certificates`, que es exactamente el núcleo de acreditación.

| Capacidad | Archivos | Tablas que posee | Importado por | Importa a |
| :--- | :--- | :--- | :--- | :--- |
| Capacitación | `courses.ts`, `progress.ts`, `quiz.ts`, `admin-questions.ts` | `courses`, `modules`, `quizzes`, `questions`, `quiz_attempts`, `course_progress` | shell | certificación |
| Certificación | `certificates.ts`, `lib/certificates/verify.ts` | `certificates` | capacitación (2 sitios) | — |
| Eventos | `events.ts`, `lib/eventos/*` | `events`, `event_sections`, `event_tasks`, `event_section_members` | shell (`/inicio`, `admin/layout`, `admin/dashboard`) | notificaciones |
| Notificaciones | `lib/push/send.ts`, `push.ts` | `push_subscriptions` | **solo eventos** | — |
| Días administrativos | `admin-days.ts` | `admin_day_requests`, `admin_day_config`, `admin_day_area_quotas` | shell | — |
| Soporte | `support.ts` | `support_tickets`, `support_messages` | — | — |
| Reportes | `analytics.ts`, `alerts.ts` | lee de todas | shell | — |
| Personas | `trabajadores.ts`, `registro.ts`, `auth.ts`, `sedes.ts` | `profiles`, `sedes` | shell | — |

### 4.2 Dependencias externas y su resiliencia

| Dependencia | Dónde | Timeout | Reintento | Circuit breaker | Caché | Fallback |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| Logo desde `ongalumco.cl` (PDF) | `certificates.ts:187` | 5 s | no | no | no | sí — `try/catch`, el PDF sale sin logo |
| Firma desde Storage (PDF) | `certificates.ts:371` | 5 s | no | no | no | sí — dibuja una línea |
| Logo desde `ongalumco.cl` (página de error) | `src/app/error.tsx:22` | no | no | no | no | **no** — la página de error depende de un host externo |
| Supabase PostgREST | 94 + 53 sitios | no | no | no | no | parcial — `?? []` en varios sitios |
| Supabase Storage | `events.ts`, `trabajadores.ts` | no | no | no | no | no |
| Web Push (VAPID) | `lib/push/send.ts` | no | no | no | no | **sí, bien hecho** |
| Google Analytics | `layout.tsx:105` | — | — | — | — | `strategy="afterInteractive"`, no bloquea |
| pdf-lib | `certificates.ts` | en proceso | — | — | **no** | — |

**Lo bueno:** `lib/push/send.ts` es la pieza más resiliente del proyecto. Nunca
lanza, degrada a no-op si falta configuración VAPID o si la tabla no existe
todavía, detecta suscripciones muertas por código 404/410 y las borra. Es un
bulkhead correcto, escrito y documentado como tal.

**Lo malo:** el envío usa `Promise.all` sobre todas las suscripciones sin límite
de concurrencia (fan-out del `subs.map` en `send.ts`). Con 60 perfiles hoy no
pasa nada; con varios dispositivos por persona y un evento que notifica a todos,
son cientos de conexiones HTTP simultáneas desde una función serverless.

**Lo peor:** el PDF del certificado se regenera **completo en cada descarga**.
`generateCertificateAction` inserta siempre `pdf_url: null`
(`certificates.ts:65`) y nada llena esa columna después. Cada descarga ejecuta
`generateCertificatePDF` (`certificates.ts:105`), que hace dos peticiones HTTP
externas y compone el documento con pdf-lib dentro del request. La columna
`pdf_url` existe, está tipada y está muerta.

### 4.3 Runtime

**Todas las rutas y Server Actions corren en Node.js.** No hay un solo
`export const runtime` en `src/app/` — el default del App Router es Node, y no
se declara Edge en ninguna parte. `src/proxy.ts` tampoco declara runtime.

Esto satisface el requisito del ramo sin trabajo adicional. Conviene dejarlo
declarado explícitamente en el informe académico, porque hoy se cumple por
omisión y no por decisión registrada. Las 0 Edge Functions de Supabase confirman
que no hay Deno en ninguna parte del sistema.

Configuración de runtime observada:

- 22 páginas con `export const dynamic = 'force-dynamic'`.
- 2 con `export const revalidate = 0`.
- 0 con `runtime`, `preferredRegion`, `unstable_cache`, `revalidateTag`,
  `cacheLife` o `'use cache'`.
- 91 llamadas a `revalidatePath`, mayoritariamente sobre páginas ya declaradas
  `force-dynamic`, donde no hay nada que invalidar.

### 4.4 Concurrencia observada en el código

Constraints reales en las tablas críticas (consultadas en el proyecto vivo):

| Tabla | Unicidad | Efecto |
| :--- | :--- | :--- |
| `certificates` | `UNIQUE (user_id, course_id)` + `UNIQUE (verification_code)` | **Protegida.** La doble emisión no puede ocurrir. |
| `course_progress` | `UNIQUE (user_id, course_id)` | Protege la fila, **no** el contenido del array. |
| `quiz_attempts` | solo PK sobre `id` | **Sin protección.** No hay unicidad sobre `(user_id, quiz_id, attempt_number)`. |

Tres carreras confirmadas:

1. **Límite de intentos.** El trigger hace `SELECT COUNT(*)` y compara antes de
   insertar, sin bloqueo. Dos envíos concurrentes leen el mismo conteo, ambos
   pasan la comparación, ambos insertan. Sin índice único sobre
   `(user_id, quiz_id, attempt_number)` nada lo detiene después. Resultado:
   más intentos que `max_attempts`, con `attempt_number` repetido.
2. **`completed_modules`.** `progress.ts:110-125` lee el array, le agrega un
   elemento en memoria y reescribe la fila completa. Dos módulos completados
   simultáneamente (dos pestañas, o quiz y módulo a la vez) pierden uno de los
   dos. Es un *lost update* de manual.
3. **Cupo de días administrativos.** `admin-days.ts:270-290` lee las solicitudes
   activas, suma los días consumidos, compara contra el cupo e inserta. Dos
   solicitudes simultáneas pueden exceder el cupo. La única constraint que
   interviene limita cada solicitud individual a 5 días, no el total del período.

**Paginación: ninguna.** `grep '\.range('` devuelve 0 sobre todo `src/`. Los
listados de admin traen la tabla completa. Con los volúmenes actuales (60
perfiles, 13 cursos, 5 filas de `course_progress`) es inofensivo. El límite
por defecto de PostgREST son 1000 filas: `course_progress` a adopción plena sería
60 × 13 = 780 filas y sigue bajo el techo, pero lo cruza al agregar cursos o al
incorporar la segunda sede completa. Cuando lo cruce, **los reportes truncarán en
silencio**, sin error.

### 4.5 Contraste con la hipótesis de diseño

La hipótesis se valida a medias. Punto por punto:

**«El núcleo es acreditar que cada trabajador cumple su capacitación obligatoria»
— confirmado como propósito, pero no existe como código.** Las entidades
propuestas (Trabajador, Curso, Intento, Aprobación) no tienen representación:
`src/lib/types/databases.ts` contiene tipos de *fila de tabla*, no entidades de
dominio. No hay ninguna función que responda "¿este trabajador cumple?" — la
pregunta se responde de tres maneras distintas en `alerts.ts`, `analytics.ts` y
`admin-days.ts`, cada una con su propio cruce de deadlines y áreas.

**Las 22 horas del Decreto 20/2022 no están en el código.** No aparece ninguna
constante de horas anuales, ningún acumulador de horas, ninguna referencia al
decreto. `analytics.ts:239` tiene `getAnnualTarget()` leyendo de
`platform_settings`, que es lo más cercano: una meta anual configurable, sin
unidad declarada. La regla que el sistema supuestamente existe para acreditar es
la única que no está implementada.

**Certificación como candidato separable — CONFIRMADO.** Es el seam más limpio.
`lib/certificates/verify.ts` es completamente autónomo (no importa nada del
resto, y su cabecera documenta por qué deliberadamente no es `'use server'`).
`lib/certificates` solo lo importan 2 archivos. `certificates.ts` no importa a
ningún otro módulo de acción. La dependencia va en un solo sentido:
capacitación → certificación, en 2 puntos de llamada
(`quiz.ts:386`, `progress.ts:172`). Extraerlo requiere invertir esa dependencia,
lo cual es un cambio acotado.

**IA (PDF→quiz) como candidato — REFUTADO: no existe.**
`grep -niE 'openai|anthropic|claude|gemini'` sobre `src/` devuelve cero. No hay
código, ni dependencia, ni tabla, ni variable de entorno. No se puede extraer un
servicio que no está escrito. Si es un requisito del ramo, es trabajo nuevo, no
refactorización, y debe presupuestarse como tal.

**Notificaciones como candidato — REFUTADO en la práctica.** `sendPushToUsers`
tiene exactamente **un** consumidor de negocio: `events.ts` (5 llamadas). El
sexto uso, `push.ts:90`, es un botón de prueba. Capacitación y certificación no
notifican nada. Extraer notificaciones hoy separa un módulo que ya es una hoja
sin acoplamiento, y no reduce ninguna dependencia. El argumento arquitectónico a
favor sería a futuro (cuando capacitación notifique vencimientos), no ahora.

**«Eventos y días administrativos quedarían dentro del core» — hay que
corregirlo en ambos sentidos.**

- **Días administrativos es el módulo más aislado del proyecto**, no parte del
  core. `admin-days.ts` no importa ni es importado por ningún otro módulo de
  acción. Posee sus tres tablas en exclusiva. Su único vínculo con capacitación
  es una consulta de cursos vencidos (`getOverdueCourseTitles`), que es una
  lectura, no un acoplamiento estructural. Es el mejor candidato a extracción del
  repositorio, mejor incluso que certificación.
- **Eventos está más acoplado que el núcleo.** `events.ts` son 1164 líneas —el
  archivo más grande— y `lib/eventos/` se filtra al shell en tres puntos:
  `/inicio` (vista del trabajador), `admin/layout.tsx` y `admin/dashboard`.
  Meterlo "dentro del core" consolidaría el módulo más pesado y con más
  superficie de UI dentro de la parte que se quiere mantener limpia.

**Conclusión.** El orden de separabilidad real, medido por acoplamiento, es:
`días administrativos > soporte > certificación > notificaciones > eventos`.
La hipótesis acierta en certificación, se equivoca en notificaciones y eventos,
y propone extraer un módulo de IA que no existe.

---

## 5. Núcleo y servicios candidatos

Resumen ejecutable de §4.5.

**Núcleo propuesto (y lo que falta para que exista).** Trabajador, Curso, Módulo,
Intento, Aprobación, Certificado. Hoy no hay ningún archivo que los modele. El
trabajo mínimo para que el núcleo exista es un módulo de dominio puro —sin
Supabase, sin `next/*`, testeable con vitest— que contenga: corrección de un
intento, decisión de aprobación de curso, y cálculo de cumplimiento anual
(incluida la regla de las 22 horas, que hay que escribir por primera vez).

**Candidatos a separación, ordenados por facilidad real:**

| Candidato | Acoplamiento actual | Veredicto |
| :--- | :--- | :--- |
| Días administrativos | ninguno | Separable hoy. No estaba en la hipótesis. |
| Soporte | ninguno | Separable hoy. No estaba en la hipótesis. |
| Certificación | 2 puntos de llamada entrantes | Separable con inversión de dependencia. **Confirma la hipótesis.** |
| Notificaciones | 1 consumidor (eventos) | Separable pero sin beneficio hoy. **Refuta la hipótesis.** |
| IA (PDF→quiz) | no existe | No es separación, es construcción. **Refuta la hipótesis.** |
| Eventos | filtra al shell en 3 puntos | El más acoplado. **Refuta la hipótesis.** |

---

## 6. Estado por criterio de la rúbrica

| # | Criterio | Estado | Evidencia | Brecha | Esfuerzo |
| :-: | :--- | :--- | :--- | :--- | :--- |
| 1 | Personalización de UI e i18n | **Parcial** | Personalización real y bien resuelta: `user_preferences` (escala tipográfica, alto contraste, movimiento reducido) leída en el servidor en `layout.tsx:75` y aplicada como `data-*` en `:81-88`, sin parpadeo de hidratación; `preferences.ts` valida con Zod y degrada a `DEFAULT_PREFERENCES` si la tabla falta. i18n: **cero infraestructura**, `<html lang="es">` fijo, 10+ `toLocaleDateString('es-CL')` dispersos, todos los textos inline en JSX | Falta toda la mitad de internacionalización: no hay diccionarios, ni negociación de locale, ni segmento `[lang]`, ni formateo centralizado | **Alto** — los textos están inline en 36 páginas y 83 client components; extraerlos es trabajo mecánico pero extenso |
| 2 | Desacoplar la solución | **Parcial** | A favor: cero acceso navegador→PostgREST (el único consumidor de `supabase/client.ts` es un hook muerto); acoplamiento entre los 19 módulos de acción reducido a `quiz→progress→certificates`. En contra: no existe capa de servicio — `@/lib/supabase/server` se importa en 53 archivos, incluidos 15 `page.tsx` con SQL en el cuerpo del componente | Falta una capa de casos de uso y repositorios que aísle el dominio del detalle Supabase | **Medio** — el grafo ya está limpio; el trabajo es introducir la capa, empezando por el núcleo |
| 3 | Arquitectura sin servidor con servicios de nube | **Cumplido** | Vercel serverless + Supabase gestionado (Auth, Postgres, Storage). Cero servidores, cero contenedores. Node.js en todas las rutas: ningún `export const runtime` en `src/app/`, 0 Edge Functions en Supabase. Web Push por VAPID. `pg_cron` instalado | Menor: el job de `pg_cron` está `active = false`, así que la tarea programada no corre. Sin colas (pgmq/pg_net disponibles, no instalados): push y generación de PDF corren dentro del request | **Bajo** para documentar; **medio** si se mueve trabajo diferido a cola |
| 4 | Principios de Clean Architecture | **Ausente** | `submitQuizAction` (`quiz.ts:119-424`) mezcla las cuatro capas en 317 líneas; `src/lib/types/databases.ts` modela filas, no entidades; regla de negocio en trigger, en `CHECK` y en TS sin criterio (§3.3); 68 `as any` con 76 `eslint-disable` de `no-explicit-any`, contra la norma "cero any" del propio `CLAUDE.md` | Falta el núcleo de dominio completo: entidades, reglas puras testeables, e inversión de dependencia hacia Supabase | **Alto** si se aplica a todo; **medio** acotado a acreditación (quiz + progreso + certificado) |
| 5 | Patrones de diseño de nube | **Parcial** | Presentes sin nombrarse: **Valet Key** (`getDocumentSignedUrlAction`, `events.ts:962`; `firmarFotos`), **Throttling** (`lib/rateLimit.ts`, 2 de 92 actions), **Bulkhead** (`lib/push/send.ts`, nunca lanza), **Timeout** (2 `AbortController` en `certificates.ts`), **Event Sourcing en miniatura** (`quiz_attempts` append-only, `Update = never` en tipos), **Static Content Hosting** (el service worker cachea solo inmutables). Ausentes: Retry, Circuit Breaker, Cache-Aside, Compensating Transaction, Queue-Based Load Leveling | **CQRS muerto**: `reporte_avance` está tipada (`databases.ts:627`) y no se usa en ningún otro lugar — BUG-14 la reemplazó por consultas directas y la vista quedó huérfana. Es además la que el advisor marca ERROR por `SECURITY DEFINER` | **Bajo-medio** — varios patrones ya existen y solo necesitan nombrarse y documentarse; Retry y Cache-Aside son adiciones acotadas |
| 6 | Chaos Architecture | **Ausente** | Lo que hay: 20 `loading.tsx`, un `error.tsx` global, `not-found.tsx`, service worker con precache de `/offline` y política explícita de no cachear HTML ni datos. Lo que no hay: cero boundaries de error por segmento, cero `Suspense`, cero `ErrorBoundary`, sin health check, sin `instrumentation.ts`, sin Sentry/OTel/logger (solo 34 `console.*`), sin experimentos de fallo | No hay forma de saber que algo falló en producción ni de provocar un fallo controlado. Detalle irónico: `error.tsx:22` carga el logo desde `ongalumco.cl` — la página de error depende de un host externo | **Bajo** para lo básico (boundaries por segmento + `instrumentation.ts` + health check); **medio** para experimentos de inyección de fallos |
| 7 | Modelo de seguridad explícito | **Parcial, con agujero crítico** | A favor: RLS en 24 tablas; `requireAdmin` cacheado por request; `previewMode` que reverifica contra la base y no confía en la cookie (`previewMode.ts:29`); `demoScope` con `courseInScope`/`profileInScope`; saneado de HTML con test; puntaje calculado siempre en servidor (`quiz.ts:324`); folio Crockford base32 con rate limit (`verificar/[codigo]/page.tsx:39`, 20 por 10 min); Zod en 10 módulos. En contra: **`sedes.ts` con `service_role` y cero autenticación** (§8.1); 94 usos de `service_role` que puentean RLS; autorización escrita en 4 dialectos distintos; sin rate limit en login ni registro; `reset_demo_world()` ejecutable por `anon` | Falta un único punto de decisión de autorización y una política sobre cuándo se permite `service_role` | **Bajo** para tapar `sedes.ts` (una guarda); **medio** para unificar los 4 dialectos |
| 8 | Alta concurrencia | **Ausente** | Tres carreras confirmadas y sin protección: límite de intentos (`quiz_attempts` sin unicidad sobre `(user_id, quiz_id, attempt_number)`; el trigger cuenta sin bloqueo), *lost update* de `completed_modules` (`progress.ts:110-125`), y cupo de días por check-then-insert (`admin-days.ts:270-290`). Sin paginación en ningún listado (0 usos de `.range()`). Fan-out de push sin límite de concurrencia. Lo único protegido: `certificates` con `UNIQUE (user_id, course_id)` | Faltan constraints que hagan imposible el estado inválido, y escrituras atómicas donde hoy hay leer-modificar-escribir | **Bajo** — son un índice único, una operación atómica sobre el array (o una RPC transaccional) y una constraint de exclusión; el diagnóstico ya está hecho |
| 9 | Disponibilidad y respuesta rápida | **Parcial** | A favor, y son decisiones deliberadas y documentadas: `getClaims()` en vez de `getUser()` en el proxy y en `getCachedUser` (verifica el JWT localmente con JWKS, ahorra 100-300 ms por navegación — `server.ts` y `supabase/middleware.ts`); `cache()` de React memoizando por request; 48 `Promise.all` paralelizando lecturas; 20 `loading.tsx`; service worker con `/offline`; `optimizePackageImports`. En contra: 22 páginas `force-dynamic` con **cero** uso de la caché de datos de Next (0 usos de `unstable_cache`, `revalidateTag`, `'use cache'`), lo que vuelve no-op buena parte de los 91 `revalidatePath`; PDF del certificado regenerado en cada descarga con 2 fetch externos y `pdf_url` siempre `null` (`certificates.ts:65`); sin observabilidad; sin health check; CI sin gates | Falta estrategia de caché deliberada, persistencia del PDF y cualquier medición de latencia real | **Medio** |

**Recuento:** 1 cumplido, 5 parciales, 3 ausentes.

---

## 7. Qué está bien

Concreto, sin generosidad.

1. **El navegador no habla con la base de datos.** Era el riesgo principal del
   levantamiento y no se materializó. El único consumidor del cliente de
   navegador (`usePendingRequestsCount.ts`) es código muerto que nadie importa.
   La mitad difícil del criterio 2 ya está resuelta.
2. **El acoplamiento entre capacidades es genuinamente bajo.** 19 módulos de
   acción y solo dos aristas entre ellos. Esto no es habitual y hace que la
   discusión de fronteras de servicio sea real en vez de teórica.
3. **`lib/push/send.ts` es un bulkhead correcto.** Nunca lanza, degrada a no-op
   ante configuración faltante o tabla inexistente, y limpia suscripciones
   muertas por código HTTP. Está escrito y comentado con esa intención explícita.
4. **La optimización de autenticación está bien razonada y bien documentada.**
   Cambiar `getUser()` por `getClaims()` en el proxy y en `getCachedUser` elimina
   un round trip de red por navegación. El comentario explica el porqué y el
   costo. Es el tipo de decisión que la rúbrica de disponibilidad quiere ver.
5. **`previewMode` no confía en la cookie.** `isPreviewMode()` siempre reverifica
   el rol contra la base (`previewMode.ts:29`). Un trabajador que se ponga la
   cookie a mano no gana nada. El comentario lo dice antes de que uno lo
   pregunte.
6. **`quiz_attempts` es append-only de verdad**, con `Update = never` en el tipo
   y sin ninguna ruta de escritura que actualice. Es un event store en miniatura
   funcionando, y sirve como base honesta para argumentar Event Sourcing.
7. **Las preferencias de accesibilidad se resuelven en el servidor.** Aplicarlas
   tras la hidratación haría parpadear la página con el tamaño equivocado justo
   para quien necesita el tamaño grande. El código lo evita y explica por qué.
8. **Los cuatro tests existentes están bien elegidos.** Candado secuencial, rate
   limit, saneado de HTML y folio: las cuatro funciones puras donde un error
   tiene consecuencia directa. La cobertura es mínima, pero no es arbitraria.
9. **`certificates` tiene la unicidad correcta.** `UNIQUE (user_id, course_id)`
   hace imposible la doble emisión. Es la única de las cuatro carreras candidatas
   que está cerrada, y lo está en el lugar correcto: la base.
10. **El service worker tiene una política de caché defendible.** Cachea solo
    inmutables con hash en el nombre y `/offline`; nunca HTML ni datos, porque
    dependen de auth y RLS. Está documentado en la cabecera del archivo.

---

## 8. Qué está mal

Ordenado por gravedad.

### 8.1 CRÍTICO — `sedes.ts` expone `service_role` sin autenticación

`src/lib/actions/sedes.ts` declara `'use server'` y exporta tres funciones. Las
tres usan `createAdminClient()` (clave `service_role`, salta RLS por completo).
**Ninguna de las tres verifica que exista una sesión, y mucho menos un rol.**

Las Server Actions son endpoints POST alcanzables desde fuera de la aplicación.
Consecuencias por función:

- `getSedesAction()` (`:6`) — enumera todas las sedes. Impacto bajo.
- `createSedeAction(formData)` (`:23`) — inserta sedes arbitrarias. Impacto medio.
- `toggleSedeAction(sedeId, activa)` (`:48`) — **destructivo**. Con
  `activa = false` desactiva la sede y a continuación ejecuta
  `UPDATE profiles SET sede = null WHERE sede = sedeId` (`:62-65`). Eso desasigna
  de su sede a todos los trabajadores afectados, de forma irreversible: el valor
  anterior no se guarda en ninguna parte. Además rompe el scoping por sede,
  porque `user_sede()` —que sustenta las políticas RLS de eventos y del mundo
  demo— pasa a devolver `null` para esos usuarios.

Con los 60 perfiles actuales, una sola llamada anónima puede desasignar a todo el
padrón. Es el mismo patrón que el `reset_demo_world()` ejecutable por `anon` que
ya se está tratando por separado, y debe cerrarse con la misma urgencia.

### 8.2 GRAVE — La autorización está escrita en cuatro dialectos

No hay un punto único de decisión. Conviven:

1. `requireAdmin()` — 41 llamadas en 10 módulos.
2. Comprobación inline `callerProfile?.role !== 'admin'` — `trabajadores.ts`,
   `registro.ts`.
3. Helper local con `caller.role !== 'admin'` — `events.ts`, 11 sitios.
4. Nada — `sedes.ts`.

Además `requireAdmin` acepta `admin` **y** `profesor` (se amplió en BUG-63), pero
los dialectos 2 y 3 comparan solo contra `'admin'`. Un `profesor` está autorizado
o no según qué módulo toque, sin que eso esté declarado en ninguna parte. Con 94
usos de `service_role`, cada uno de esos `if` es la única barrera que queda.

### 8.3 GRAVE — Tres carreras de escritura concurrente sin protección

Detalladas en §4.4. La más seria es el límite de intentos, porque es la regla que
decide si alguien puede seguir rindiendo una evaluación cuyo resultado emite un
certificado con validez ante fiscalización. `quiz_attempts` no tiene unicidad
sobre `(user_id, quiz_id, attempt_number)`, y el trigger cuenta sin bloqueo.

### 8.4 GRAVE — Ningún gate entre un commit y producción

`auto-deploy.yml` hace `curl` al deploy hook en cada push a `main`. Los scripts
`test`, `lint` y `typecheck` existen y no se ejecutan nunca de forma automática.
Con 68 `as any` en el código y una suite de 4 archivos, la única verificación
real hoy es que alguien los corra a mano y se acuerde de mirar el resultado.

### 8.5 IMPORTANTE — Deuda de tipos contra la propia norma del proyecto

`CLAUDE.md` establece "TypeScript estricto — cero `any`". El código tiene **68
`as any`** en 17 archivos y **76 `eslint-disable` de `no-explicit-any`**. El
patrón dominante es `const sp = supabase as any` para esquivar los tipos
generados. El efecto práctico: las consultas Supabase no tienen verificación de
tipos en ninguna parte del proyecto, y los `as unknown as never` en los `insert`
desactivan también la verificación de escritura. Es la norma más citada del
`CLAUDE.md` y la más incumplida.

### 8.6 IMPORTANTE — Sin observabilidad

34 `console.*` en todo `src/`. Sin Sentry, sin OpenTelemetry, sin logger
estructurado, sin `instrumentation.ts`, sin health check. Google Analytics mide
navegación de usuarios, no salud del sistema. Hoy no hay forma de saber si
`submitQuizAction` está fallando en producción salvo que alguien reclame. Esto
bloquea a la vez el criterio 6 y el 9: no se puede argumentar resiliencia sobre
un sistema que no se observa.

### 8.7 IMPORTANTE — El PDF del certificado se regenera en cada descarga

`generateCertificateAction` inserta `pdf_url: null` (`certificates.ts:65`) y
nada llena esa columna. Cada descarga ejecuta `generateCertificatePDF`
(`certificates.ts:105`): dos peticiones HTTP externas más composición con pdf-lib
dentro del request. Es el camino más lento de la aplicación y el más expuesto a
un fallo externo, para un documento que por definición es inmutable una vez
emitido.

### 8.8 MEDIO — Reglas de negocio duplicadas con divergencia posible

Los cuatro casos de §3.4. El más peligroso es el límite de intentos, donde el
código conoce una regla que el trigger ignora ("no reintentar un quiz aprobado").

### 8.9 MEDIO — Sin paginación en ningún listado

0 usos de `.range()`. Con los volúmenes actuales no duele. El límite por defecto
de PostgREST son 1000 filas y `course_progress` a adopción plena queda cerca
(60 × 13 = 780). Cuando lo cruce, los reportes de cumplimiento **truncarán en
silencio**, sin error y sin señal. Para un sistema cuyo propósito es acreditar
cumplimiento ante fiscalización, un reporte que subcuenta sin avisar es peor que
uno que falla.

### 8.10 MEDIO — Código muerto en posiciones sensibles

- `reporte_avance`: tipada en `databases.ts:627`, sin ningún uso. Es la vista que
  el advisor marca ERROR por `SECURITY DEFINER`. Se arrastra una alerta de
  seguridad por una vista que nadie consulta.
- `usePendingRequestsCount.ts`: único consumidor del cliente de navegador, sin
  importadores. Además su consulta (`count` de perfiles `pendiente`) no
  funcionaría bajo la política RLS `auth.uid() = id`.
- `pdf_url` en `certificates`: columna tipada, siempre `null`.
- `playwright` en `devDependencies` sin un solo test que lo use.
- El job de `pg_cron` que ejecuta `reset_demo_world()` está `active = false`: el
  reinicio programado del mundo demo no corre. La ruta programada está apagada y
  la ruta anónima está abierta — exactamente al revés de lo correcto.

### 8.11 MENOR — La página de error depende de un host externo

`src/app/error.tsx:22` carga el logo desde `ongalumco.cl` con `<img>`. Si ese
host está caído, la pantalla que se muestra cuando algo ya falló muestra un
recurso roto. Es también la única `<img>` sin `next/image` que queda tras BUG-69.

---

## 9. Qué hay que hacer

Backlog priorizado. Ejecutable por tres personas en un trimestre, incremental,
sin herramientas nuevas salvo donde se justifica. Nada de esto exige reescribir.

### Bloque A — Cerrar riesgos (primero, sin discusión)

**A1. Autenticar `sedes.ts`.**
*Qué:* añadir `requireAdmin()` al inicio de las tres actions; en
`toggleSedeAction`, además, decidir si el `UPDATE profiles SET sede = null` debe
existir (probablemente deba bloquearse la desactivación mientras haya perfiles
asignados).
*Criterio:* 7. *Archivos:* `src/lib/actions/sedes.ts`.
*Depende de:* nada. *Esfuerzo:* bajo (una hora).

**A2. Gates en CI.**
*Qué:* un workflow que corra `npm run typecheck && npm run lint && npm test`
antes del deploy hook.
*Criterio:* 9, y habilita todo lo demás. *Archivos:*
`.github/workflows/auto-deploy.yml`.
*Depende de:* nada. *Esfuerzo:* bajo.

**A3. Cerrar las tres carreras con constraints.**
*Qué:* (a) índice único sobre `quiz_attempts (user_id, quiz_id, attempt_number)`;
(b) reemplazar el leer-modificar-escribir de `completed_modules` por una
escritura atómica —`array_append` con `DISTINCT` en una RPC, o una tabla
`module_completions` con unicidad, que además elimina el array; (c) constraint o
RPC transaccional para el cupo de días.
*Criterio:* 8. *Archivos:* migración nueva, `progress.ts:110-125`,
`admin-days.ts:270-290`.
*Depende de:* A2 (para no romper producción a ciegas). *Esfuerzo:* bajo-medio.
*Nota:* (b) es la decisión de diseño más interesante del bloque y conviene
tomarla explícitamente — ver D2.

**A4. Unificar la autorización.**
*Qué:* llevar los dialectos 2, 3 y 4 a `requireAdmin`, y decidir de una vez qué
puede hacer `profesor`.
*Criterio:* 7, 4. *Archivos:* `events.ts`, `trabajadores.ts`, `registro.ts`,
`sedes.ts`, `lib/auth/requireAdmin.ts`.
*Depende de:* A1. *Esfuerzo:* medio.

**A5. Borrar el código muerto.**
*Qué:* eliminar `usePendingRequestsCount.ts` (y con él el último importador de
`supabase/client.ts`, lo que permite argumentar el desacoplamiento sin
asteriscos), eliminar la vista `reporte_avance` (cierra el advisor ERROR sin
tocar nada vivo), y decidir sobre `pdf_url` (ver B3) y el job de `pg_cron`.
*Criterio:* 2, 7. *Archivos:* `src/hooks/`, migración, `databases.ts:627`.
*Depende de:* nada. *Esfuerzo:* bajo.

### Bloque B — Ganar criterios con trabajo acotado

**B1. Núcleo de dominio para acreditación.**
*Qué:* un módulo `src/core/` sin imports de `@supabase`, `next/*` ni React, con:
`corregirIntento(preguntas, respuestas, notaAprobacion)`,
`cursoAprobado(modulos, completados)`, `cumplimientoAnual(...)` incluida la regla
de las 22 horas del Decreto 20/2022 —que hoy no está escrita en ninguna parte—.
Las actions pasan a llamar a estas funciones en vez de contener las reglas.
*Criterio:* 4, y es la evidencia central del ramo. *Archivos:* `src/core/*`
nuevo, `quiz.ts`, `progress.ts`, `alerts.ts`, `analytics.ts`.
*Depende de:* A2 (los tests del núcleo necesitan correr en CI).
*Esfuerzo:* medio. Es el ítem con mejor relación entre esfuerzo y rúbrica.

**B2. Repositorios para el núcleo.**
*Qué:* interfaces `RepositorioIntentos`, `RepositorioProgreso`,
`RepositorioCertificados` con implementación Supabase. Solo para el núcleo, no
para los 19 módulos. Al tipar el repositorio desaparecen de paso los `as any` de
esa ruta.
*Criterio:* 2, 4. *Archivos:* `src/core/puertos/*`, `src/infra/supabase/*`,
`quiz.ts`, `progress.ts`, `certificates.ts`.
*Depende de:* B1. *Esfuerzo:* medio.

**B3. Persistir el PDF del certificado.**
*Qué:* generar una vez al emitir, subir a Storage, guardar la URL en `pdf_url`
—que ya existe—, servir desde ahí. Elimina dos fetch externos por descarga y
convierte el camino más lento en una redirección.
*Criterio:* 9, 5 (Cache-Aside / Static Content Hosting). *Archivos:*
`certificates.ts:22`, `:65`, `:105`.
*Depende de:* nada. *Esfuerzo:* medio.

**B4. Observabilidad mínima.**
*Qué:* `instrumentation.ts`, un logger estructurado que reemplace los 34
`console.*`, y una ruta de health check. Sin esto no hay nada que mostrar para el
criterio 6.
*Criterio:* 6, 9. *Archivos:* `src/instrumentation.ts` nuevo,
`src/app/api/health/route.ts` nuevo, sustitución de `console.*`.
*Depende de:* nada. *Esfuerzo:* bajo-medio.
*Nota:* sería la primera `route.ts` del proyecto. Es la excepción justificada:
un health check no puede ser una Server Action.

**B5. Boundaries de error por segmento.**
*Qué:* `error.tsx` en `(dashboard)/`, `admin/`, `admin/eventos/` y
`(dashboard)/cursos/`, para que un fallo en reportes no tumbe la navegación
entera. Quitar la dependencia de `ongalumco.cl` del `error.tsx` raíz.
*Criterio:* 6. *Archivos:* nuevos `error.tsx`, `src/app/error.tsx:22`.
*Depende de:* nada. *Esfuerzo:* bajo.

**B6. Estrategia de caché deliberada.**
*Qué:* auditar los 22 `force-dynamic` y quitarlo donde no haga falta; aplicar
`'use cache'` con `cacheTag` a las lecturas de catálogo (cursos publicados,
sedes, configuración) e invalidar con `revalidateTag` desde las actions que
escriben. Los 91 `revalidatePath` pasan a significar algo.
*Criterio:* 9, 5 (Cache-Aside). *Archivos:* las 22 páginas, `courses.ts`,
`sedes.ts`.
*Depende de:* A2. *Esfuerzo:* medio.

**B7. Paginación en los listados de admin.**
*Qué:* `.range()` en trabajadores, reportes y certificados. Antes de que
`course_progress` cruce las 1000 filas y los reportes empiecen a truncar en
silencio.
*Criterio:* 8, 9. *Archivos:* `admin/trabajadores/page.tsx`,
`admin/reportes/page.tsx`, `admin/certificados/page.tsx`.
*Depende de:* nada. *Esfuerzo:* bajo-medio.

### Bloque C — Criterios que exigen construir, no refactorizar

**C1. Internacionalización.**
*Qué:* como mínimo, centralizar el formateo de fechas y números y extraer los
textos a diccionarios. La decisión de alcance es de equipo — ver D1.
*Criterio:* 1. *Archivos:* transversal (36 páginas, 83 client components).
*Depende de:* D1. *Esfuerzo:* alto.

**C2. Extraer certificación como servicio.**
*Qué:* invertir la dependencia `capacitación → certificación` publicando un
evento de dominio ("curso aprobado") que certificación consuma, en vez de la
llamada directa de `quiz.ts:386` y `progress.ts:172`. Con `pgmq` —disponible, no
instalado— esto además da Queue-Based Load Leveling y desacopla la emisión del
request del trabajador.
*Criterio:* 2, 5, 8, 9. *Archivos:* `quiz.ts`, `progress.ts`, `certificates.ts`,
migración.
*Depende de:* B1, B2, B3. *Esfuerzo:* alto. Es el ítem que más rúbrica cubre de
una vez, y el único que justifica instalar algo nuevo.

**C3. IA (PDF→quiz), si se mantiene en el alcance.**
*Qué:* no existe una línea. Es construcción desde cero, no separación.
*Criterio:* ninguno de los nueve lo exige directamente.
*Depende de:* D4. *Esfuerzo:* alto.
*Recomendación:* sacarlo del alcance del trimestre salvo que D4 diga lo
contrario. Compite por tiempo con B1 y C2, que sí puntúan en la rúbrica.

### Orden sugerido

`A1 → A2 → A5 → A3 → B1 → B4 → B5 → B2 → B3 → A4 → B7 → B6 → C2 → C1`

A1 primero porque es un riesgo abierto. A2 segundo porque todo lo demás se apoya
en poder cambiar código sin romper producción a ciegas. B1 es el centro de
gravedad académico del trimestre.

---

## 10. Decisiones que requieren definición

Preguntas que el código no puede responder.

**D1. ¿Qué significa "internacionalización" para la evaluación?**
El sistema es para una ONG chilena, con trabajadores chilenos, en un solo idioma.
Traducirlo a inglés no tiene uso real. Las opciones son (a) i18n completa con
diccionarios y una segunda lengua, (b) infraestructura de i18n con un solo locale
poblado, demostrando que la arquitectura lo soporta, o (c) argumentar que
personalización de UI —que sí está implementada y es útil— cubre el criterio.
El esfuerzo va de bajo a alto según la respuesta. **Es la pregunta que más cambia
la planificación del trimestre y hay que hacérsela al profesor.**

**D2. `completed_modules`: ¿array o tabla?**
Cerrar el *lost update* admite dos caminos. Una RPC que haga el append atómico es
un parche de bajo costo. Una tabla `module_completions` con
`UNIQUE (user_id, module_id)` elimina la clase entera de error, es la
representación correcta del dominio, encaja con `quiz_attempts` como registro
append-only, y habilita reportes que hoy no se pueden hacer (¿cuándo completó
cada módulo?). Cuesta una migración de datos. **Recomendación: la tabla**, pero
es decisión de equipo porque toca varios lectores.

**D3. ¿Qué puede hacer un `profesor`?**
`requireAdmin` acepta `admin` y `profesor`; los dialectos inline de `events.ts`,
`trabajadores.ts` y `registro.ts` solo aceptan `admin`. Hoy el permiso depende de
qué módulo se toque. Antes de unificar (A4) hay que escribir la matriz de
permisos por rol. No está en ninguna parte del repositorio.

**D4. ¿La IA (PDF→quiz) está en el alcance del trimestre?**
No existe ni una línea. Si es requisito, es un proyecto en sí mismo y compite con
B1 y C2. Si es aspiración, hay que sacarlo del documento de diseño para no
presentar como "candidato a extracción" algo que habría que construir entero.

**D5. ¿Cuál es la regla exacta de las 22 horas del Decreto 20/2022?**
Es el propósito declarado del sistema y no está implementada. `getAnnualTarget()`
lee una meta de `platform_settings` sin unidad declarada. Antes de escribir B1
hace falta saber: ¿22 horas cronológicas o pedagógicas?, ¿por año calendario o
móvil?, ¿los cursos tienen duración en horas —hoy `modules` guarda duración en
minutos solo para video—?, ¿prorratea para quien ingresa a mitad de año?

**D6. ¿Qué política gobierna el uso de `service_role`?**
94 usos. Algunos son inevitables (leer perfiles de otros para el panel admin);
otros parecen inercia (`sedes.ts`, `search.ts`). Definir la regla —por ejemplo:
"solo tras `requireAdmin`, y solo cuando la RLS haga la consulta imposible"—
permite auditar los 94 en vez de discutirlos uno por uno.

**D7. ¿Se instala `pgmq`?**
C2 gana bastante con una cola: desacopla la emisión del certificado del request,
da Queue-Based Load Leveling y sostiene el criterio 5 con algo real. Está
disponible en el proyecto y no instalado. Es la única herramienta nueva que este
levantamiento propone, y solo si C2 entra en el alcance.

**D8. ¿Se despliega el mundo demo en la misma base que producción?**
`is_demo` aparece en 99 sitios del código: es una dimensión de tenencia
implementada por convención, donde cada action nueva debe acordarse de llamar
`courseInScope`/`profileInScope` y nada la obliga. Sumado a que
`reset_demo_world()` es ejecutable por `anon` y a que su job de `pg_cron` está
apagado, conviene decidir si el demo se mantiene así o se mueve a un proyecto
Supabase aparte.
