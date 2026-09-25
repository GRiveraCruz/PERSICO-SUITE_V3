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
