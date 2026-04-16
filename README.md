# ONPE 2026 — Extrapolación de la Elección Presidencial

Extrapolación en tiempo real de la **Elección Presidencial del Perú 2026** (primera vuelta) a partir de los resultados parciales de la ONPE. El procedimiento escala los votos contabilizados (`actas contabilizadas`) al total esperado (`total actas`) usando una estrategia de imputación jerárquica para los distritos y ciudades del extranjero que aún no han reportado.

> **Última instantánea:** ONPE al **92.950 %** de `actas contabilizadas` — archivo `onpe_2026_complete_20260416T1209.csv` (descargado el 2026-04-16, 12:09).

---

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `Analisis_ONPE_2026.R` | Script de extrapolación (R). Lee el CSV de la ONPE, imputa los distritos/ciudades faltantes y genera los votos extrapolados por distrito junto con un reporte nacional. |
| `onpe_2026_complete_<TIMESTAMP>.csv` | Instantánea del conteo oficial de la ONPE, parseada con una fila por distrito (nacional) o por ciudad del extranjero. El nombre del archivo lleva la marca temporal de descarga (`AAAAMMDDTHHMM`). |
| `geordir-ubigeo-distritos_csv.csv` | Tabla de referencia con los códigos UBIGEO de los 1,895 distritos nacionales del Perú (códigos INEI/RENIEC, región, capital, superficie, densidad, altitud, lat/lon, IDH, índices de pobreza). Se usa para unir los resultados de la ONPE con covariables geográficas y socioeconómicas. Fuente: GeoDIR. |

El script de descarga (browser-side) que genera el CSV de la ONPE **no** está incluido en este repositorio.

---

## Esquema del CSV de la instantánea (`onpe_2026_complete_*.csv`)

2,102 filas: 1,892 distritos nacionales + 210 ciudades de votación en el extranjero.

| Columna | Descripción |
|---|---|
| `departamento`, `provincia`, `distrito` | Nombres administrativos. Las filas del extranjero usan `EXT-<continente>` / país / `CITY_<código>`. |
| `dist_code`, `prov_code`, `dept_code` | Nacional: UBIGEO estándar. Extranjero: `dept_code` = continente, `prov_code` = país, `dist_code` = ciudad. |
| `pct_actas` | % de actas esperadas ya contabilizadas en esa fila. |
| `actas_contabilizadas`, `total_actas` | Actas contabilizadas frente al total esperado. |
| `votos_<Candidato>`, `pct_<Candidato>` | Votos brutos y % de votos válidos por candidato (5 candidatos, ver abajo). |
| `type` | `domestic` o `extranjero`. |

### Candidatos considerados

| Código | Candidato | Partido |
|---|---|---|
| FUJ | Keiko Fujimori | Fuerza Popular |
| RLA | Rafael López Aliaga | Renovación Popular |
| NTO | Jorge Nieto | Buen Gobierno |
| BLM | Ricardo Belmont | Cívico Obras |
| RSP | Roberto Sánchez | Juntos por el Perú |

---

## Metodología de extrapolación

El script trata por separado los votos nacionales y los del extranjero porque sus jerarquías geográficas son distintas.

### A. Extranjero — a nivel ciudad, imputación jerárquica

Para cada ciudad del extranjero, el método se selecciona en este orden:

1. **`direct`** — `pct_actas > 0` → escalar `votos_X` por `100 / pct_actas`.
2. **`imputed_country`** — la ciudad no tiene actas, pero otras ciudades del mismo país sí → imputar usando los votos-por-acta (VPA) y las cuotas por candidato agregadas a nivel país.
3. **`imputed_continent`** — el país no tiene datos en ninguna ciudad → recurrir al VPA y cuotas del continente.
4. **`skipped`** — no se esperan actas.

El VPA y las cuotas agregadas se calculan a partir de los **votos válidos estimados** en las ciudades con datos. Los votos válidos estimados por fila usan el primer candidato con un par `(votos, pct)` distinto de cero, lo que hace al estimador robusto frente a candidatos con cero votos en una ciudad.

### B. Nacional — a nivel distrito, fallback provincia → departamento

1. **`direct`** — `pct_actas > 0` → escalar por `100 / pct_actas`.
2. **`province`** — imputar a partir de las tasas agregadas de la provincia (VPA + cuotas).
3. **`department`** — recurrir a las tasas del departamento si la provincia no tiene reportes.
4. **`skipped`** — no se esperan actas.

**Exclusión de las grandes ciudades del pool de imputación.** Dentro de cada provincia, los distritos del cuartil superior por `total_actas` (o el más grande, si la provincia solo tiene 2–3 distritos reportando) se marcan como `is_big_city` y se excluyen al calcular los promedios de provincia/departamento. Justificación: los distritos urbanos grandes votan de manera sistemáticamente distinta a los distritos rurales para los que se usarían como referencia, e incluirlos sesga las imputaciones de los distritos pequeños hacia las preferencias de los candidatos urbanos.

### C. Total nacional

Los votos extrapolados nacionales y del extranjero se suman por candidato. El denominador del `% válido` nacional es la suma de (i) votos válidos extrapolados por departamento (nacional) y (ii) votos válidos extrapolados por ciudad (extranjero, usando la misma jerarquía que para los votos por candidato).

---

## Cómo ejecutarlo

```r
# Dependencias: readr (única)
install.packages("readr")

# Editar Analisis_ONPE_2026.R:
#   - setwd(...) a la raíz del repositorio
#   - CSV_FILE <- "onpe_2026_complete_<TIMESTAMP>.csv"

Rscript Analisis_ONPE_2026.R
```

### Salidas

| Archivo | Descripción |
|---|---|
| `onpe_2026_extrapolated_full.csv` | Una fila por distrito/ciudad con las columnas `extrap_<Candidato>` y el `method` de imputación utilizado. |
| `onpe_extrapolation_output.txt` | Log en texto plano: promedios por país/continente/provincia/departamento, extrapolaciones por ciudad, totales nacionales, proyección de segunda vuelta, desglose por departamento y desglose por país en el extranjero. |

---

## Notas y limitaciones

- La extrapolación asume que **los distritos no reportados dentro de una provincia (o las ciudades dentro de un país) votan como los que sí han reportado**. Cuando las zonas que reportan tarde difieren sistemáticamente — común en distritos rurales remotos — la proyección estará sesgada. La exclusión de las grandes ciudades es una corrección parcial de este problema en el lado nacional.
- El fallback a nivel continente para el extranjero es una imputación gruesa; solo se activa cuando un país entero tiene cero actas contabilizadas.
- El script no es bayesiano y reporta únicamente estimaciones puntuales. No incluye cuantificación de incertidumbre.
- El CSV de la ONPE es una instantánea; volver a ejecutar el script con una descarga posterior cambiará los números.
