# Guía de desarrollo — App de Inventario para Panaderías

> Documento técnico para construir la app descrita en [PLAN.md](./PLAN.md).
> Stack: **Flutter + Supabase (PostgreSQL) + Drift (SQLite local)**.

---

## 1. Arquitectura

```
┌──────────────────────────── Teléfono / Tablet ────────────────────────────┐
│                                                                           │
│  UI (Flutter widgets)                                                     │
│     │                                                                     │
│  Estado (Riverpod providers)                                              │
│     │                                                                     │
│  Repositorios ──► Base local Drift (SQLite) ◄──► Cola de sincronización   │
│                                                        │                  │
└────────────────────────────────────────────────────────┼──────────────────┘
                                                         │ HTTPS / WebSocket
┌────────────────────────────── Supabase ────────────────┼──────────────────┐
│  Auth (correo, Google, Apple)                          ▼                  │
│  PostgreSQL + Row Level Security  ◄── Funciones RPC (registrar_venta, …)  │
│  Realtime (cambios de stock en vivo)                                      │
│  Storage (fotos de productos)                                             │
└───────────────────────────────────────────────────────────────────────────┘
```

**Principios**
1. **Offline-first para la caja**: el cajero siempre escribe en la base local; la sincronización ocurre en segundo plano.
2. **El inventario es un libro de movimientos**: el stock nunca se edita directamente, se calcula sumando movimientos (ingresos, ventas, mermas, ajustes). Así hay auditoría completa.
3. **IDs generados en el cliente (UUID v4)**: permiten crear registros sin conexión y reenviarlos sin duplicar (idempotencia).
4. **Seguridad en el servidor**: los permisos por rol se validan con RLS en PostgreSQL, no solo en la app.

---

## 2. Requisitos del entorno

| Herramienta | Versión sugerida |
|---|---|
| Flutter SDK | estable más reciente (3.x) |
| Dart | incluido con Flutter |
| Android Studio + SDK | último, con el nivel de API exigido por Google Play |
| Xcode (solo iOS, en macOS) | último estable |
| Supabase CLI | último |
| Docker (para Supabase local) | opcional, recomendado |
| Git | cualquiera reciente |

```bash
flutter doctor            # verificar instalación
npm i -g supabase         # o: brew install supabase/tap/supabase
```

---

## 3. Estructura del repositorio

```
Panaderia/
├── PLAN.md
├── DESARROLLO.md
├── README.md
├── app/                          # Proyecto Flutter
│   ├── lib/
│   │   ├── main.dart
│   │   ├── app.dart              # MaterialApp, router, tema
│   │   ├── core/
│   │   │   ├── config/           # env, constantes
│   │   │   ├── db/               # Drift: tablas locales, DAOs
│   │   │   ├── sync/             # cola y motor de sincronización
│   │   │   ├── router/           # go_router
│   │   │   ├── theme/
│   │   │   └── utils/            # formateo CLP, fechas, validadores
│   │   ├── features/
│   │   │   ├── auth/             # login, registro, borrar cuenta
│   │   │   ├── panaderia/        # crear, unirse con código, miembros
│   │   │   ├── productos/        # catálogo
│   │   │   ├── inventario/       # ingresos, mermas, ajustes, stock
│   │   │   ├── caja/             # turnos y pantalla de venta
│   │   │   ├── reportes/         # reportes, CSV, PDF
│   │   │   ├── donaciones/       # compras integradas
│   │   │   └── ajustes/          # perfil, permisos, legal
│   │   │       # cada feature: data/ (repositorio), domain/ (modelos), ui/ (pantallas)
│   │   └── l10n/                 # textos (es por defecto, en opcional)
│   ├── test/
│   ├── integration_test/
│   ├── android/
│   ├── ios/
│   └── pubspec.yaml
├── supabase/
│   ├── migrations/               # SQL versionado (esquema, RLS, funciones)
│   ├── seed.sql                  # datos de ejemplo para desarrollo / revisor de Apple
│   └── config.toml
├── docs/
│   ├── privacidad.md             # publicado con GitHub Pages
│   ├── terminos.md
│   └── borrar-cuenta.md          # URL de solicitud de borrado (Google Play)
└── .github/workflows/
    ├── ci.yml                    # analyze + test en cada PR
    └── release.yml               # build AAB/IPA al crear un tag
```

---

## 4. Dependencias de Flutter (`pubspec.yaml`)

| Paquete | Uso |
|---|---|
| `supabase_flutter` | Auth, base de datos, realtime, storage |
| `flutter_riverpod` / `riverpod_annotation` | Estado |
| `go_router` | Navegación con protección por rol |
| `drift`, `drift_flutter`, `sqlite3_flutter_libs` | Base local offline |
| `connectivity_plus` | Detectar conexión para sincronizar |
| `uuid` | IDs en el cliente |
| `freezed`, `json_serializable` | Modelos inmutables |
| `intl` | Formato de moneda (CLP) y fechas en español |
| `in_app_purchase` | Donaciones (Google Play Billing / StoreKit) |
| `google_sign_in`, `sign_in_with_apple` | Login social |
| `mobile_scanner` / `qr_flutter` | Leer y mostrar QR de invitación |
| `fl_chart` | Gráficos de reportes |
| `pdf`, `printing`, `csv`, `share_plus` | Exportar reportes |
| `sentry_flutter` | Reporte de errores |
| Dev: `build_runner`, `drift_dev`, `riverpod_generator`, `mocktail`, `flutter_lints` | Generación de código y pruebas |

---

## 5. Modelo de datos (PostgreSQL / Supabase)

### 5.1 Diagrama

```
auth.users ─┐
            ▼
         perfiles 1───* miembros *───1 panaderias
                            │              │
                            │              ├──* productos ──* movimientos_inventario
                            │              ├──* turnos_caja ──* ventas ──* venta_items
                            │              └──* invitaciones
                            └── rol: dueno | admin | cajero
```

### 5.2 Migración inicial — `supabase/migrations/0001_esquema.sql`

```sql
-- Tipos
create type rol_miembro      as enum ('dueno', 'admin', 'cajero');
create type unidad_venta     as enum ('kg', 'unidad');
create type tipo_movimiento  as enum ('ingreso', 'venta', 'merma', 'ajuste', 'anulacion');
create type motivo_merma     as enum ('quemado', 'vencido', 'danado', 'consumo_interno', 'donado', 'otro');
create type medio_pago       as enum ('efectivo', 'debito', 'credito', 'transferencia', 'otro');
create type estado_venta     as enum ('completada', 'anulada');

-- Perfil público del usuario
create table perfiles (
  id          uuid primary key references auth.users on delete cascade,
  nombre      text not null,
  creado_en   timestamptz not null default now()
);

create table panaderias (
  id                      uuid primary key default gen_random_uuid(),
  nombre                  text not null,
  direccion               text,
  moneda                  char(3) not null default 'CLP',
  zona_horaria            text not null default 'America/Santiago',
  cajero_puede_mermas     boolean not null default true,
  cajero_puede_rebajar    boolean not null default false,
  rebaja_maxima_cajero    numeric(5,2) not null default 10,   -- % máximo
  creado_en               timestamptz not null default now()
);

create table miembros (
  panaderia_id  uuid not null references panaderias on delete cascade,
  usuario_id    uuid not null references perfiles on delete cascade,
  rol           rol_miembro not null,
  activo        boolean not null default true,
  creado_en     timestamptz not null default now(),
  primary key (panaderia_id, usuario_id)
);

create table invitaciones (
  codigo        char(6) primary key,                 -- ej. 'K7P2QX'
  panaderia_id  uuid not null references panaderias on delete cascade,
  rol           rol_miembro not null check (rol <> 'dueno'),
  creada_por    uuid not null references perfiles,
  expira_en     timestamptz not null default now() + interval '7 days',
  usos_max      int not null default 1,
  usos          int not null default 0
);

create table productos (
  id            uuid primary key,                    -- generado en el cliente
  panaderia_id  uuid not null references panaderias on delete cascade,
  nombre        text not null,
  categoria     text not null default 'pan',
  unidad        unidad_venta not null default 'kg',
  precio        numeric(12,2) not null check (precio >= 0),  -- por kg o por unidad
  stock_minimo  numeric(12,3) not null default 0,
  foto_url      text,
  activo        boolean not null default true,
  actualizado_en timestamptz not null default now()
);

create table turnos_caja (
  id               uuid primary key,
  panaderia_id     uuid not null references panaderias on delete cascade,
  cajero_id        uuid not null references perfiles,
  abierto_en       timestamptz not null default now(),
  cerrado_en       timestamptz,
  monto_inicial    numeric(12,2) not null default 0,
  efectivo_contado numeric(12,2),                    -- al cerrar
  nota_cierre      text
);

create table ventas (
  id             uuid primary key,
  panaderia_id   uuid not null references panaderias on delete cascade,
  turno_id       uuid references turnos_caja,
  cajero_id      uuid not null references perfiles,
  fecha          timestamptz not null,               -- hora real en el dispositivo
  subtotal       numeric(12,2) not null,
  descuento      numeric(12,2) not null default 0,   -- rebaja a nivel de venta
  total          numeric(12,2) not null,
  medio_pago     medio_pago not null,
  estado         estado_venta not null default 'completada',
  anulada_por    uuid references perfiles,
  motivo_anulacion text,
  sincronizado_en timestamptz not null default now()
);

create table venta_items (
  id              uuid primary key,
  venta_id        uuid not null references ventas on delete cascade,
  producto_id     uuid not null references productos,
  cantidad        numeric(12,3) not null check (cantidad > 0),   -- kg o unidades
  precio_unitario numeric(12,2) not null,            -- precio al momento de la venta
  descuento       numeric(12,2) not null default 0,  -- rebaja por ítem
  total           numeric(12,2) not null
);

-- Libro de inventario: ÚNICA fuente de verdad del stock
create table movimientos_inventario (
  id            uuid primary key,
  panaderia_id  uuid not null references panaderias on delete cascade,
  producto_id   uuid not null references productos,
  tipo          tipo_movimiento not null,
  cantidad      numeric(12,3) not null,              -- positiva = entra, negativa = sale
  motivo_merma  motivo_merma,
  nota          text,
  venta_id      uuid references ventas,
  usuario_id    uuid not null references perfiles,
  fecha         timestamptz not null,
  check (tipo <> 'merma' or motivo_merma is not null),
  check (tipo <> 'ajuste' or nota is not null)
);

create index on movimientos_inventario (panaderia_id, producto_id);
create index on movimientos_inventario (panaderia_id, fecha);
create index on ventas (panaderia_id, fecha);
create index on ventas (panaderia_id, cajero_id, fecha);

-- Stock actual
create view stock_actual with (security_invoker = true) as
select p.panaderia_id, p.id as producto_id, p.nombre, p.unidad, p.stock_minimo,
       coalesce(sum(m.cantidad), 0) as stock,
       coalesce(sum(m.cantidad), 0) <= p.stock_minimo as stock_bajo
from productos p
left join movimientos_inventario m on m.producto_id = p.id
where p.activo
group by p.id;
```

### 5.3 Seguridad por filas — `0002_rls.sql`

```sql
-- Helpers (security definer para evitar recursión en políticas)
create function mi_rol(p_panaderia uuid) returns rol_miembro
language sql stable security definer set search_path = public as $$
  select rol from miembros
  where panaderia_id = p_panaderia and usuario_id = auth.uid() and activo
$$;

create function es_admin(p_panaderia uuid) returns boolean
language sql stable as $$ select mi_rol(p_panaderia) in ('dueno','admin') $$;

alter table panaderias             enable row level security;
alter table miembros               enable row level security;
alter table invitaciones           enable row level security;
alter table productos              enable row level security;
alter table turnos_caja            enable row level security;
alter table ventas                 enable row level security;
alter table venta_items            enable row level security;
alter table movimientos_inventario enable row level security;
alter table perfiles               enable row level security;

-- Panadería: la ven sus miembros; la editan admins
create policy ver_panaderia    on panaderias for select using (mi_rol(id) is not null);
create policy editar_panaderia on panaderias for update using (es_admin(id));

-- Productos: todos los miembros leen; solo admins escriben
create policy ver_productos    on productos for select using (mi_rol(panaderia_id) is not null);
create policy admin_productos  on productos for all    using (es_admin(panaderia_id))
                                                       with check (es_admin(panaderia_id));

-- Ventas: admins ven todo; cajero solo las suyas
create policy ver_ventas on ventas for select using (
  es_admin(panaderia_id) or cajero_id = auth.uid()
);
-- (las inserciones de ventas se hacen solo vía la función registrar_venta)

-- Movimientos: admins ven todo; cajeros pueden insertar mermas si está permitido
create policy ver_movimientos on movimientos_inventario for select
  using (mi_rol(panaderia_id) is not null);
create policy admin_movimientos on movimientos_inventario for insert
  with check (es_admin(panaderia_id) and usuario_id = auth.uid());
create policy cajero_mermas on movimientos_inventario for insert
  with check (
    mi_rol(panaderia_id) = 'cajero' and tipo = 'merma' and usuario_id = auth.uid()
    and (select cajero_puede_mermas from panaderias where id = panaderia_id)
  );
-- Sin políticas de UPDATE/DELETE: los movimientos son inmutables.

-- (completar políticas análogas para miembros, invitaciones, turnos_caja, venta_items, perfiles)
```

### 5.4 Funciones RPC — `0003_funciones.sql`

**Registrar venta (atómica e idempotente).** Recibe la venta completa en JSON; si el `id` ya existe (reintento de sincronización), no hace nada.

```sql
create function registrar_venta(p jsonb) returns void
language plpgsql security definer set search_path = public as $$
declare
  v_panaderia uuid := (p->>'panaderia_id')::uuid;
  v_item jsonb;
begin
  if mi_rol(v_panaderia) is null then
    raise exception 'No pertenece a esta panadería';
  end if;

  insert into ventas (id, panaderia_id, turno_id, cajero_id, fecha, subtotal,
                      descuento, total, medio_pago)
  values ((p->>'id')::uuid, v_panaderia, (p->>'turno_id')::uuid, auth.uid(),
          (p->>'fecha')::timestamptz, (p->>'subtotal')::numeric,
          (p->>'descuento')::numeric, (p->>'total')::numeric,
          (p->>'medio_pago')::medio_pago)
  on conflict (id) do nothing;

  if not found then return; end if;   -- ya estaba sincronizada

  -- TODO: validar descuento contra rebaja_maxima_cajero si el rol es cajero

  for v_item in select * from jsonb_array_elements(p->'items') loop
    insert into venta_items (id, venta_id, producto_id, cantidad,
                             precio_unitario, descuento, total)
    values ((v_item->>'id')::uuid, (p->>'id')::uuid, (v_item->>'producto_id')::uuid,
            (v_item->>'cantidad')::numeric, (v_item->>'precio_unitario')::numeric,
            (v_item->>'descuento')::numeric, (v_item->>'total')::numeric);

    insert into movimientos_inventario (id, panaderia_id, producto_id, tipo,
                                        cantidad, venta_id, usuario_id, fecha)
    values (gen_random_uuid(), v_panaderia, (v_item->>'producto_id')::uuid, 'venta',
            -(v_item->>'cantidad')::numeric, (p->>'id')::uuid, auth.uid(),
            (p->>'fecha')::timestamptz);
  end loop;
end $$;
```

**Otras funciones a implementar**

| Función | Qué hace |
|---|---|
| `crear_panaderia(nombre, direccion)` | Crea la panadería y agrega al usuario como `dueno`. |
| `crear_invitacion(panaderia_id, rol)` | Solo admins. Genera código de 6 caracteres sin ambigüedades (sin 0/O, 1/I). |
| `unirse_con_codigo(codigo)` | Valida expiración y usos; agrega al usuario como miembro. |
| `anular_venta(venta_id, motivo)` | Solo admins. Marca la venta como anulada y crea movimientos `anulacion` que devuelven el stock. |
| `cerrar_turno(turno_id, efectivo_contado, nota)` | Cierra el turno y devuelve el resumen (esperado vs. contado). |
| `reporte_ventas(panaderia_id, desde, hasta, agrupar, cajero_id?)` | Ver 5.5. |
| `eliminar_mi_cuenta()` | Borra datos personales; conserva ventas anonimizadas para la panadería (ver sección 9). |

### 5.5 Reportes

Una sola función cubre diario, semanal, mensual, por cajero y general:

```sql
create function reporte_ventas(
  p_panaderia uuid, p_desde date, p_hasta date,
  p_agrupar text default 'dia',          -- 'dia' | 'semana' | 'mes'
  p_cajero uuid default null             -- null = general
) returns table (periodo date, cajero_id uuid, cajero text,
                 n_ventas bigint, total numeric, descuentos numeric)
language sql stable security invoker as $$
  select date_trunc(case p_agrupar when 'semana' then 'week'
                                   when 'mes'    then 'month'
                                   else 'day' end,
                    v.fecha at time zone pa.zona_horaria)::date,
         v.cajero_id, pr.nombre,
         count(*), sum(v.total), sum(v.descuento)
  from ventas v
  join panaderias pa on pa.id = v.panaderia_id
  join perfiles  pr on pr.id = v.cajero_id
  where v.panaderia_id = p_panaderia
    and v.estado = 'completada'
    and (v.fecha at time zone pa.zona_horaria)::date between p_desde and p_hasta
    and (p_cajero is null or v.cajero_id = p_cajero)
  group by 1, 2, 3
  order by 1, 3
$$;
```

Reportes adicionales (vistas o funciones similares):
- **Top productos**: `venta_items` agrupado por `producto_id` (cantidad y monto).
- **Mermas**: `movimientos_inventario where tipo = 'merma'` agrupado por motivo y producto, valorizado a precio actual.
- **Medios de pago**: `ventas` agrupado por `medio_pago`.
- **Cuadre de turno**: `monto_inicial + ventas en efectivo` vs. `efectivo_contado`.

> Los cálculos siempre usan la **zona horaria de la panadería** (`America/Santiago`) para que "ventas del día" corresponda al día local.

---

## 6. Sincronización offline

**Tablas locales (Drift)** reflejan `productos`, `stock_actual` (caché), `turnos_caja`, `ventas`, `venta_items`, `movimientos_inventario`, más:

```dart
class ColaSync extends Table {
  TextColumn get id => text()();            // UUID del registro
  TextColumn get operacion => text()();     // 'registrar_venta', 'merma', 'abrir_turno', ...
  TextColumn get payload => text()();       // JSON
  IntColumn get intentos => integer().withDefault(const Constant(0))();
  DateTimeColumn get creadoEn => dateTime()();
  TextColumn get ultimoError => text().nullable()();
  @override Set<Column> get primaryKey => {id};
}
```

**Flujo**
1. El cajero confirma la venta → se guarda en Drift + se agrega a `ColaSync` en **una sola transacción local**. La UI responde al instante.
2. El `MotorSync` se ejecuta: al recuperar conexión, cada 30 s mientras haya pendientes, y al volver la app a primer plano.
3. Procesa la cola **en orden (FIFO)**, llamando la RPC correspondiente. Como las RPC son idempotentes, reintentar es seguro.
4. Éxito → elimina de la cola. Error de red → reintento con espera exponencial. Error de validación (ej. sin permiso) → marca el error y lo muestra al admin.
5. **Bajada de datos**: catálogo y stock se refrescan con Supabase Realtime cuando hay conexión; sin conexión, se usa la última copia local y el stock local se ajusta con las ventas pendientes.

**Conflictos**
- Ventas y movimientos: no hay conflictos (solo se insertan, nunca se editan).
- Productos/precios: "gana el último" usando `actualizado_en`; solo admins los editan.
- Stock negativo: se permite (puede ocurrir con ventas offline simultáneas) y se resalta para que el admin haga un ajuste.

Indicador visible en la caja: 🟢 sincronizado · 🟡 N ventas pendientes · 🔴 error.

---

## 7. Pantallas principales

| Pantalla | Rol | Notas |
|---|---|---|
| Bienvenida / Login / Registro | Todos | Correo, Google, Apple (iOS). |
| Elegir o crear panadería / Unirse con código | Todos | Escanear QR o escribir código. |
| Inicio (panel) | Admin | Ventas de hoy, stock bajo, mermas del día, turnos abiertos. |
| **Caja** | Cajero / Admin | Grilla de productos con botones grandes; teclado numérico para kg; carrito; cobrar. Usable con una mano. |
| Abrir / cerrar turno | Cajero | Monto inicial; conteo final con diferencia. |
| Productos (lista / editar) | Admin | |
| Inventario: stock, ingreso, merma, ajuste, historial | Admin (merma también cajero) | Ingreso masivo: lista con cantidades para registrar la producción de la mañana en un paso. |
| Reportes | Admin (cajero: solo "mis ventas de hoy") | Filtros: período, cajero, producto. Exportar CSV/PDF. |
| Miembros e invitaciones | Admin | Cambiar rol, desactivar. |
| Ajustes | Todos | Perfil, permisos de cajero, privacidad, términos, **borrar cuenta**. |
| Apoya el proyecto | Todos | Donaciones IAP. |

**Diseño:** Material 3, tema cálido (tonos trigo/café), modo claro y oscuro, tipografía grande en la caja, montos en formato `$1.990` (CLP sin decimales).

---

## 8. Donaciones con compras integradas

1. **Crear productos consumibles** con los mismos IDs en ambas tiendas:
   `donacion_1`, `donacion_3`, `donacion_5`, `donacion_10`.
   - Google Play Console → Monetizar → Productos → Productos integrados.
   - App Store Connect → Compras dentro de la app → Consumible.
2. Inscribirse en el **programa de 15 %** de cada tienda (Google: nivel de tarifa de servicio; Apple: App Store Small Business Program).
3. Implementación con `in_app_purchase`:

```dart
const idsDonacion = {'donacion_1', 'donacion_3', 'donacion_5', 'donacion_10'};

final respuesta = await InAppPurchase.instance.queryProductDetails(idsDonacion);
// Mostrar respuesta.productDetails con su precio localizado (p.price)

await InAppPurchase.instance.buyConsumable(
  purchaseParam: PurchaseParam(productDetails: producto),
);

// Escuchar InAppPurchase.instance.purchaseStream:
//   status == purchased → completePurchase(compra) y mostrar "¡Gracias!"
```

4. Texto en la pantalla: *"La app es y seguirá siendo gratuita. Tu aporte es voluntario, no desbloquea funciones y ayuda a pagar el servidor y nuevas mejoras."*
5. **No** incluir enlaces a PayPal/Ko-fi/transferencias dentro de la app (riesgo de rechazo, especialmente en iOS). Esos enlaces van en el sitio web y en GitHub.

---

## 9. Privacidad y borrado de cuenta

- Datos personales mínimos: nombre y correo. Los datos de ventas pertenecen a la panadería.
- `eliminar_mi_cuenta()`:
  1. Si el usuario es el único `dueno` de una panadería, pedir transferir propiedad o eliminar la panadería.
  2. Reemplazar `perfiles.nombre` por "Usuario eliminado" (las ventas históricas se conservan para los reportes del negocio, sin datos personales).
  3. Eliminar al usuario de `auth.users` (desde una Edge Function con la service role key).
- `docs/borrar-cuenta.md` publicado en GitHub Pages explica cómo borrar la cuenta desde la app o por correo (URL exigida por Google Play).
- Todo el tráfico es HTTPS; la service role key **nunca** va en la app, solo en Edge Functions.

---

## 10. Configuración inicial (paso a paso)

```bash
# 1. Crear proyecto Flutter
flutter create --org cl.panaderia --project-name panaderia app
cd app
flutter pub add supabase_flutter flutter_riverpod riverpod_annotation go_router \
  drift drift_flutter sqlite3_flutter_libs connectivity_plus uuid intl \
  freezed_annotation json_annotation in_app_purchase google_sign_in \
  sign_in_with_apple mobile_scanner qr_flutter fl_chart pdf printing csv \
  share_plus sentry_flutter
flutter pub add -d build_runner drift_dev riverpod_generator freezed \
  json_serializable mocktail flutter_lints

# 2. Supabase local
cd ..
supabase init
supabase start                       # levanta Postgres, Auth, etc. en Docker
supabase migration new esquema       # pegar SQL de la sección 5
supabase db reset                    # aplica migraciones + seed.sql

# 3. Ejecutar la app contra Supabase local
cd app
flutter run --dart-define=SUPABASE_URL=http://10.0.2.2:54321 \
            --dart-define=SUPABASE_ANON_KEY=<clave-anon-local>

# 4. Generar código (Drift, Riverpod, Freezed)
dart run build_runner watch -d
```

Para producción: crear el proyecto en supabase.com, `supabase link --project-ref <ref>` y `supabase db push`.

> La clave **anon** de Supabase puede ir en la app (la seguridad la da RLS). Pasarla con `--dart-define`, no subirla al repositorio.

---

## 11. Calidad y pruebas

| Tipo | Qué cubrir | Herramienta |
|---|---|---|
| Unitarias | Cálculo de totales, descuentos, redondeo CLP, motor de sincronización | `flutter test`, `mocktail` |
| Widgets | Caja: agregar productos, aplicar rebaja, cobrar | `flutter test` |
| Integración | Flujo completo offline → online | `integration_test` |
| Base de datos | Políticas RLS (un cajero no ve ventas ajenas, no puede editar precios) | `pgTAP` vía `supabase test db` |

**CI (`.github/workflows/ci.yml`)** en cada PR:
```yaml
- flutter analyze
- dart format --set-exit-if-changed .
- flutter test
- supabase db start && supabase test db
```

**Convenciones**
- Código y nombres de dominio en español (`venta`, `merma`, `turno`); términos técnicos en inglés si son estándar (`repository`, `provider`).
- Commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`.
- Ramas: `main` (estable) ← `feat/<nombre>` vía pull request.
- Versionado semántico: `1.0.0+1` en `pubspec.yaml` (el número tras `+` sube en cada envío a tienda).

---

## 12. Plan de sprints (2 semanas c/u, medio tiempo)

### Sprint 0 — Fundaciones
- [ ] Crear proyecto Flutter y estructura de carpetas.
- [ ] Supabase local + migración 0001.
- [ ] CI con analyze y test.
- [ ] Tema, router, formato CLP.
- [ ] Abrir cuenta Google Play (iniciar verificación de identidad).

### Sprint 1 — Cuentas y panaderías
- [ ] Login/registro (correo + Google).
- [ ] `crear_panaderia`, `crear_invitacion`, `unirse_con_codigo`.
- [ ] Pantalla de miembros; QR de invitación.
- [ ] Políticas RLS + pruebas pgTAP.
- [ ] Router con protección por rol.

### Sprint 2 — Catálogo e inventario
- [ ] CRUD de productos con foto (Supabase Storage).
- [ ] Ingreso (individual y masivo), merma, ajuste.
- [ ] Vista de stock con alertas de stock bajo.
- [ ] Historial de movimientos con filtros.

### Sprint 3 — Caja (online)
- [ ] Abrir/cerrar turno.
- [ ] Pantalla de venta, carrito, kg/unidades, medios de pago.
- [ ] Rebajas por ítem y por venta, con límites para cajeros.
- [ ] `registrar_venta`, `anular_venta`.

### Sprint 4 — Offline y sincronización
- [ ] Tablas Drift y `ColaSync`.
- [ ] `MotorSync` con reintentos y indicador de estado.
- [ ] Realtime para catálogo/stock.
- [ ] Prueba de integración: 50 ventas offline → sincronizar sin duplicados.

### Sprint 5 — Reportes
- [ ] `reporte_ventas` y reportes de top productos, mermas, medios de pago.
- [ ] Pantallas con gráficos y filtros (día/semana/mes, cajero/general).
- [ ] Exportar CSV y PDF; compartir.
- [ ] Panel de inicio del administrador.

### Sprint 6 — Cumplimiento y donaciones
- [ ] Donaciones IAP (Android primero).
- [ ] Borrar cuenta (app + Edge Function + página web).
- [ ] Política de privacidad y términos en GitHub Pages.
- [ ] Sentry; ícono, splash, capturas de tienda.
- [ ] Datos de demostración (`seed.sql`) para revisores.

### Sprint 7 — Prueba cerrada y lanzamiento
- [ ] Subir AAB a prueba cerrada; reclutar ≥ 12 testers por 14 días.
- [ ] Corregir errores reportados.
- [ ] Completar Data Safety, clasificación de contenido, ficha de tienda.
- [ ] Solicitar acceso a producción y publicar en Google Play.
- [ ] (Siguiente) Sign in with Apple, IAP de iOS, envío a App Store.

---

## 13. Checklist de publicación

**Antes de cada versión**
- [ ] `flutter analyze` y `flutter test` sin errores.
- [ ] Probado en un Android de gama baja (las panaderías no suelen usar teléfonos caros).
- [ ] Probado sin internet: vender, cerrar turno, volver a conectar.
- [ ] Subir `version` en `pubspec.yaml`.
- [ ] Notas de la versión en español.

**Build Android**
```bash
flutter build appbundle --release \
  --dart-define=SUPABASE_URL=... --dart-define=SUPABASE_ANON_KEY=...
# Resultado: build/app/outputs/bundle/release/app-release.aab
```
- Firmar con la clave de subida (`key.properties`, **nunca** al repositorio) y activar Play App Signing.

**Build iOS**
```bash
flutter build ipa --release --dart-define=...
# Subir con Xcode Organizer o Transporter
```

---

## 14. Referencias

- Flutter: https://docs.flutter.dev
- Supabase + Flutter: https://supabase.com/docs/reference/dart
- Supabase RLS: https://supabase.com/docs/guides/database/postgres/row-level-security
- Drift: https://drift.simonbinder.eu
- in_app_purchase: https://pub.dev/packages/in_app_purchase
- Políticas de Google Play: https://play.google.com/about/developer-content-policy/
- App Store Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
