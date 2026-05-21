# 📊 Estado de Resultados (P&L) — Power BI

Proyecto de Business Intelligence que implementa un Estado de Resultados interactivo en Power BI utilizando únicamente visuales nativos, una tabla auxiliar y medidas DAX.

> Basado en el artículo y tutorial de **[Valerie Junk](https://www.valeriejunk.nl/profit-and-loss-statement-in-power-bi/)** — *Profit and Loss Statement in Power BI*. El enfoque estructural, el patrón de Helper Table y la lógica del Switch DAX provienen de su trabajo original. Esta implementación adapta el modelo al español, extiende la lógica comparativa año actual vs. año anterior con títulos dinámicos, y simplifica la medida Switch eliminando la variable `isPLID`.

---

## 📸 Vista previa

![Dashboard Estado de Resultados](./assets/dashboard_preview.png)

---

## 📌 Descripción del proyecto

El dashboard presenta el Estado de Resultados de una empresa en dos vistas complementarias:

- **Desglose mensual** — Ingresos, costos y utilidades columna por columna por cada mes del año seleccionado.
- **Análisis comparativo** — Año actual vs. año anterior vs. diferencia (Δ), con títulos que se actualizan automáticamente según los datos disponibles.

Ambas matrices comparten la misma jerarquía de filas definida por la `Tabla_Auxiliar_PyG`, que controla el orden, las categorías y las filas calculadas del estado de resultados.

---

## 💡 El problema que resuelve

La matriz nativa de Power BI solo acepta un tipo de dato por columna de valores y no permite mezclar montos, porcentajes y subtotales calculados en la misma visualización. Este proyecto resuelve eso con tres elementos:

1. Una **tabla auxiliar** que define el layout exacto de filas.
2. **IDs virtuales** (997, 998, 999) para filas calculadas que no existen en los datos fuente.
3. Una **medida Switch** que lee el ID de cada fila y devuelve la medida correspondiente.

---

## 🗂️ Arquitectura del modelo

```
DIM_Calendario ──────────────────────────────────────┐
                                                      │
Dim_Cuenta ─────────────── Fct_Transacciones ─────────┤
               │                                      │
               └── Tabla_Auxiliar_PyG                 │
                   (conectada vía ID_Cuenta)          │
                                                      │
P&L Auxiliar ────────────────────────────────────────┘
(tabla desconectada — slicer de vista comparativa)
```

---

## 📐 Tablas del modelo

### `Fct_Transacciones`
Tabla de hechos principal. Contiene todas las transacciones del período.

> ⚠️ Los costos (COGS y Gastos Operativos) se almacenan como **valores negativos**. Las medidas base los multiplican por `-1` para presentarlos como positivos en el estado de resultados.

### `Dim_Cuenta`
Dimensión de cuentas contables con `ID_Cuenta`, `Categoria` y `Cuenta`.

### `DIM_Calendario`
Tabla de fechas estándar. Requerida para `SAMEPERIODLASTYEAR`.

### `Tabla_Auxiliar_PyG`
Define el layout de la matriz: qué filas aparecen, en qué orden y bajo qué categoría. Incluye IDs virtuales para filas calculadas.

| ID_Cuenta | Tipo | Descripción |
|---|---|---|
| 1 – 8 | Dato real | Corresponde a cuentas en `Fct_Transacciones` |
| 997 | Virtual | Margen Bruto % (porcentaje calculado) |
| 998 | Virtual | Utilidad Neta (calculada) |
| 999 | Virtual | Utilidad Bruta (calculada) |

### `P&L Auxiliar`
Tabla desconectada de 3 filas usada como slicer de la matriz comparativa. Sus valores (`Año Actual`, `Año Anterior`, `Diff`) se generan dinámicamente desde los datos — no son texto fijo.

---

## 📏 Medidas DAX

Las medidas están organizadas en 6 capas de construcción. Solo se documentan las presentes en el dashboard.

---

### Capa 1 — Medida base

#### `Monto Total (Bruto)`
- **Carpeta:** Ingresos
- **Descripción:** Suma bruta de todas las transacciones. Incluye ingresos y costos. Usada internamente por todas las medidas base y para la verificación `HasData`.
- **Formato:** `#,0`
```dax
Monto Total (Bruto) = 
--Se incluyen gastos e ingresos, esta es una medida de soporte
SUM(Fct_Transacciones[Monto])
```

---

### Capa 2 — Medidas por cuenta (Año Actual)

Filtran `Monto Total (Bruto)` por nombre de cuenta. Los costos multiplican por `-1` para invertir el signo negativo del origen.

#### `Ingreso Venta de Productos`
- **Carpeta:** Ingresos | **Formato:** `#,0`
```dax
Ingreso Venta de Productos = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Ventas de Productos"
)
```

#### `Ingreso por Venta de Servicios`
- **Carpeta:** Ingresos | **Formato:** `#,0`
```dax
Ingreso por Venta de Servicios = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Ingresos por Servicios"
)
```

#### `Ingreso por otras actividades`
- **Carpeta:** Ingresos
```dax
Ingreso por otras actividades = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Otros Ingresos"
)
```

#### `Costo Materiales`
- **Carpeta:** Costos | **Formato:** `#,0`
```dax
Costo Materiales = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Materiales"
) * -1
```

#### `Costo Mano de Obra Directa`
- **Carpeta:** Costos | **Formato:** `#,0`
```dax
Costo Mano de Obra Directa = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Mano de Obra Directa"
) * -1
```

#### `Costo de Embalaje`
- **Carpeta:** Costos | **Formato:** `#,0`
```dax
Costo de Embalaje = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Embalaje"
) * -1
```

#### `Costo de Marketing`
- **Carpeta:** Costos | **Formato:** `#,0`
```dax
Costo de Marketing = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Marketing"
) * -1
```

#### `Costo de Instalaciones`
- **Carpeta:** Costos
```dax
Costo de Instalaciones = 
CALCULATE(
    [Monto Total (Bruto)],
    Dim_Cuenta[Cuenta] = "Arrendamiento de Oficina"
) * -1
```

---

### Capa 3 — Medidas agregadas (Año Actual)

Combinan las medidas base para producir totales y KPIs calculados del P&L.

#### `Ingresos Totales (General)`
- **Carpeta:** Ingresos
```dax
Ingresos Totales (General) = 
[Ingreso Venta de Productos] + [Ingreso por Venta de Servicios] + [Ingreso por otras actividades]
```

#### `Costos Total - Operativos (General)`
- **Carpeta:** Costos
```dax
Costos Total - Operativos (General) = 
[Costo Mano de Obra Directa] + [Costo Materiales] + [Costo de Embalaje]
```

#### `Costo Total - Administrativos (General)`
- **Carpeta:** Costos
```dax
Costo Total - Administrativos (General) = 
[Costo de Marketing] + [Costo de Instalaciones]
```

#### `Utilidad bruta`
- **Carpeta:** Utilidad
```dax
Utilidad bruta = 
[Ingresos Totales (General)] - [Costos Total - Operativos (General)]
```

#### `Margen bruto % (General)`
- **Carpeta:** Utilidad | **Formato:** `0.00 %;-0.00 %;0.00 %`
```dax
Margen bruto % (General) = 
-- Margen bruto como porcentaje de los ingresos
DIVIDE(
    [Utilidad bruta],
    [Ingresos Totales (General)]
)
```

#### `Utilidad neta`
- **Carpeta:** Utilidad
```dax
Utilidad neta = 
[Utilidad bruta] - [Costo Total - Administrativos (General)]
```

---

### Capa 4 — Medidas Año Anterior

Todas usan `SAMEPERIODLASTYEAR` sobre `DIM_Calendario[Fecha]` para trasladar el contexto al mismo período del año anterior.

#### `Ingreso Total (Año Anterior)`
- **Carpeta:** Año_Anterior
```dax
Ingreso Total (Año Anterior) = 
CALCULATE(
    [Ingresos Totales (General)],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### `Costo Operativo Total (Año Anterior)`
- **Carpeta:** Año_Anterior
```dax
Costo Operativo Total (Año Anterior) = 
CALCULATE(
    [Costos Total - Operativos (General)],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### `Costo Administrativo Total (Año Anterior)`
- **Carpeta:** Año_Anterior
```dax
Costo Administrativo Total (Año Anterior) = 
CALCULATE(
    [Costo Total - Administrativos (General)],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### `Utilidad bruta (Año Anterior)`
- **Carpeta:** Año_Anterior
```dax
Utilidad bruta (Año Anterior) = 
CALCULATE(
    [Utilidad bruta],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### `Margen bruto % (Año Anterior)`
- **Carpeta:** Año_Anterior | **Formato:** `0.0 %;-0.0 %;0.0 %`
```dax
Margen bruto % (Año Anterior) = 
CALCULATE(
    [Margen bruto % (General)],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### `Utilidad neta (Año Anterior)`
- **Carpeta:** Año_Anterior
```dax
Utilidad neta (Año Anterior) = 
CALCULATE(
    [Utilidad neta],
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)
```

#### Medidas Año Anterior por línea individual

Requeridas por `P&L Switch (Año anterior) - Sin Formato`. Todas siguen el mismo patrón:

```dax
-- Patrón: CALCULATE([medida_base], SAMEPERIODLASTYEAR(DIM_Calendario[Fecha]))
Ingreso Venta de Productos (Año Anterior)
Ingreso Venta de Servicio (Año Anterior)
Ingreso por otras actividades (Año Anterior)
Costo de Materiales (Año Anterior)
Costo Mano de Obra Directa (Año Anterior)
Costo de Embalaje (Año Anterior)
Costo de Marketing (Año Anterior)
Costo de Instalaciones (Año Anterior)
```

---

### Capa 5 — Medidas Delta (Tarjetas del header)

Generan el texto `(🡹 X,XXX $)` o `(🡻 X,XXX $)` que aparece como subtítulo en las tarjetas KPI. Usan `UNICHAR(129145)` (🡹) y `UNICHAR(129147)` (🡻).

#### `Diferencia Ingresos (Delta Formato Especifico)`
- **Carpeta:** KPIs
```dax
Diferencia Ingresos (Delta Formato Especifico) = 
VAR _Valores = [Ingresos Totales (General)] - [Ingreso Total (Año Anterior)]
VAR _Arrow = IF(_Valores >= 0, UNICHAR(129145), UNICHAR(129147))
RETURN "(" & _Arrow & " " & FORMAT(_Valores, "#,0 $") & ")"
```

#### `Diferencia Costos Operativos (Delta Formato Especifico)`
- **Carpeta:** KPIs
```dax
Diferencia Costos Operativos (Delta Formato Especifico) = 
VAR _Valores = [Costos Total - Operativos (General)] - [Costo Operativo Total (Año Anterior)]
VAR _Arrow = IF(_Valores >= 0, UNICHAR(129145), UNICHAR(129147))
RETURN "(" & _Arrow & " " & FORMAT(_Valores, "#,0 $") & ")"
```

#### `Diferencia Margen Bruto % (Delta Formato Especifico)`
- **Carpeta:** KPIs
```dax
Diferencia Margen Bruto % (Delta Formato Especifico) = 
VAR _Valores = [Margen bruto % (General)] - [Margen bruto % (Año Anterior)]
VAR _Arrow = IF(_Valores >= 0, UNICHAR(129145), UNICHAR(129147))
RETURN "(" & _Arrow & " " & FORMAT(_Valores, "0.00%") & ")"
```

#### `Diferencia Costos Administrativos (Delta Formato Especifico)`
- **Carpeta:** KPIs
```dax
Diferencia Costos Administrativos (Delta Formato Especifico) = 
VAR _Valores = [Costo Total - Administrativos (General)] - [Costo Administrativo Total (Año Anterior)]
VAR _Arrow = IF(_Valores >= 0, UNICHAR(129145), UNICHAR(129147))
RETURN "(" & _Arrow & " " & FORMAT(_Valores, "#,0 $") & ")"
```

#### `Diferencia Utilidad Neta (Delta Formato Especifico)`
- **Carpeta:** KPIs
```dax
Diferencia Utilidad Neta (Delta Formato Especifico) = 
VAR _Valores = [Utilidad neta] - [Utilidad neta (Año Anterior)]
VAR _Arrow = IF(_Valores >= 0, UNICHAR(129145), UNICHAR(129147))
RETURN "(" & _Arrow & " " & FORMAT(_Valores, "#,0 $") & ")"
```

> Las versiones `(Sin formato)` de estas mismas medidas (`Diferencia Ingresos (Sin formato)`, etc.) devuelven el valor numérico sin texto. Se usan para formato condicional de color en las tarjetas — si el valor es positivo la etiqueta se colorea verde, si es negativo roja.

---

### Capa 6 — Medidas Switch (Motor de la Matriz)

Son el corazón del dashboard. Permiten que una sola columna de valores en la matriz muestre líneas heterogéneas del P&L.

---

#### `P&L Switch (General) - Sin formato`
- **Carpeta:** Matriz
- **Descripción:** Versión base del Switch para el año actual. Itera los IDs visibles en el contexto de fila con `SUMX` y devuelve la medida correspondiente a cada uno. Devuelve número puro sin formato.
```dax
P&L Switch (General) - Sin formato = 
VAR HasData = CALCULATE(NOT ISBLANK([Monto Total (Bruto)]))

RETURN
IF(
    NOT HasData,
    BLANK(),
    SUMX(
        VALUES(Tabla_Auxiliar_PyG[ID_Cuenta]),
        SWITCH(
            Tabla_Auxiliar_PyG[ID_Cuenta],
            1,   [Ingreso Venta de Productos],
            2,   [Ingreso por Venta de Servicios],
            3,   [Ingreso por otras actividades],
            4,   [Costo Materiales],
            5,   [Costo Mano de Obra Directa],
            6,   [Costo de Embalaje],
            999, [Utilidad bruta],
            997, [Margen bruto % (General)],
            7,   [Costo de Marketing],
            8,   [Costo de Instalaciones],
            998, [Utilidad neta],
            BLANK()
        )
    )
)
```

---

#### `P&L Switch (Año anterior) - Sin Formato`
- **Carpeta:** Matriz
- **Descripción:** Idéntico al anterior pero referencia las medidas de año anterior. La verificación `HasData` también aplica `SAMEPERIODLASTYEAR` para confirmar que existen datos en el año anterior antes de calcular.
```dax
P&L Switch (Año anterior) - Sin Formato = 
VAR HasData = 
CALCULATE(
    NOT ISBLANK([Monto Total (Bruto)]),
    SAMEPERIODLASTYEAR(DIM_Calendario[Fecha])
)

RETURN
IF(
    NOT HasData,
    BLANK(),
    SUMX(
        VALUES(Tabla_Auxiliar_PyG[ID_Cuenta]),
        SWITCH(
            Tabla_Auxiliar_PyG[ID_Cuenta],
            1,   [Ingreso Venta de Productos (Año Anterior)],
            2,   [Ingreso Venta de Servicio (Año Anterior)],
            3,   [Ingreso por otras actividades (Año Anterior)],
            4,   [Costo de Materiales (Año Anterior)],
            5,   [Costo Mano de Obra Directa (Año Anterior)],
            6,   [Costo de Embalaje (Año Anterior)],
            999, [Utilidad bruta (Año Anterior)],
            997, [Margen bruto % (Año Anterior)],
            7,   [Costo de Marketing (Año Anterior)],
            8,   [Costo de Instalaciones (Año Anterior)],
            998, [Utilidad neta (Año Anterior)],
            BLANK()
        )
    )
)
```

---

#### `P&L Switch UltimoAño/AñoAnterior/Diff`
- **Carpeta:** Matriz
- **Descripción:** **Medida final de la matriz comparativa.** Lee el valor del slicer `P&L Auxiliar[Categoria]` y decide qué devolver: año actual, año anterior o diferencia. Los valores del slicer no son texto fijo sino que se generan dinámicamente desde `[Ultima fecha]` — si los datos van hasta 2024, el slicer muestra "2024", "2023" y "Diff". El ID 997 (Margen Bruto %) recibe formato de porcentaje; el resto se formatea como moneda.
```dax
P&L Switch UltimoAño/AñoAnterior/Diff = 
VAR ViewType = SELECTEDVALUE('P&L Auxiliar'[Categoria])
VAR PLID     = SELECTEDVALUE(Tabla_Auxiliar_PyG[ID_Cuenta])

VAR Actuals  = [P&L Switch (General) - Sin formato]
VAR LYTD     = [P&L Switch (Año anterior) - Sin Formato]

VAR fechamax  = FORMAT([Ultima fecha], "0")
VAR lastyear  = FORMAT([Ultima fecha] - 1, "0")

VAR Result = 
    SWITCH(
        TRUE(),
        ViewType = fechamax, Actuals,
        ViewType = lastyear, LYTD,
        ViewType = "Diff",   Actuals - LYTD
    )

RETURN 
IF(
    ISBLANK(Result),
    BLANK(),
    SWITCH(
        TRUE(),
        PLID = 997, FORMAT(Result, "0.00 %"),
        FORMAT(Result, "$#,##0")
    )
)
```

---

### Medidas de presentación

#### `Ultima fecha`
- **Carpeta:** Matriz
- **Descripción:** Devuelve el año más reciente disponible en `Fct_Transacciones` ignorando todos los filtros. Ancla temporal usada por los títulos dinámicos y el Switch comparativo.
```dax
Ultima fecha = 
CALCULATE(
    YEAR(MAX(Fct_Transacciones[Fecha])),
    ALL(Fct_Transacciones[Fecha])
)
```

#### `Titulo matriz por mes`
- **Descripción:** Título dinámico de la matriz de desglose mensual. Se actualiza automáticamente con el año de los datos.
```dax
Titulo matriz por mes = 
VAR maxfecha = FORMAT([Ultima fecha], "0")
RETURN "P&L: Desglose Mensual - " & maxfecha
```

#### `Titulo matriz comparativa`
- **Descripción:** Título dinámico de la matriz comparativa. Muestra los dos años que se están comparando.
```dax
Titulo matriz comparativa = 
VAR fechamax  = [Ultima fecha]
VAR anterior  = fechamax - 1
VAR actual    = FORMAT(fechamax, "0")
VAR pasada    = FORMAT(anterior, "0")
RETURN "Análisis Comparativo " & actual & " vs " & pasada
```

#### `Fondo Transparente`
- **Descripción:** Devuelve el string `"rgba(0,0,0,0)"`. Usada para establecer fondos transparentes en elementos visuales vía formato condicional.
```dax
Fondo Transparente = "rgba(0,0,0,0)"
```

---

## 🔄 Diferencias respecto al trabajo original de Valerie Junk

| Aspecto | Versión original | Esta implementación |
|---|---|---|
| **Idioma** | Inglés | Español |
| **Medida Switch** | Usa `isPLID` + `HASONEVALUE` para distinguir filas de detalle vs. encabezado de categoría | Simplificada con `SUMX` directo — `HASONEVALUE` eliminado ya que en una jerarquía de dos niveles bien definida el contexto siempre es limpio |
| **Slicer comparativo** | Valores de texto fijos ("TY", "LY", "Delta") | Valores dinámicos generados desde `[Ultima fecha]` — el slicer muestra los años reales de los datos |
| **Títulos** | Estáticos | Dinámicos — se actualizan automáticamente con el año del último dato cargado |
| **IDs virtuales** | 991, 992, 993 | 997, 998, 999 |
| **Nombres de medidas** | En inglés (Revenue, COGS, Net Profit) | Adaptados al contexto contable en español |

---

## 🛠️ Stack tecnológico

| Herramienta | Uso |
|---|---|
| **Power BI Desktop** | Desarrollo del dashboard y visualizaciones |
| **DAX** | Medidas, lógica del Switch y títulos dinámicos |
| **Power Query (M)** | Transformación y carga de datos |
| **SQL Server** | Base de datos fuente |

---

## 📄 Créditos

Este proyecto está basado en el tutorial de **Valerie Junk**:
- 📝 Artículo: [Profit and Loss Statement in Power BI](https://www.valeriejunk.nl/profit-and-loss-statement-in-power-bi/)
- 🎥 Video: [YouTube](https://www.youtube.com/watch?v=mg94G2520G8)

---

## 👤 Autor

**Diego L. Villavicencio Merino**
Economista | Analista de Datos

[![github](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/diegovillavicencio1208-ops)
[![linkedin](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/diegovillavicenciodl/)

---
