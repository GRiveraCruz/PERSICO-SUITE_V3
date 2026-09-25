# Cambios — Cantidades NORMAL/MIRROR y Órdenes de Producción (rev38 → rev39)

## 1. BOM de Manufactura: columnas NORMAL y MIRROR
- Dos columnas editables por pieza. Al subir un plano: Normal 1, Mirror 0.
- **Cantidad requerida = Normal + Mirror.** Es la que usa la orden de compra de
  manufactura como cantidad por omisión y la que cuenta para el pendiente.

## 2. Órdenes de Producción (Operaciones ▸ Órdenes de Producción)
El módulo, que estaba "en construcción", ya funciona.

**Creación (desde Requisición ▸ Manufactura, botón "Generar Orden de Producción"):**
- **Qué piezas se listan:** las de Fabricación **Interna** en estatus **Solicitado** que no
  están en otra orden. Si hay piezas Solicitadas con otra fabricación, se avisa.
- **Datos de la orden:** prioridad (número, 1 = más urgente; obligatoria), fecha de
  entrega requerida (obligatoria) y notas.
- **Folio consecutivo `MNO-000001`,** con el contador atómico de folios: no se repite ni
  se reutiliza.
- **Herencia por pieza:** ID, Tipo, Material, Acabado, revisión y **plano** (el PDF de la
  revisión vigente, clic para abrirlo), más las cantidades Normal y Mirror.

**Listado:**
- Columnas: folio, Job, piezas (con sus IDs), prioridad y fecha de entrega (⚠ en rojo si
  ya venció y la orden no está Concluida ni Cancelada).
- También: estatus con color, **barra de avance** (procesos concluidos / procesos que
  aplican) y quién la creó.
- Filtros por Job y por estatus.

**Modal de la orden (clic en el listado o en el folio desde el BOM):**
- Se editan prioridad, fecha de entrega, estatus y notas.
- **Estatus:** Pendiente · En proceso · En pausa · Concluida · Cancelada.
- **Matriz de procesos por pieza** (Corte, Soldadura, CNC, Torno, Fresa, Pintura):
  - Cada celda puede ser "No aplica", "Pendiente" (el proceso aplica) o "✔ Concluido".
  - Al concluir se registra quién y cuándo, y se muestra debajo.
- **Historial** de cambios (quién, cuándo, qué).
- **"📄 PDF del estatus":** página lista para imprimir o guardar como PDF, con los datos,
  el avance y la matriz.
- **"Eliminar":** solo para nivel completo, y solo en órdenes Pendientes o Canceladas.

**Reglas:**
- **No se puede concluir** una orden con procesos Pendientes.
- En órdenes **Concluidas o Canceladas**, la matriz queda de solo lectura.

## 3. Vínculo con el BOM de Manufactura
| Evento en la orden | Pieza en el BOM |
|---|---|
| Creada | **Orden interna**, con enlace al folio |
| Concluida | **Fabricado** |
| Cancelada / eliminada | **Solicitado**, sin orden (se puede volver a ordenar) |
| Reabierta (de Concluida o Cancelada a otro estatus) | **Orden interna** otra vez (si la pieza no quedó en otra orden) |

- Mientras la pieza está en una orden, su estatus y su fabricación **los controla la
  orden** (selectores bloqueados) y la pieza **no se puede eliminar**.
- En el Dashboard de Compras, las piezas en Orden interna o Fabricado cuentan como
  ordenadas.

## 4. Permisos (módulo "Operaciones — Órdenes de Producción")
| Nivel | Puede |
|---|---|
| Ver | consultar el listado, la orden y el PDF |
| Crear | crear órdenes desde el BOM, configurar la matriz, marcar procesos, cambiar estatus |
| Completo | además, eliminar órdenes |

El perfil **MANUFACTURING** (supervisor de manufactura) pasa de "Ver" a **"Crear"**.
Aplica a usuarios nuevos; los existentes se ajustan en Config ▸ Administrador.

## API
- `GET`/`POST /api/ordenes-produccion`
- `GET`/`PUT`/`DELETE /api/ordenes-produccion/<folio>`
- `GET /api/ordenes-produccion/<folio>/pdf`
- Tabla nueva `ordenes_produccion`; se crea sola al arrancar.

## Cómo se probó
- **PostgreSQL (23 verificaciones), entre otras:**
  - Normal 2 + Mirror 2 → cantidad 4; compra externa por 4 → Comprado.
  - Pieza Externa y prioridad vacía → rechazadas.
  - MNO-000001 hereda los datos del plano; en el BOM → Orden interna.
  - No se puede duplicar ni eliminar una pieza en orden, ni cambiar su estatus.
  - No se concluye con procesos pendientes; los concluidos registran usuario.
  - Concluida → Fabricado; reabrir → Orden interna; Cancelada → Solicitado.
  - El PDF muestra los procesos concluidos.
  - Perfil con solo "Ver": consulta sí, modificar no.
  - Eliminar la orden cancelada; siguiente folio MNO-000002.
- **Chromium:**
  - NORMAL/MIRROR editados desde la tabla.
  - Orden creada desde el BOM con 3 piezas.
  - Matriz configurada y un proceso concluido: avance 33 % en el listado.
  - Folio visible en el BOM; PDF generado.
  - A 1366 px y 1500 px se ven las 6 columnas de procesos sin desplazamiento.
  - Sin errores de JavaScript.
- **Regresión:** suites de manufactura/planos, orden de compra desde requisición y
  estatus de requisición; todas OK.
