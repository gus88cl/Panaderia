# Plan del proyecto — App de Inventario para Panaderías

> Documento de planificación: qué se va a construir, para quién, con qué tecnología, cuánto cuesta, cuánto demora y qué tan factible es.
> El detalle técnico (modelo de datos, estructura del código, sprints) está en [DESARROLLO.md](./DESARROLLO.md).

---

## 1. Resumen

Una app **gratuita** para Android (Google Play) e iOS (App Store) que permite a una panadería llevar el control de:

- **Ingreso de panes y productos** (producción del día o compras a proveedor).
- **Ventas** registradas por cada cajero.
- **Pérdidas / mermas** (pan quemado, vencido, regalado, dañado).
- **Rebajas y ajustes de inventario** (descuentos de precio y correcciones de stock).
- **Reportes** de ventas diarios, semanales y mensuales, por cajero y generales.

Varios usuarios se conectan a la misma panadería con distintos **roles** (administrador y cajero), y los datos se sincronizan en tiempo real entre dispositivos.

La app se financia con **donaciones voluntarias dentro de la app** ("invítanos un café"), sin publicidad y sin funciones bloqueadas.

---

## 2. Factibilidad

### Veredicto: **ALTA** ✅

| Aspecto | Evaluación | Comentario |
|---|---|---|
| Técnica | Alta | Es una app CRUD + reportes. No requiere tecnología experimental. Existen herramientas maduras y gratuitas (Flutter + Supabase). |
| Económica | Alta | Puede operar con **USD 25 únicos** (Google Play) y backend en plan gratuito al inicio. iOS agrega **USD 99/año**. |
| Publicación en tiendas | Alta, con requisitos | Hay que cumplir políticas (privacidad, borrado de cuenta, pagos). Están documentados abajo y son abordables. |
| Tiempo | Media-alta | MVP en **10–12 semanas** con 1 desarrollador a medio tiempo; **6–8 semanas** a tiempo completo. |
| Sostenibilidad | Media | Las donaciones suelen cubrir costos de servidor, pero rara vez un sueldo. Ver sección 8. |

### Riesgos principales y mitigación

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Internet inestable en el local | Alto: el cajero no puede vender | Modo **offline-first**: ventas se guardan en el teléfono y se sincronizan al volver la conexión. |
| Plan gratuito del backend se queda corto | Medio | Diseño multi-panadería eficiente; si crece, plan Pro de Supabase (~USD 25/mes) financiado con donaciones. |
| Rechazo en tiendas por donaciones | Medio | Usar **compras integradas (IAP) consumibles** de Google/Apple para las donaciones; no usar links externos de pago dentro de la app. |
| Requisito de Google: prueba cerrada con testers | Medio | Cuentas personales nuevas deben hacer una prueba cerrada (actualmente **12 testers durante 14 días**) antes de publicar. Reclutar panaderías y conocidos desde el inicio. |
| Confusión con boleta electrónica (SII) | Medio | Dejar claro que la app es de **control interno**, no reemplaza la boleta/factura electrónica. |
| Abandono del proyecto | Medio | Código abierto en GitHub, documentado, para que otros puedan contribuir. |

---

## 3. Usuarios y roles

| Rol | Puede hacer |
|---|---|
| **Dueño** (creador de la panadería) | Todo lo del administrador + eliminar la panadería, transferir propiedad, gestionar administradores. |
| **Administrador** (uno o varios) | Crear/editar productos y precios, registrar ingresos, mermas y ajustes, invitar/desactivar cajeros, ver todos los reportes, aplicar rebajas. |
| **Cajero** | Abrir/cerrar su turno de caja, registrar ventas, registrar mermas (si el admin lo permite), ver solo sus propias ventas del día. |

**Conexión de usuarios:** el administrador genera un **código de invitación** (o QR) de 6 caracteres; el cajero instala la app, crea su cuenta e ingresa el código para unirse a la panadería.

Una persona puede pertenecer a más de una panadería (ej. dueño con dos locales).

---

## 4. Funcionalidades

### 4.1 MVP (versión 1.0 — lo mínimo para publicar)

**Cuenta y panadería**
- Registro / inicio de sesión con correo + contraseña y con Google (en iOS también "Iniciar sesión con Apple", obligatorio si hay login social).
- Crear panadería (nombre, dirección, moneda — por defecto CLP).
- Invitar usuarios por código/QR y asignar rol.
- Eliminar cuenta desde la app (requisito de ambas tiendas).

**Catálogo de productos**
- Productos con nombre, categoría (pan, pastelería, bebidas, otros), unidad de venta (**kg** o **unidad**), precio, foto opcional y stock mínimo.
- Ejemplos precargados opcionales: marraqueta, hallulla, pan amasado, dobladitas, coliza, etc.

**Inventario**
- **Ingreso**: producción propia o compra, con cantidad, fecha y nota.
- **Venta**: descuenta stock automáticamente.
- **Merma / pérdida**: cantidad + motivo (quemado, vencido, dañado, consumo interno, donado, otro).
- **Ajuste**: corrección manual tras un conteo físico (con motivo obligatorio).
- Stock actual en tiempo real y alerta de stock bajo.
- Historial de movimientos (quién, qué, cuándo) — nada se borra, todo queda auditado.

**Ventas (pantalla de caja)**
- Pantalla rápida con botones grandes por producto; ingreso de peso en kg o cantidad.
- Carrito con múltiples productos, total, medio de pago (efectivo, débito, crédito, transferencia).
- **Rebajas**: descuento por producto o por venta (monto o %), con permiso configurable para cajeros.
- Anular venta (solo admin o con aprobación), queda registrada.
- **Turnos de caja**: apertura con monto inicial, cierre con conteo de efectivo y diferencia.
- Funciona **sin internet** y sincroniza después.

**Reportes**
- Ventas **diarias, semanales, mensuales** y por rango de fechas.
- Ventas **por cajero** y **generales**.
- Productos más vendidos, mermas por motivo, ventas por medio de pago.
- Resumen de cierre de turno.
- Exportar a **CSV** (para Excel) y compartir **PDF**.

**Donaciones**
- Pantalla "Apoya el proyecto" con 3–4 montos (ej. USD 1, 3, 5, 10) mediante compra integrada consumible.
- Nunca bloquea funciones; solo un agradecimiento (insignia opcional).

### 4.2 Versión 1.x (después del lanzamiento)
- Proveedores y costo de insumos → **margen de ganancia** por producto.
- Recetas: descontar harina, levadura, etc. al registrar producción.
- Gráficos en el panel de inicio.
- Lector de código de barras para productos envasados.
- Impresión de comprobante interno por Bluetooth (impresoras térmicas de 58 mm).
- Notificaciones push (stock bajo, cierre de caja con diferencia).
- Modo multi-sucursal con reportes consolidados.
- Versión web para el administrador (mismo código Flutter).

### 4.3 Fuera de alcance
- Emisión de boleta/factura electrónica ante el SII (requiere certificación y proveedor autorizado).
- Procesamiento de pagos con tarjeta (se registra el medio de pago, no se cobra).
- Contabilidad.

---

## 5. Tecnología recomendada

| Capa | Elección | Por qué |
|---|---|---|
| App móvil | **Flutter (Dart)** | Un solo código para Android, iOS y web. Buen rendimiento, gratis, gran comunidad. |
| Backend | **Supabase** | PostgreSQL + autenticación + tiempo real + almacenamiento. Plan gratuito generoso. Seguridad por filas (RLS) ideal para multi-panadería y roles. |
| Base local (offline) | **Drift (SQLite)** | Guarda ventas y catálogo en el teléfono; cola de sincronización. |
| Estado | **Riverpod** | Simple, testeable, estándar en Flutter. |
| Donaciones | **in_app_purchase** (oficial de Flutter) | Usa Google Play Billing y Apple StoreKit, cumpliendo las políticas de pago. |
| Reportes PDF | **pdf** + **printing** | Generación local, sin costo. |
| CI/CD | **GitHub Actions** + **Fastlane** (opcional) | Compilar, probar y subir builds automáticamente. |
| Errores | **Sentry** (plan gratuito) | Detectar fallos en producción. |

**Alternativa:** React Native (Expo) + Supabase, si el desarrollador domina JavaScript/TypeScript. El plan es igual de válido; solo cambia la capa de la app.

---

## 6. Requisitos de publicación

### Google Play
- [ ] Cuenta de desarrollador: **USD 25 pago único**.
- [ ] Verificación de identidad del desarrollador (documento).
- [ ] **Prueba cerrada** obligatoria para cuentas personales nuevas: actualmente 12 testers activos durante 14 días seguidos.
- [ ] Apuntar al **nivel de API de Android vigente** exigido por Google (revisar al momento de publicar).
- [ ] Formato **AAB** (Android App Bundle) firmado.
- [ ] **Política de privacidad** pública (URL), enlazada en la ficha y dentro de la app.
- [ ] Formulario de **Seguridad de los datos** (Data Safety): declarar correo, nombre, datos de ventas, cifrado en tránsito, posibilidad de borrado.
- [ ] **Borrado de cuenta**: opción dentro de la app **y** URL web para solicitarlo.
- [ ] Clasificación de contenido (cuestionario IARC).
- [ ] Declaración de público objetivo (no dirigida a niños).
- [ ] Ficha: icono 512×512, gráfico destacado 1024×500, mínimo 2 capturas, descripción corta y larga.
- [ ] Donaciones mediante **Google Play Billing** (productos consumibles).

### App Store (iOS)
- [ ] Apple Developer Program: **USD 99 por año** (existe exención de cuota para algunas organizaciones sin fines de lucro).
- [ ] Mac con Xcode para compilar (o servicio en la nube como Codemagic, con plan gratuito limitado).
- [ ] **Iniciar sesión con Apple** si se ofrece inicio con Google.
- [ ] **Borrado de cuenta dentro de la app** (Guideline 5.1.1(v)).
- [ ] Política de privacidad y "etiquetas de privacidad" (App Privacy).
- [ ] Donaciones al desarrollador como **"propinas" vía In-App Purchase consumible** (Guideline 3.1.1). No usar enlaces externos de pago en la app.
- [ ] Cuenta de demostración para el revisor de Apple (usuario admin + cajero con datos de ejemplo).
- [ ] Capturas para iPhone (6.7" / 6.9") y, si se habilita, iPad.

> ⚠️ Las políticas de las tiendas cambian con frecuencia. Revisar la documentación oficial de Google Play Console y App Store Review Guidelines antes de cada envío.

### Legal (Chile)
- Política de privacidad conforme a la **Ley 19.628** y a la nueva **Ley 21.719** de protección de datos personales (entra en vigencia en diciembre de 2026).
- Términos de uso: la app es de control interno y **no es un documento tributario**.

---

## 7. Costos estimados

| Concepto | Inicio | Mensual / anual |
|---|---|---|
| Google Play | USD 25 (único) | — |
| Apple Developer | — | USD 99 / año |
| Supabase | Gratis (500 MB BD, 50.000 usuarios activos/mes) | USD 25/mes si se supera el plan gratuito |
| Dominio para política de privacidad | Opcional (puede usarse GitHub Pages gratis) | ~USD 12 / año |
| Sentry, GitHub, GitHub Actions | Gratis | — |
| **Total mínimo solo Android** | **USD 25** | **USD 0** |
| **Total Android + iOS** | **USD 25** | **USD 99 / año** |

**Comisión de donaciones:** Google y Apple retienen **15 %** para desarrolladores pequeños (bajo USD 1 millón al año, inscribiéndose en sus programas), es decir, de una donación de USD 5 llegan aprox. USD 4,25.

> Nota sobre el plan gratuito de Supabase: los proyectos se pausan tras 7 días sin actividad. Con uso diario de una panadería esto no ocurre, pero en desarrollo hay que tenerlo presente.

---

## 8. Modelo de donaciones y sostenibilidad

- Todas las funciones son **gratis para siempre**; sin anuncios.
- Pantalla "Apoya el proyecto" accesible desde el menú; un recordatorio amable **máximo una vez al mes**, solo a administradores, y nunca durante una venta.
- Transparencia: mostrar en la pantalla en qué se usan las donaciones (servidor, cuota de Apple, nuevas funciones).
- Fuera de la app (sitio web, README de GitHub, redes sociales) se pueden ofrecer también GitHub Sponsors, Ko-fi o Buy Me a Coffee.
- Si el proyecto crece, opción futura: plan "Pro" para cadenas de panaderías (multi-sucursal avanzado), manteniendo gratis todo lo actual.

---

## 9. Cronograma (1 desarrollador, medio tiempo)

| Fase | Semanas | Entregable |
|---|---|---|
| 0. Preparación | 1 | Cuentas (Supabase, Google Play), diseño de pantallas en papel/Figma, repositorio y CI. |
| 1. Base | 2 | Autenticación, crear panadería, roles e invitaciones. |
| 2. Catálogo e inventario | 2 | Productos, ingresos, mermas, ajustes, stock y alertas. |
| 3. Caja y ventas | 2–3 | Pantalla de venta, rebajas, turnos, modo offline y sincronización. |
| 4. Reportes | 1–2 | Reportes por período y cajero, CSV y PDF. |
| 5. Donaciones y cumplimiento | 1 | IAP, política de privacidad, borrado de cuenta, textos legales. |
| 6. Prueba cerrada | 2 | 12+ testers durante 14 días en Google Play; corrección de errores. |
| 7. Publicación | 1 | Producción en Google Play; luego envío a App Store. |
| **Total** | **~12–14 semanas** | App 1.0 publicada |

A tiempo completo, el total baja a **~7–8 semanas** (la prueba cerrada de 14 días no se puede acortar).

**Recomendación:** lanzar primero en **Android** (más barato, más usado en Chile en comercios pequeños) y publicar en iOS cuando las donaciones o el uso lo justifiquen.

---

## 10. Indicadores de éxito

- 10 panaderías usando la app activamente a los 3 meses del lanzamiento.
- Calificación ≥ 4,5 en la tienda.
- Tasa de fallos (crash-free) ≥ 99 %.
- Donaciones que cubran al menos la cuota anual de Apple y el servidor.

---

## 11. Próximos pasos inmediatos

1. Validar este plan con 2–3 panaderías reales (¿cómo venden hoy? ¿por kilo o unidad? ¿cuántos cajeros?).
2. Crear la cuenta de Google Play (la verificación puede tardar días).
3. Crear el proyecto en Supabase y aplicar el esquema de [DESARROLLO.md](./DESARROLLO.md).
4. Comenzar la Fase 1.
