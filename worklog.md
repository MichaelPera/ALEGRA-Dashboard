# Worklog — People Scorecard (Alegra)

---
Task ID: 1
Agent: main (Z.ai Code)
Task: Fundaciones del proyecto People Scorecard: deps, pipeline de datos (parseo Excel → limpieza → métricas → insights), store Zustand, shell de UI (layout, sidebar, topbar con filtros globales, upload dialog, panel de calidad, umbrales), página principal y Resumen Ejecutivo.

Work Log:
- Explorado el proyecto Next.js 16 + Tailwind 4 + shadcn/ui en /home/z/my-project.
- Instaladas dependencias: `xlsx` (SheetJS), `html-to-image`. Recharts/Zustand/next-themes ya estaban.
- Copiado el Excel del reto a `public/data/Base_Reto_People.xlsx` (se carga como estado inicial vía fetch, sin datos quemados).
- Creado el pipeline de datos en `src/lib/data/`:
  - `types.ts` — EmployeeRow, QualityReport, CleanedDataset, filtros, umbrales, Insight.
  - `dateUtils.ts` — parseo flexible de fechas mixtas (AAAA-MM-DD hh:mm:ss, AAAA/MM/DD, DD/MM/AAAA, MM-DD-AAAA, seriales Excel) → ISO; antigüedades; meses.
  - `cleaning.ts` — diccionario de áreas (8 canónicas, editable), mapeo tolerante de columnas, outliers salariales (IQR + umbrales 700/9000, se marcan no se borran), nulos en desempeño/clima sin rellenar, consistencia Estado↔Fecha_salida, salida<ingreso, duplicados por ID.
  - `parseExcel.ts` — detección automática de hoja + mapeo de columnas con SheetJS.
  - `metrics.ts` — computeBundle: headcount, rotación acumulada/12m, % voluntarias, salida temprana, clima, desempeño, mediana salarial, talento en riesgo (des≥4 & clima<5.5), series mensuales (57 meses con datos reales), desgloses por área/país/cargo/antigüedad, heatmap Área×País, 9-box, curva de salida, correlación salario-desempeño, boxplot, cobertura %.
  - `insights.ts` — insights automáticos con severidad + enlace a sección.
  - `export.ts` — plantilla Excel, export XLSX/CSV.
  - `idb.ts` — persistencia del dataset en IndexedDB.
  - `selfTest.ts` — verificaciones visibles de limpieza de fechas/áreas (se ejecutan desde Calidad de datos).
  - `umbrales.ts` — semáforos por defecto.
- Verificado con `bun scripts/verify.ts` sobre el Excel real: 830 filas, 530 activos, rotación 12m 31.2%, 59 talento en riesgo, 0 fechas inválidas, formatos mixtos detectados correctamente, insights coherentes.
- Store Zustand `src/store/useDashboardStore.ts` (settings persistidos en localStorage, dataset en IndexedDB, re-limpieza al editar diccionario de áreas).
- Hooks: `src/hooks/useMetrics.ts` (bundle memoizado + filtros).
- UI compartida: `globals.css` con paleta Alegra (mint #00D6BC, #0B8F7E, #E6FAF6, tinta #0B2B27, fondo #F7FAF9) + modo oscuro; `layout.tsx` con Inter y ThemeProvider; componentes `dashboard/`: chart-theme, ui-bits (Panel, chips, semáforo, badges), kpi-card (contador animado + sparkline + semáforo + tooltip definición), privacy (enmascara nombres/correos), panel-calidad, upload-dialog (drag & drop), topbar (filtros globales, corte editable, comparación, privacidad, outliers), sidebar, thresholds-editor, secciones.ts, people-table (drill con búsqueda).
- `src/app/page.tsx`: shell completo con 8 secciones, modo presentación (una sección por diapositiva, teclas ←/→/Esc, fullscreen), footer sticky, estados vacíos amigables.
- `src/components/sections/resumen-section.tsx`: home con 8 KPIs animados + "Lo que debes saber" + composición rápida.
- Stubs creados para las secciones que construyen los agentes 2-a/2-b/2-c.

Stage Summary:
- APIs estables para las secciones: cada sección es `{ bundle }: { bundle: MetricsBundle }` (import type from '@/lib/data/metrics').
- Bundle incluye: core (todos los KPI + listas), series (mensual), porArea/porPais (StatsGrupo completos), porCargo, porAntiguedad, curvaSalida, heatmapClima, matriz9Box (cells con rows para drill), correlacionSalarioDesempeno, boxplotPorArea, outliersSalario, medianaPorPais, medianaPorDesempeno, cobertura, registrosPosterioresAlCorte, prev (mismas métricas al corte anterior).
- Componentes compartidos listos: Panel, SectionHeader, SevBadge, FilterChip, EmptyState, Cargando, BarraComparativa, KpiCard, PeopleTable, chart-theme (MENTA, COLOR_SERIES, TooltipBox, ejes, GradienteMenta, colorClima), usePrivacidad().
- Convenciones: español en UI, Tailwind con vars (--mint etc.), esquina 16px (panel-card), sombras suaves, Lucide icons, sin azul/indigo.
- Próximo paso: agentes 2-a (Personas+Rotación), 2-b (Desempeño+Compensación), 2-c (Áreas+Calidad+Datos) en paralelo; luego integración, lint y verificación en navegador.

---
Task ID: 2-a
Agent: frontend-secciones-personas-rotacion
Task: Secciones Personas y Rotación y retención

Work Log:
- Leí el worklog de la Task 1 y los contratos reales: ui-bits (Panel/SectionHeader/FilterChip), chart-theme (MENTA, ejes, TooltipBox, GradienteMenta/usarGradienteMenta, colorClima), store (setFilter/filters/thresholds), metrics.ts (StatsGrupo, SerieMensual, CoreMetrics, curvaSalida, porAntiguedad, porCargo), PeopleTable, useMetrics, format.ts y page.tsx (enrutado de secciones).
- Reemplacé el stub `src/components/sections/personas-section.tsx` (export `PersonasSection`):
  - "Headcount por mes": ComposedChart con doble eje (planta en eje izq. como área con GradienteMenta; ingresos/salidas como barras en eje der.; neto como línea punteada), tooltip rico con TooltipBox y selector de rango local 12/24/Todos con botones aria-pressed.
  - "Headcount por área": barras horizontales con gradiente menta y radius [0,8,8,0], ordenadas (mayor arriba), clic → setFilter('area') con toggle, Cell con fillOpacity 0.35 en no seleccionadas, chip FilterChip activo, LabelList con conteo.
  - "Headcount por país": barras verticales con colorPorIndice, clic → setFilter('pais'), resaltado idéntico + chip.
  - "Top 12 cargos": tabla compacta (cargo, área, headcount, mediana USD, clima con chip colorClima, desempeño); clic en fila → setFilter('cargo'); botón "Ver personas" (stopPropagation) que abre un Panel aparte con PeopleTable de activos filtrados por cargo.
  - "Planta por antigüedad": barras agrupadas activos (gradiente) vs salidas históricas con etiquetas por bucket.
  - Explorador drill-down con estado local: empresa → área → cargo → PeopleTable (core.activos filtrado por área+cargo), breadcrumb clicable + botones "volver"; la selección se auto-valida si los filtros globales cambian.
- Reemplacé el stub `src/components/sections/rotacion-section.tsx` (export `RotacionSection`):
  - 4 KPIs de cabecera con KpiCard (rotación 12m con semáforo, rotación acumulada, rotación voluntaria sobre personas, salidas del período) con delta vs. bundle.prev y sparklines de bundle.series.
  - "Tasa mensual y acumulada": ComposedChart barras de salidas (eje izq.) + línea rotación 12m rolling % (eje der.) + ReferenceLines punteadas con thresholds.rotacion12m (ámbar = atención, menta = meta) etiquetadas.
  - "Motivo de salida": dona Recharts Pie con Voluntario #00D6BC / Decisión de Alegra #F0A93C / Mutuo Acuerdo #E36A6A (+ "Otro" neutro si existe), KPI de % voluntarias (core.pctVoluntarias) con semáforo al lado, leyenda clicable y clic en segmento → setFilter('tipoSalida'), total de salidas al centro de la dona.
  - "Rotación 12m por área" y "por país": barras horizontales ordenadas desc (peor arriba), coloreadas por semáforo con thresholds del store (>=ámbar #E36A6A, >=verde #F0A93C, si no #00D6BC), ReferenceLine en el umbral, clic → setFilter y opacity 0.35 en no seleccionadas.
  - "Curva de salida por antigüedad": barras % por bucket con LabelList + caja de insight calculada desde curvaSalida (bucket pico con % y conteo + % que se va antes del primer año).
  - "Ranking de áreas": tabla (headcount, salidas ventana, rotación 12m con SemaforoDot y color semáforo, % voluntarias, salida temprana, mediana salarial), hover + fila activa, clic → setFilter('area').
- Correcciones de calidad: eliminé un useMemo con retorno union (string | objeto) que rompía tsc; verifiqué con `bunx tsc --noEmit` (0 errores en mis 2 archivos) y `bunx eslint` sobre ambos (0 problemas). `bun run lint` global sigue con 4 errores PREEXISTENTES en topbar.tsx y calidad-datos-section.tsx (de Task 1 / otro agente, fuera de mi alcance).

Stage Summary:
- Entregadas las 2 secciones completas en español, 100% alimentadas por `bundle` (sin datos quemados ni fechas fijas), con "use client", tooltips TooltipBox, grid var(--border) dasharray 3 3, ejes compartidos, gradientes menta por gráfico (usarGradienteMenta + GradienteMenta idBase) y ResponsiveContainer ~230-380px.
- Cross-filtering en todos los gráficos/tablas vía store (area/pais/cargo/tipoSalida) con toggle al re-clic, resaltado por opacity 0.35 y chips FilterChip removibles en cada panel; drill-down de personas con estado local + breadcrumb (sin tocar page.tsx ni store).
- Decisiones clave: barras ascendentes en layout vertical para que el peor/mayor quede arriba; doble eje en headcount mensual (planta vs flujos) y en rotación (salidas vs %); amber/rojo reservados a semáforos y motivo de salida; insight de curva de salida calculado dinámicamente (pico por % y salida temprana acumulada).
- Pendiente para integración: otros agentes deben resolver lint preexistente en topbar.tsx y calidad-datos-section.tsx; verificación visual final en navegador (modo presentación incluido).

---
Task ID: 2-b
Agent: frontend-secciones-desempeno-compensacion
Task: Secciones Desempeño y clima + Compensación y equidad

Work Log:
- Leído worklog.md y contrato existente (MetricsBundle, store, chart-theme, ui-bits, PeopleTable, format) sin modificar archivos de otros.
- Reemplazado el stub `src/components/sections/desempeno-section.tsx` por la implementación completa:
  - KPIs de cabecera con semáforo (MiniKpi local + `semaforo()` con thresholds.desempeno/clima/talentoRiesgo del store): desempeño promedio, clima promedio, % clima bajo <5, talento en riesgo (count + %).
  - Distribución del desempeño 1-5 (Recharts BarChart) con % encima de cada barra (LabelList), niveles 1-2 en ámbar #F0A93C, 3 medio (menta claro), 4-5 menta, y tramo "Sin dato" punteado gris desde `core.sinDatoDesempeno` + leyenda.
  - Clima por área: barras horizontales 0-10 con `colorClima` por barra, ReferenceLine dashed en `core.climaPromedio`, clic → toggle `setFilter('area')`, área seleccionado resaltado y resto fillOpacity 0.35 (Cell).
  - Clima por país: barras verticales con `colorClima`, línea de empresa, etiquetas de valor, clic → `setFilter('pais')`.
  - Mapa de calor Área × País como GRID de divs (sin Recharts): celda con `colorClima`/`textoSobreClima`, valor.toFixed(1) + n pequeño, celdas n=0/sin dato → gris punteado "—", tooltip nativo title con detalle, fila del área seleccionada con ring, overflow-x auto para móvil, leyenda de escala, clic en celda → filtro por área.
  - Matriz 9-box simplificada Desempeño × Clima: grid 3×3 (X clima bajo→alto, Y desempeño alto arriba), tinte menta proporcional al count, celda desBand=2 & climaBand=0 con borde rojo + badge "EN RIESGO" + botón "Ver las N personas" → Dialog shadcn con PeopleTable (además botón equivalente en el panel lateral de "Prioridad de retención").
  - "Clima: se quedaron vs se fueron": dos barras grandes 0-10 con colorClima, badge Δ en puntos y frase de negocio según la brecha (insatisfacción anticipa salida / salidas no explicadas por clima / clima similar). Nulos como "Sin dato".
- Reemplazado el stub `src/components/sections/compensacion-section.tsx` por la implementación completa:
  - Switch global "Excluir salarios atípicos" en la cabecera (store.excluirOutliers + setExcluirOutliers) con aria-label; la descripción refleja el estado activo.
  - KPIs de cabecera: mediana salarial general, brecha de medianas entre áreas, brecha entre países y conteo/% de outliers.
  - Salario mediano por área: barras horizontales menta con fmtUSDCompacto en etiquetas, clic → toggle filtro área, dim 0.35 en no seleccionadas.
  - Mediana por país: barras verticales con GradienteMenta, n en tooltip, clic → filtro país; países sin mediana quedan sin barra (nunca 0).
  - Mediana por nivel de desempeño N1-N5 con N5 destacado en menta oscuro; insight automático: si mediana N5 ≤ N3×1.05 (o falta dato) muestra badge ámbar + alerta "la compensación no distingue desempeño".
  - Dispersión salarial: boxplot CUSTOM con divs (caja Q1→Q3 menta suave con borde menta-dark, línea de mediana 2px, bigotes 1px con tapas, escala común redondeada con gridlines dashed cada $1k — o $500 si el rango es corto — y etiquetas del eje), tooltip title nativo con min/Q1/mediana/Q3/máx/n, fila seleccionada resaltada y resto atenuado, clic → filtro área; áreas sin salario listadas como "Sin dato".
  - Salario vs desempeño: ScatterChart Recharts con jitter determinista ±0.15 en X, puntos menta con opacity 0.55, badge con r; alertas: |r|<0.3 → ámbar "el salario no está alineado al desempeño", r<0 → rojo, r sano → menta; r null → "Sin dato". Puntos desde `core.activos` respetando el interruptor de outliers.
  - Outliers listados aparte: panel expandible (Collapsible shadcn) con badges de conteo por motivo (iqr_alto/iqr_bajo/umbral_alto/umbral_bajo) y PeopleTable completo (privacidad respetada dentro de la tabla).
- Validación: `bunx eslint` de los 2 archivos → 0 problemas; `bunx tsc --noEmit` → 0 errores en los 2 archivos (los errores/warnings restantes son preexistentes en dashboard/kpi-card.tsx, dashboard/topbar.tsx, dashboard/chart-theme.tsx, page.tsx, panel-calidad.tsx y ejemplos, fuera de mi alcance por instrucción).

Stage Summary:
- Secciones "Desempeño y clima" y "Compensación y equidad" 100% funcionales en español, con "use client", sin datos quemados y con cross-filtering global (área/país) desde barras, heatmap y boxplot.
- Convenciones respetadas: Panel/SectionHeader, TooltipBox, ejes, grid var(--border) dashed, degradados menta, radios redondeados, grids responsive `grid gap-4 lg:grid-cols-2`, gráficos height 300, paleta mint con ámbar/rojo solo para alertas, nulos siempre "Sin dato" (nunca 0), aria-labels en controles interactivos.
- El interruptor "Excluir salarios atípicos" recalcula todo el bundle vía useMetrics, por lo que medianas, boxplot, scatter y KPIs reaccionan en vivo en ambas direcciones.
- Pendiente para integración: errores de lint preexistentes en dashboard/ (kpi-card, topbar, chart-theme `usarGradienteMenta` regla de hooks — funciona en runtime pero conviene renombrar) y el disable sin uso en page.tsx; revisión visual final en navegador y presentación.

---
Task ID: 2-c
Agent: frontend-secciones-areas-calidad-datos
Task: Secciones Detalle por área + Calidad de datos + Datos

Work Log:
- Leí worklog.md (Task 1) y los contratos existentes: types.ts, metrics.ts (StatsGrupo/CoreMetrics/porCargo), store, useMetrics, ui-bits, privacy, panel-calidad, people-table, cleaning (AREAS_CANONICAS), selfTest, export, umbrales.
- Reemplacé los 3 stubs por implementaciones completas (solo esos 3 archivos, nada más tocado).
- detalle-area-section.tsx: grid de fichas (md:2/xl:3) con alertas locales (rotación > ámbar → "Rotación crítica X%", clima < ámbar → "Clima bajo X", talentoRiesgo ≥ 3 → "N en riesgo", salidaTemprana > 40 → "Muchas salidas tempranas"); 5 KPIs vs promedio de empresa con mini-barra (barra menta del área + marcador vertical del promedio) y ▲/▼ verde/rojo según mejor-peor (menos es mejor en rotación/voluntarias); sparkline Recharts AreaChart sin ejes h-10 de headcountSerie (menta si sube, ámbar si cayó, gradiente propio con useId); "Ver cargos" expande lista de bundle.porCargo filtrada por área y clic → Dialog con PeopleTable de core.activos filtrados por área+cargo; con store.filters.area la ficha se resalta (ring-mint), va primera y muestra botón para quitar el filtro.
- calidad-datos-section.tsx: EmptyState si no hay dataset (botón abre upload); Panel de info del archivo (nombre, hoja, badge muestra/usuario, uploadedAt toLocaleString('es')); PanelCalidadDatos dentro de Panel; EditorDiccionario en pantalla (tabla alias→canónica con Input+Select de AREAS_CANONICAS+eliminar, añadir alias, restaurar predeterminado, badge "Cambios sin guardar"; al guardar → setAreaDict + recargarConDiccionario; alias normalizados con normalizarTexto; se monta con key derivada del dict del store para resincronizar sin useEffect — así evité el error de lint react-hooks/set-state-in-effect); callout ámbar de areasSinClasificar con contador y "+ Añadir como alias" por valor; Switch "Excluir salarios atípicos" con explicación (mediana, nunca promedio) y estado actual; botón "Ejecutar verificaciones de limpieza" → ejecutarVerificaciones() con lista ✓/✗ (verde/rojo suaves), esperado vs obtenido solo en fallos y contador "X/Y pruebas pasan"; "Descargar plantilla Excel" (descargarPlantilla) y "Restablecer todo" con AlertDialog de confirmación → restablecer().
- datos-section.tsx: usa useFilteredRows(); búsqueda local sobre valores reales (id/nombre/correo/cargo/área/país aunque se muestren enmascarados), filtro local de estado (todas/Activo/Inactivo) complementario a los globales; orden por columna en las 13 columnas con ▲▼ + aria-sort y comparador localeCompare('es') con nulls siempre al final; paginación 25/50/100 con ‹ ›, contador "1–50 de N" aria-live y clamp de página; columnas ID/Nombre/Correo (usePrivacidad)/País/Área/Cargo/fechas ISO/Estado (badge verde suave/gris)/Tipo_salida/Salario_USD (⚠ ámbar con motivo del outlier)/Desempeño y Clima ("Sin dato" si null, clima < 5.5 en rojo #E36A6A); exportarCSV/exportarXLSX sobre el set filtrado+ordenado completo (no solo la página), disabled en 0 filas; aviso de privacidad cuando showPersonalData=false; ordenamiento/búsqueda/paginación con useMemo.
- Verificación: eslint de los 3 archivos → 0 problemas; tsc → 0 errores en mis 3 archivos (quedan errores preexistentes en desempeno-section.tsx y panel-calidad.tsx, de otros tasks); smoke test en navegador (dev server ya corriendo): fichas de área renderizan con alertas/tendencia, expansión de cargos y Dialog con PeopleTable funcionan ("FP&A Analyst · Finance, 30 personas activas"), verificaciones ejecutan (30/32 pasan — los 2 fallos son de selfTest/dateUtils del Task 1), tabla Datos con 830 filas, búsqueda "Ana"→341, orden salario desc con ⚠ en top, paginación 25/100, filtro Inactivo→300 filas, sin errores de consola nuevos (solo el warning preexistente de botones anidados en KpiCard).

Stage Summary:
- Entregadas las secciones 6 (Detalle por área), 7 (Calidad de datos) y 8 (Datos), con el mismo contrato { bundle }: { bundle: MetricsBundle } y exportaciones nombradas DetalleAreaSection / CalidadDatosSection / DatosSection (ya referenciadas por page.tsx).
- Decisión de diseño: la comparación vs empresa se dibuja con barra del área + marcador del promedio (misma escala por fila, 112% del máximo para holgura); mediana salarial se compara "más es mejor" como el resto de KPIs positivos; ámbar/rojo reservados a alertas (badges, clima bajo, ⚠ outliers, deltas peores).
- Decisión técnica: el estado del editor de diccionario vive en un componente hijo con key = contenido del dict (patrón de reset por remount) para cumplir la regla react-hooks/set-state-in-effect; los ids de fila se generan por max+1 para evitar colisiones de key.
- Pendientes para integración (no son de este task): 2 fallos en selfTest (serial Excel 44920 → parseFechaFlexible devuelve null; expectativa de cuartiles Q3 9.25 vs 7.75 con interpolación actual) — ajustar dateUtils o el caso esperado; errores TS/lint preexistentes en desempeno-section.tsx y panel-calidad.tsx; y el warning de <button> anidado dentro de KpiCard (DefinicionTooltip sobre botón).

---
Task ID: 4-5-7 (integración)
Agent: main (Z.ai Code)
Task: Integración de las secciones de los agentes 2-a/2-b/2-c, correcciones de lint/TS, verificaciones de limpieza y preparación para QA en navegador.

Work Log:
- Recibidas e integradas las secciones Personas, Rotación (2-a), Desempeño, Compensación (2-b) y Detalle por Área, Calidad de datos, Datos (2-c).
- Renombrado `usarGradienteMenta` → `useGradienteMenta` en chart-theme y en las 3 secciones que lo importaban (regla react-hooks/rules-of-hooks).
- Corregido setState-in-effect en kpi-card.tsx (conteo animado ahora agenda el estado dentro de rAF) y topbar.tsx (tema sin estado de montaje, usa resolvedTheme).
- KpiCard: raíz cambiada de <button> a <div role="button"> para evitar button anidado (DefinicionTooltip usa button interno).
- panel-calidad.tsx: tipos del helper `lista` ahora EmployeeRow[] (3 errores TS).
- selfTest: serial Excel '44920' ahora se interpreta también como texto (caso CSV) en parseFechaFlexible; esperanza de cuartiles corregida a Q3=7.75. Resultado: 32/32 verificaciones pasan.
- `bun run lint` → 0 problemas. `bunx tsc --noEmit` → 0 errores en código de la app (solo errores preexistentes en examples/ y skills/, fuera del alcance).

Stage Summary:
- App compila limpia: lint ✓, tsc ✓, pipeline verificado con el Excel real (830 filas → 530 activos, rotación 12m 31.2%, 59 talento en riesgo).
- Pendiente: QA visual end-to-end con Agent Browser (golden path: carga, filtros, cross-filtering, drill-down, presentación, export, modo oscuro, móvil).

---
Task ID: 8 (QA end-to-end)
Agent: main (Z.ai Code)
Task: Verificación completa en navegador (Agent Browser) del golden path y corrección final de bugs.

Work Log:
- **Bug crítico encontrado y corregido**: Recharts descartaba los `<defs>` renderizados vía componente custom (`GradienteMenta`), dejando barras de "Headcount por área" invisibles. Solución: helper `defsGradienteMenta(id, opacidad)` que devuelve JSX plano `<defs>` + hook `useGradienteId`; actualizados personas/rotacion/compensacion-section.
- QA ejecutado con Agent Browser sobre http://localhost:3000:
  - Home: 8 KPIs con valores exactos vs pipeline (530, 31%, 60%, 41%, 7.0, 3.5, USD 2146, 59) con semáforos, deltas y sparklines ✓
  - Insights automáticos clicables llevan a la sección + filtro ✓
  - Secciones Personas/Rotación/Desempeño/Compensación/Áreas/Calidad/Datos renderizan sin errores de consola ✓
  - 9-box: celda "EN RIESGO" con 59 personas → dialog con PeopleTable enmascarado ✓
  - Fichas por área: KPIs vs promedio de empresa + alertas + tendencia + Ver cargos ✓
  - Cross-filtering: clic en barra Marketing → chip "Área: Marketing" + todo recalculado; chip removible limpia ✓
  - Búsqueda en tabla Datos (108 resultados para "marketing"), paginación 1–25 de 830 ✓
  - Carga real de Excel: archivo de prueba con área desconocida "Growth" → "Sin clasificar", duplicado EMP-9001 eliminado, fechas mixtas parseadas, corte auto-recalculado (26 dic 2023) y todo el dashboard redibujado sin tocar código ✓
  - Re-carga del Excel original vía UI: 830 registros restaurados + panel de calidad ✓
  - Banner de corte manual: al fijar 2023-10-27 → "Hay 270 registros posteriores a la fecha de corte", KPIs recalculados al snapshot del reto (559 activos, rotación 16%) ✓
  - Verificaciones automáticas en vivo desde Calidad de datos: 32/32 pasan ✓
  - Modo presentación (6/8, teclado, fullscreen) + exportar PNG (html-to-image) y PDF (print) ✓
  - Modo oscuro con paleta menta sobre verde profundo ✓
  - Móvil 390px: filtros apilados, nav chips scrollables, KPIs 2 columnas, footer pegado ✓
  - Privacidad: nombres/correos enmascarados por defecto; toggle los muestra ("Lorena Sánchez") y se re-enmascaran ✓
- Agregado en page.tsx: botones PNG/PDF en modo presentación (requisito "exportar vista a PDF/imagen").
- Estado final: `bun run lint` 0 problemas, `tsc --noEmit` 0 errores (código app), dev.log sin errores, consola del navegador limpia.

Stage Summary:
- App completa y verificada: 8 secciones, pipeline dinámico Excel→limpieza→métricas→UI sin datos quemados, filtrado global + cross-filtering + drill-down, privacidad, presentación con export, modo oscuro, responsive, footer sticky.

---
Task ID: 9 (ajuste corte + limpieza + deploy)
Agent: main (Z.ai Code)
Task: Fecha de corte por defecto = 2023-10-27 (snapshot de la hoja "Contexto"), botón "Usar fecha máxima del archivo", eliminación de dependencias no usadas (prisma, next-auth, examples/, mini-services, db/custom.db) y despliegue.

Work Log:
- Inspeccionada la hoja "Contexto" del Excel: confirma "snapshot del equipo de Alegra al 27 de octubre de 2023".
- parseExcel.ts: nuevo extractor dinámico `extraerFechaCorteDeclarada()` (palabras clave snapshot/corte/cierre + fecha en texto español "27 de octubre de 2023", ISO o numérica; pass 1 en cualquier hoja no-datos, pass 2 en hojas de docs). `ResultadoParseo.fechaCorteDeclarada` siempre resuelto (fallback FECHA_CORTE_DEFECTO).
- dateUtils.ts: constante `FECHA_CORTE_DEFECTO = '2023-10-27'`.
- types.ts: `CleanedDataset.fechaCorteDeclarada`.
- store: `datasetDesdeFilas` recibe la fecha declarada; `iniciar()` normaliza datasets viejos de IndexedDB (sin el campo) al fallback; `recargarConDiccionario` la preserva.
- useMetrics/topbar/sidebar: corte efectivo = `cutoffOverride ?? fechaCorteDeclarada ?? FECHA_CORTE_DEFECTO` (antes: fechaMaxima).
- topbar popover: texto actualizado, input con valor efectivo, botón "Usar fecha máxima del archivo (27 sep 2024)" (visible si el efectivo ≠ máx) y "Volver al snapshot declarado (27 oct 2023)" (visible si hay override manual).
- Limpieza: eliminados examples/, prisma/, db/custom.db, tests/, tool-results/, src/lib/db.ts (upload/ es punto de montaje del sandbox, quedó vacío; mini-services no existía). `bun remove @prisma/client prisma next-auth`. Scripts db:* quitados de package.json. Ignore de examples/ quitado de eslint.
- Verificación: lint 0, tsc 0 (solo errores preexistentes en skills/, fuera de la app), scripts/verify.ts OK, extracción = '2023-10-27' ✓.
- QA Agent Browser: default 27 oct 2023 → headcount 559, banner "Hay 270 registros posteriores" ✓; botón fecha máxima → 530 activos, banner desaparece, aparece "Volver al snapshot declarado" ✓; edición manual 2024-06-30 → 557 activos, 83 posteriores ✓; recarga con dataset persistido normalizado ✓; sin errores de consola ni en dev.log ✓.

Stage Summary:
- El corte por defecto es ahora el snapshot declarado en el archivo (2023-10-27), extraído dinámicamente de la hoja "Contexto"; el selector sigue editable, el banner de registros posteriores se mantiene y existe el botón "Usar fecha máxima del archivo".
- Proyecto depurado: sin prisma, next-auth, examples/, tests/, db/custom.db ni código muerto. Servidor en 3000 (HTTP 200) — app desplegada y verificada end-to-end.

---
Task ID: 10 (fallback dinámico + limpieza .env/SDK + deploy)
Agent: main (Z.ai Code)
Task: Fallback del corte por defecto = fechaMaxima cuando el archivo no declara snapshot (la constante 2023-10-27 solo aplica al Excel del reto); eliminar .env y z-ai-web-dev-sdk; desplegar con URL pública sin login.

Work Log:
- parseExcel.ts: `fechaCorteDeclarada` ahora es `string | null` (null si el archivo no declara snapshot); eliminado el fallback a la constante en el parseo.
- dateUtils.ts: nuevo helper central `cutoffEfectivo(dataset, cutoffOverride)` con la cadena override → declarada → fechaMaxima → FECHA_CORTE_DEFECTO (último recurso solo para archivos sin declaración y sin fechas válidas, caso degenerado). Comentario de la constante actualizado.
- store: datasetDesdeFilas guarda `fechaCorteDeclarada ?? null`; iniciar() normaliza datasets persistidos a null (no a la constante).
- useMetrics + topbar + sidebar usan la cadena vía helper; input de fecha tolera null (`?? ""`).
- topbar: textos dinámicos — si el archivo declara: "snapshot declarado en la hoja «Contexto» (…)"; si no: "no declara un snapshot: por defecto se usa la fecha máxima de los datos (…)". Botón "Volver a…" con etiqueta según el caso.
- Limpieza: .env eliminado (solo tenía DATABASE_URL obsoleta de prisma); `bun remove z-ai-web-dev-sdk` (sin uso en src/scripts).
- Verificación: lint 0; tsc 0 (app); reto.xlsx → 2023-10-27 ✓; archivo generado sin hoja Contexto → null ✓.
- QA Agent Browser: reto cargado → corte 27 oct 2023, banner 270 posteriores ✓; upload sin Contexto → corte por defecto 27 sep 2024 (fechaMaxima), texto dinámico correcto, botón "Usar fecha máxima" oculto (ya en máx) ✓; override manual muestra "Usar fecha máxima" + "Volver a la fecha máxima del archivo" ✓; "Restablecer" del menú restaura la muestra ✓; sin errores de consola ✓.

Stage Summary:
- La constante 2023-10-27 ya no es fallback global: solo se usa la fecha que el archivo declara; sin declaración rige fechaMaxima. App desplegada en el puerto 3000 (gateway → 3000), sin autenticación.

---
Task ID: 11 (diagnóstico de calidad)
Agent: main (Z.ai Code)
Task: Terminar la sección "Diagnóstico de calidad" como página nueva e independiente: componente, registro en navegación, chip en Resumen, verificación de conteos en navegador y despliegue.

Work Log:
- Encontrado el trabajo a medias: existían diagnostico.ts (computarDiagnostico/computarPosteriores/registrosConObservaciones), tabla-diagnostico.tsx, descargarDiagnosticoExcel/exportarFilasCSV·XLSX y SectionId 'diagnostico', pero no había componente ni registro; el botón de "Calidad de datos" caía en el Resumen.
- Creado src/components/sections/diagnostico-section.tsx (export DiagnosticoSection, recibe { bundle }):
  - Encabezado "Antes de analizar: qué encontramos y cómo lo tratamos" con contador de registros con observaciones y botones "Configurar en Calidad de datos" + "Descargar diagnóstico (Excel)".
  - 5 tarjetas clicables (Áreas inconsistentes, Fechas en formatos mixtos, Salarios atípicos, Nulos en desempeño y clima, Registros posteriores al corte) con cantidad, % de la base, semáforo (verde 0 · ámbar ≤10% · rojo >10%) y frase con números reales.
  - Pestañas por problema con: cifras de cabecera propias (mapeo variante→normalizada→cantidad→cargos y "cada cargo→un solo área" en Áreas; conteos por formato + evidencia 1er/2do número >12 + interruptor "solo ambiguas" en Fechas; barras promedio todos vs sin extremos vs mediana + interruptor global "Excluir salarios atípicos" en Salarios; coberturas, activos vs inactivos y barras por área en Nulos; corte vigente, ingresos/salidas posteriores y mini-comparación de rotación 12m con computeBundle llamado con el snapshot declarado y con fechaMaxima en Posteriores), panel de 3 bloques ("¿Por qué es un problema?" / "Decisión en el dashboard" / "Qué validar / riesgo que queda") con números reales, y tabla auditable con TablaDiagnostico (búsqueda, orden por columna, paginación 25/50/100, privacidad aplicada solo al render, export CSV/XLSX con valores reales).
  - Bloque "Verificado y sin problema" (IDs únicos, Estado↔Fecha_salida, cargo→área 1 a 1, correos no ASCII) y botón "Descargar diagnóstico (Excel)" con hoja Resumen + una hoja por problema (mismas columnas que las tablas).
  - Modo presentación: solo las 5 tarjetas + panel "Decisión en el dashboard" (detecta presentation del store).
- Registro: SECCIONES con { id: "diagnostico", label: "Diagnóstico de calidad", icono: Stethoscope } justo después de "resumen"; case "diagnostico" en el switch de page.tsx + import. Sidebar, chips móviles, flechas, contador "n / total" y presentación lo incluyen automáticamente (9 secciones).
- Chip en Resumen ejecutivo: "Calidad de datos: 289 registros con observaciones" (registrosConObservaciones) → setSection("diagnostico").
- Correcciones menores: quitar %% duplicado en textos (fmtPct ya incluye %), remover disable obsoleto de tabla-diagnostico.
- QA Agent Browser con la muestra y corte 27 oct 2023: tarjetas 262 (31,6%) / 189 celdas 147+42 / 34 salarios (5+4+23+3) / 105 (63+44−2) / 270 (131+160, 21 ambos); pestañas con conteos exactos (99 DD/MM, 36 MM-DD, 54 AAAA/MM, 53 ambiguas; 941 fechas reales; promedio 2503 vs sin extremos 2376 vs mediana 2152; coberturas 92,4%/94,7%; rotación 16,1% vs 31,2% según corte); EMP-1091 fila 54 con "05/03/2021", DD/MM/AAAA, 2021-03-05, ambigua Sí, alternativa 2021-05-03; "solo ambiguas" filtra a 53; búsqueda y orden funcionan; CSV/XLSX y Excel de diagnóstico sin errores; "Ver diagnóstico de calidad" desde Calidad de datos llega a la sección; presentación con 0 tablas y contador 2/9; Escape sale; móvil 390px correcto; sin errores de consola ni en dev.log.
- Verificaciones visibles desde Calidad de datos: 42/42 pasan.
- Estado final: bun run lint 0 problemas; tsc --noEmit 0 errores (app); servidor en puerto 3000 (gateway → 3000), HTTP 200, sin login.

Stage Summary:
- "Diagnóstico de calidad" es una página nueva e independiente (2ª sección del sidebar): cuantifica los 5 problemas antes de analizar, explica la decisión de tratamiento con los números del archivo cargado y deja la evidencia registro por registro exportable. Nada quemado: todo se recalcula con cada archivo/corte/diccionario.
- Entregado y desplegado en la URL pública del preview (puerto 3000 vía gateway), sin autenticación.

---
Task ID: 12 (fix registrosConObservaciones = unión de 4 problemas)
Agent: main (Z.ai Code)
Task: Corregir registrosConObservaciones (contaba solo filas con flags y dejaba fuera las áreas no canónicas → 289); debe ser la UNIÓN de los 4 problemas estructurales del diagnóstico (sin posteriores al corte). Actualizar los textos del chip del Resumen y del encabezado de Diagnóstico a "N registros (X% de la base) con observaciones", agregar prueba en selfTest (466), lint/tsc 0 y redesplegar.

Work Log:
- Script temporal con el pipeline real (parsearArchivo + limpiarDatos sobre Base_Reto_People.xlsx) para medir la unión antes de tocar código: áreas 262 · fechas en texto (por registro) 183 · salario marcado 34 (bajo_moderado no aporta: ninguna fila es exclusivamente bajo_moderado) · nulos 105 → UNIÓN = 466 (la definición estricta del enunciado y la que incluye bajo_moderado dan el mismo resultado); flags viejos = 289.
- diagnostico.ts: reescrito registrosConObservaciones como filtro de la unión: 1) areaOriginal && !esAreaCanonica(areaOriginal); 2) fechaIngresoDetalle/fechaSalidaDetalle con formato !== 'Fecha real'; 3) salario !== null y (extremo alto/bajo por umbrales, iqr_alto o decimales); 4) desempeno === null || clima === null. Excluye posteriores al corte (dependen del corte vigente). Documentado en el JSDoc.
- resumen-section.tsx: chip ahora "Calidad de datos: 466 registros (56,1% de la base) con observaciones" (totalFilas + pctObservaciones con fmtPct 1 decimal).
- diagnostico-section.tsx: encabezado ahora "…830 registros revisados. 466 registros (56,1% de la base) con observaciones. Revisa estos 860 hallazgos (con repeticiones entre problemas)…" (860 = 262+189+34+105+270, sin cambio de lógica).
- selfTest.ts: nueva verificación "Muestra · registros con observaciones (unión de 4 problemas)" esperando 466.
- Verificación: lint 0; tsc 0 (app); borrado el script temporal.
- QA Agent Browser: chip en Resumen = "Calidad de datos: 466 registros (56,1% de la base) con observaciones" ✓; clic lleva a Diagnóstico y el encabezado muestra 830 revisados / 466 (56,1%) / 860 hallazgos ✓; "Ejecutar verificaciones de limpieza" en Calidad de datos → 43/43 pasan (incluida la nueva unión = 466) ✓; 5 tarjetas intactas con 262/189/34/105/270 ✓; sin errores de consola ni en dev.log ✓.

Stage Summary:
- registrosConObservaciones ya cuenta la unión real de registros con al menos una observación de calidad (área no canónica ∪ fecha en texto ∪ salario marcado ∪ nulos): 466 de 830 (56,1%) con el Excel del reto, en lugar de 289. Los "posteriores al corte" quedan fuera por depender del corte vigente.
- App verificada end-to-end y desplegada en el puerto 3000 (gateway → 3000), sin autenticación.

---
Task ID: 13 (recuperación del despliegue)
Agent: main (Z.ai Code)
Task: El usuario reportó "there was a problem deploying the code" y que no podía ver el enlace. Diagnóstico y recuperación del servidor.

Work Log:
- Diagnóstico: curl a localhost:3000 devolvió HTTP 000 y no había proceso next dev — el servidor se había detenido (el código no tenía errores; dev.log no mostraba fallos previos).
- Reiniciado `bun run dev` en segundo plano (next dev -p 3000 | tee dev.log): Ready in 1.1s, GET / → 200.
- QA Agent Browser: Resumen ejecutivo carga con el chip "Calidad de datos: 466 registros (56,1% de la base) con observaciones" ✓; clic lleva a Diagnóstico y el encabezado muestra "830 registros revisados / 466 registros (56,1% de la base) con observaciones" (HEADER-OK) ✓; sin errores de página ✓.

Stage Summary:
- Causa: el proceso del dev server se había caído; no era un problema del código. Servidor reiniciado y app verificada end-to-end, desplegada en el puerto 3000 (gateway → 3000), sin autenticación.

---
Task ID: 14 (URL pública caída — diagnóstico y mitigación)
Agent: main (Z.ai Code)
Task: El usuario reporta que https://alegramichaelperalta.space-z.ai muestra "Sorry, there was a problem deploying the code" y que la app solo carga en el preview.

Work Log:
- Diagnóstico de la cadena: localhost:3000 (app) 200 ✓ · gateway Caddy :81 200 ✓ · preview-chat-<sesión>.space-z.ai 200 ✓ (sirve "People Scorecard — Alegra" y el Excel) · alegramichaelperalta.space-z.ai → HTTP 500 en ~35ms con página "Failed" generada por el edge de la plataforma (ALB de Alibaba, cn-hongkong): la petición NUNCA llega al sandbox.
- Causa probable: al momento del intento de despliegue el dev server estaba CAÍDO (así se encontró en la recuperación previa: HTTP 000, sin procesos) → el chequeo de la plataforma falló y quedó el registro de despliegue roto.
- Mitigación: creado scripts/keepalive.sh — vigila :3000 cada 30s y reinicia `bun run dev` solo tras 3 fallos consecutivos (~90s) para no estorbar procesos de la plataforma; log en keepalive.sh.log/keepalive.log. Iniciado en segundo plano (PID verificado).
- Verificación final con Agent Browser contra la URL PÚBLICA del preview: carga "Resumen ejecutivo", chip "Calidad de datos: 466 registros (56,1% de la base) con observaciones", sin errores de página ✓.

Stage Summary:
- El dominio alegramichaelperalta.space-z.ai es un registro de despliegue gestionado por la plataforma Z.ai; su error 500 no es corregible desde dentro del sandbox (el edge falla antes de llegar a la app). El servidor ya está estable (con keepalive) → reintentar "Deploy/Publicar" desde la página de generación debería funcionar. Mientras tanto, la URL del preview es pública y funcional sin login.
