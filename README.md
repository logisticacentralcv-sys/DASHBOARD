# Flota — Campo Verde

Sistema de gestión operativa de la flota de reparto. Vive como un proyecto
**separado** de TransferApp, pero comparte el mismo proyecto de Supabase
(mismo `SUPABASE_URL`), para poder cruzar datos entre ambos sin duplicar
padrones (repartidores, sucursales, recorridos).

## Arquitectura: módulos independientes

Cada módulo es un **archivo HTML aparte**, no una pestaña dentro de un
archivo gigante. Esto es a propósito: si un módulo se rompe (bug, cambio a
medias, lo que sea), **solo se cae ese módulo** — el resto del sistema
sigue funcionando, porque ni siquiera comparten el mismo código cargado en
el navegador.

| Archivo | Qué hace | Estado |
|---|---|---|
| `index.html` | Hub — navegación entre módulos | ✅ Activo |
| `flota-operativo.html` | Dashboard operativo (carga, tiempos, paradas, Verdulería, histórico) | ✅ Activo |
| `qr.html` | Página que escanea el repartidor (fábrica/sucursal) | ✅ Activo |
| `flota-mantenimiento.html` | Mantenimiento de camiones (service, VTV, seguros) | 🔜 Por hacer |
| `flota-insumos.html` | Pedidos de insumos de limpieza + stock | 🔜 Por hacer |

### Cómo agregar un módulo nuevo
1. Copiá el bloque de utilidades de cualquier módulo existente (conexión a
   Supabase, `esc()`, `fmt()`, `$()`, `renderSeguro()`) — es el mismo en
   todos, a propósito, para no depender de un build tool.
2. Usá SIEMPRE tablas con el prefijo del módulo (ej: `flota_mant_*`,
   `flota_insumos_*`) para que nunca choquen entre sí ni con TransferApp.
3. Agregá la tarjeta correspondiente en `index.html`.
4. Si el módulo necesita datos que ya existen en otra tabla (sucursales,
   usuarios/repartidores, recorridos), **leé de ahí directo** — no
   dupliques el padrón.

## Estructura interna de cada módulo (capas)

```
CAPA DE DATOS   (fetch*)     → trae filas de Supabase, no sabe de HTML
CAPA DE CÓMPUTO (compute*)   → funciones puras, arman lo que cada sección necesita
CAPA DE RENDER  (render*)    → dibuja HTML, envuelta en renderSeguro()
```

`renderSeguro(idElemento, funcionDeRender, nombreModulo)` es la pieza clave
de aislación de fallas: si `funcionDeRender` tira una excepción, en vez de
romper toda la página se muestra una tarjeta de error SOLO en esa sección,
y el resto del dashboard sigue andando. Toda sección nueva debe pasar por
acá.

## Base de datos (Supabase)

### Tablas propias de Flota (prefijo `flota_`)
- `flota_camiones` — vehículos y su capacidad
- `flota_viajes` — un viaje = un repartidor + recorrido, del día
- `flota_registros_qr` — cada escaneo (llegada/salida) en fábrica o sucursal
- `flota_carga_planificada` — foto diaria de la planificación (viene del puente con Apps Script)
- `flota_tiempos_estimados` — tiempo estimado de descarga por sucursal/recorrido
- `flota_historico` — cierre de días anteriores
- `flota_viz_verd` — seguimiento de las 3 fases de Verdulería (Mercado/Armado/Reparto)

### Tablas compartidas con TransferApp (no duplicar)
- `usuarios` — identidad de repartidores (por DNI, tipo=REPARTIDOR)
- `sucursales` — nombres oficiales + `lat`/`lon` (agregadas para el chequeo de GPS)
- `recorridos` — SUR, CENTRO, NORTE, REFUERZO, CONGELADOS, MARKETS, HUEVOS, SECOS
- `solicitudes` — usada solo para el aviso de "transferencias pendientes" en `qr.html`

Los archivos SQL de migración (`flota_01_...`, `flota_02_...`, etc.) quedan
numerados en orden — nunca se edita uno viejo, siempre se agrega uno nuevo.

## El puente con Apps Script

El motor de cálculo de la planificación (ocupación de camión, bultos,
tiempo estimado de descarga) sigue viviendo en Google Sheets — es lógica
ya afinada y no vale la pena reconstruirla. Lo que se agregó es una
función (`sincronizarASupabase`, ver `flota_02_puente_appscript.gs`) que
toma el resultado YA CALCULADO por las hojas y lo copia a
`flota_carga_planificada` / `flota_tiempos_estimados` / `flota_camiones`.
Se corre a mano desde el editor de Apps Script, o con un trigger de tiempo.

## Identidad del repartidor

Por diseño, **no hay PIN** — se usa el mismo DNI que ya identifica a la
persona en TransferApp (`usuarios`, tipo=REPARTIDOR). Si un repartidor
pierde el registro en su celular (se borró el `localStorage`), puede
recuperar su viaje activo con solo volver a escribir su DNI — no hace
falta ningún código adicional.
