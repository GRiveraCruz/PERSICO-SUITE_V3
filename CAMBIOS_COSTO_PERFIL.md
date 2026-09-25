# Cambios — Costo promedio por hora por perfil (rev42 → rev43)

**Menú:** Proyectos ▸ **Costo por Perfil (USD/h)**.

**Qué muestra:** tabla con el costo promedio por hora en dólares de cada perfil, más el
mínimo, el máximo y cuántas personas entran en el cálculo. Tiene selector de año.

**Origen de los datos:** se calcula de las tarifas por hora (USD) de **Recursos Humanos ▸
Hourly Rate** del año, agrupadas por departamento:

| Perfil | Departamento |
|---|---|
| Pintor | MANUFACTURING - PAINT |
| Soldador | MANUFACTURING - WELD |
| Mecánico de ensamble | ASSEMBLY |
| Diseñador mecánico | MECHANIC ENG |
| Diseñador eléctrico | ELECTRIC ENG - DESIGN |
| Programador de PLC | ELECTRIC ENG - PLC |
| Operador de CNC | MANUFACTURING - CNC |

- **Se actualiza sola:** al cambiar tarifas en Hourly Rate, la tabla cambia; no se captura
  nada aparte.
- **Perfil sin trabajadores** en el año → "sin datos".
- **Perfil con una sola persona** → aviso ⚠, porque en ese caso el promedio es la tarifa
  de esa persona.
- **Permiso nuevo "Costo por Perfil (USD/h)":** cada perfil hereda el nivel que tiene en
  Hourly Rate. El endpoint solo devuelve promedios, mínimos y máximos, no la tarifa de
  cada persona.
- **API:** `GET /api/costos-perfil?year=`.

**Probado con las tarifas 2026 de data_seed:**
- Pintor $7.43, Soldador $8.64, Mecánico de ensamble $8.88 (12 personas), Diseñador
  mecánico $8.63, Diseñador eléctrico $8.37, Programador de PLC $8.36, Operador de CNC
  $8.63. Promedio general $8.42.
- Chromium: 7 filas, menú visible, sin errores de JavaScript.

---
# rev44 — Configurar Proyecto con el board BOARD_CONFIG_PROYECTOS

La sección de estimados de cada Job (Proyectos ▸ Configurar Proyecto) queda como en el
board.

## Cálculos (sin cambios, coinciden con el board)
- **Monto Markup** = Revenue − Revenue ÷ (1 + Markup %).
- **Calculation Cost** = Revenue − Monto Markup.
- **Internal Target** = Calculation Cost × (1 − Target de Ahorro %).
- **Delta vs Internal Target** = Internal Target − Suma de estimados.

Con el ejemplo del board (Revenue 40,800, Markup 50 %, Ahorro 30 %) se obtiene: Markup
13,600, Calculation Cost 27,200 e Internal Target 19,040.

## Estimados
- **Montos:** Material mecánico, Material eléctrico, Major items y Servicios externos.
- **Mano de obra: HORAS × COSTO PROMEDIO (USD/h) = IMPORTE**, en 7 líneas ligadas a los
  perfiles de "Costo por Perfil":

  | Línea | Perfil |
  |---|---|
  | Diseño mecánico | Diseñador mecánico |
  | Soldadura | Soldador |
  | Manufactura | Operador de CNC |
  | Pintura | Pintor |
  | Diseño eléctrico | Diseñador eléctrico |
  | Programación de PLC | Programador de PLC |
  | Ensamble (electromecánico) | Mecánico de ensamble |

- **Costo promedio:**
  - Se llena con el promedio vigente del perfil y **se guarda en el Job**. Si después
    cambian las tarifas, los Jobs ya configurados no se mueven solos.
  - El botón **"↻ Actualizar costos promedio"** toma los valores vigentes.
  - El valor es editable por Job si se necesita ajustar.
- **Totales:** Materiales, Mano de obra, Suma y Delta vs Internal Target.
- **Resumen por áreas del PT:** muestra los 4 materiales y las 7 líneas de mano de obra.

## Compatibilidad
- **Target Compras** = suma de los 4 materiales (igual que antes).
- **Target M.O.** = suma de las 7 líneas (horas × costo).
- Se siguen guardando `est_ing_mecanica`, `est_ing_electrica` y `est_ensamble`, derivados
  de las líneas (diseño mecánico; diseño eléctrico + PLC; ensamble + soldadura +
  manufactura + pintura). Job Report, Dashboards y demás consumidores no cambian.
- **Jobs configurados antes sin horas:** sus montos de mano de obra aparecen en el renglón
  **"Mano de obra (monto anterior, sin horas)"** y cuentan en la suma. Al capturar horas,
  ese renglón se pone en 0.
- **Permisos:** Configurar Proyecto puede consultar los promedios (solo promedios) aunque
  el usuario no tenga acceso al módulo "Costo por Perfil".

## Cómo se probó (Chromium, PT-0099 / Job 652-50, Revenue 168,000)
- **Captura:** Markup 50 % y Ahorro 30 % → Monto 56,000; Calculation Cost 112,000;
  Internal Target 78,400.
- **Materiales y mano de obra:** material mecánico 10,000; Diseño mecánico 100 h × 8.63 =
  863.00; Soldadura 20 h × 8.64 = 172.80. Mano de obra 1,035.80, Suma 11,035.80, Delta
  67,364.20.
- **Guardado:** horas y costo por línea, `target_compras` 10,000, `target_mo` 1,035.80 y
  campos derivados. Al recargar se conservan.
- Sin errores de JavaScript.

---
# rev45 — Nuevas líneas de mano de obra y columna "Horas consumidas"

## Líneas nuevas en Configurar Proyecto (9 en total, en el orden del board)
Diseño mecánico · Soldadura · Manufactura · Pintura · Diseño eléctrico · Programación de
PLC · **Programación de robots** · **Simulación** · Ensamble (electromecánico).

**Perfiles nuevos en "Costo por Perfil":**

| Perfil | Departamentos aceptados en Hourly Rate |
|---|---|
| Programador de robots | ELECTRIC ENG - ROBOTICS / ELECTRIC ENG - ROBOTS / ROBOTICS / ROBOTS |
| Ingeniero de simulación | MECHANIC ENG - SIMULATION / SIMULATION / SIMULACION |

Hoy ningún trabajador tiene esos departamentos, así que aparecen "sin datos" y su costo
promedio se captura a mano en el Job. En cuanto se registren trabajadores con alguno de
esos departamentos, el promedio se calcula solo. Cada perfil ahora acepta varios nombres
de departamento.

**Campos derivados para reportes:** Robots suma a `est_ing_electrica` y Simulación a
`est_ing_mecanica`. Target M.O. incluye las 9 líneas.

## Columna "Horas consumidas" (verde)
- **Qué muestra:** las horas registradas en **Work Hours** para ese Job hasta el momento
  de abrir la configuración. Toma todos los años con registros y el mismo criterio de
  coincidencia de Job que el Job Report.
- **Clasificación por línea:** cada hora se asigna según el **departamento del trabajador
  en Hourly Rate**: departamento → perfil → línea.
- **Avance contra lo estimado:** junto a las horas aparece el %. En verde por debajo de
  85 %, en ámbar de 85 a 100 %, en rojo arriba de 100 %. Si hay horas consumidas en una
  línea sin horas estimadas, dice "sin est.".
- **"Otras horas":** renglón aparte con las horas de trabajadores sin tarifa o cuyo
  departamento no corresponde a ningún perfil. El tooltip muestra el desglose.
- **API:** `GET /api/projconfig/horas-consumidas?jobs=652-50,665-00`.

## Cómo se probó (Job 652-50 de data_seed)
- **Total:** 5,278.6 horas, igual al total del Job Report.
- **Por línea:** Diseño mecánico 1,089 · Soldadura 137 · Manufactura 153 · Pintura 250 ·
  Diseño eléctrico 616 · PLC 575 · Ensamble 715.
- **Otras horas:** 1,743.6, de trabajadores sin tarifa.
- **Porcentajes con horas estimadas:** Diseño mecánico 1,000 → 109 % en rojo; PLC 800 →
  72 %; Ensamble 900 → 79 %.
- Chromium sin errores de JavaScript.

---
# rev46 — Columna "Costo real" en la mano de obra de Configurar Proyecto

- **Columna nueva "Costo real"** (verde, junto a Horas consumidas): costo de las horas
  registradas de cada línea. Se calcula igual que el Job Report:
  - Cada registro de Work Hours × `cost_per_hour` del registro, si lo trae.
  - Si no, × la tarifa del trabajador en Hourly Rate para el año del registro.
  - Si no hay tarifa de ese año, × su tarifa más reciente.
  - Redondeo a centavos por registro.
- **Avance del gasto:** junto al costo aparece el % contra el **Importe estimado** de la
  línea (horas × costo promedio). Verde por debajo de 85 %, ámbar de 85 a 100 %, rojo
  arriba de 100 %.
- **Renglón "Otras horas":** muestra también su costo. Si hay horas de trabajadores sin
  tarifa, aparece ⚠: esas horas no se pueden costear.
- **Renglón "Total consumido":** horas y costo real totales del Job.
- **API:** `/api/projconfig/horas-consumidas` agrega `costo` por línea, `otras_costo`,
  `horas_sin_tarifa` y `costo_total`.

**Probado (Job 652-50):**
- **Total:** costo real $32,272.82, **idéntico** al "amount_wh" del Job Report. Antes de
  igualar el redondeo por registro había una diferencia de $0.29.
- **Por línea:** Diseño mecánico $10,620.40 de $8,630.00 estimados (123 %, rojo); PLC
  $5,058.78 (76 %); Ensamble $7,284.40 (91 %, ámbar).
- **Horas sin tarifa:** 1,743.6 h, marcadas con ⚠.
- Chromium sin errores de JavaScript.
