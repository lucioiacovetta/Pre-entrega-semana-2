# Pre-entrega-semana-2


## Objetivo
Simular un escenario real donde los datos de ventas provienen de distintos departamentos
en formatos diferentes (un CSV de transacciones y un Excel con el catálogo de productos),
e integrarlos y normalizarlos con Python.

## Proceso

### Paso 1 – Búsqueda y preparación de los datos
La primera idea fue trabajar con un dataset argentino. Busqué en Kaggle, en datos.gob.ar
y en Precios Claros (SEPA), pero no encontré datos públicos de ventas a nivel transacción:
lo que existe en Argentina son índices agregados (INDEC) o listas de precios sin cantidades
vendidas (SEPA), que no alcanzaban para cumplir la consigna.

Por eso usé un dataset real parecido: **Online Retail** del UCI Machine Learning Repository
(licencia CC BY 4.0), con ventas reales de un comercio online del Reino Unido entre
diciembre de 2010 y diciembre de 2011.

Como el dataset original viene en un único Excel, le pedí a Claude (asistente de IA de
Anthropic) que me ayudara a separarlo en los dos archivos que pide la consigna:
- `transacciones.csv`: id_factura, id_producto, cantidad, fecha, id_cliente, pais.
- `maestro_productos.xlsx`: id_producto, nombre_producto, precio_unitario. Como un mismo
  producto aparecía con varios precios y descripciones, se tomó el nombre y el precio
  más frecuentes de cada uno.

Los datos no se limpiaron en esta etapa, para hacer la limpieza en el notebook.

### Paso 2 – Configuración
- Creé el repositorio en GitHub y subí los dos archivos.
- El CSV original pesaba demasiado para GitHub, así que lo recorté a una muestra
  (01/12/2010 al 17/01/2011).
- Trabajé en Google Colab, leyendo los archivos directamente desde el repositorio
  (link "raw" de GitHub). Para eso el repositorio tiene que ser público.

### Paso 3 – Adquisición
- CSV leído con `pd.read_csv`, indicando `id_factura` e `id_producto` como texto.
- Excel leído con `pd.read_excel` usando el motor `openpyxl`.

### Paso 4 – Transformación
**Tipos de datos**
- Al recortar el CSV en Excel quedaron filas vacías: había 541.909 filas y solo
  61.540 con datos. Se eliminaron con `dropna(how="all")`.
- La fecha venía como texto con formato `01-12-10 8:26`. Se convirtió a datetime con
  formato explícito `%d-%m-%y %H:%M`, porque sin indicarlo pandas podía leer el
  1 de diciembre como 12 de enero.
- `cantidad` pasó a entero e `id_cliente` a `Int64`.
- `id_producto` se unificó como texto, sin espacios y en mayúsculas, en ambas tablas,
  para evitar el error de hacer un merge con llaves de distinto tipo.

**Valores nulos**
- `id_cliente` (23.491 nulos, 38%): imputado con 0 = cliente no identificado. La venta
  ocurrió igual, y eliminarlas hacía perder casi 4 de cada 10 ventas.
- `precio_unitario` (132 nulos): productos eliminados, porque sin precio no se puede
  calcular el total.
- `nombre_producto` (112 nulos): imputado con "SIN DESCRIPCION", ya que no interviene
  en ningún cálculo.
- Al pasar los códigos a mayúsculas aparecieron 110 IDs duplicados en el maestro
  (ej. `15056bl` y `15056BL`). Se eliminaron para que el merge no duplicara ventas.

**Merge**
- Left join entre ventas y productos usando `id_producto` como llave, con
  `validate="many_to_one"` para asegurar que cada producto aparezca una sola vez
  en el maestro.
- Resultado: 61.522 ventas cruzadas y 18 sin producto (productos eliminados por no
  tener precio), que se descartaron. La cantidad de filas no aumentó, lo que confirma
  que no hubo duplicación.

### Paso 5 – Normalización
- Nueva columna `total_venta = cantidad * precio_unitario`.
- Las 1.134 ventas con total negativo son devoluciones o cancelaciones (facturas que
  empiezan con "C"). Se conservaron para que el total refleje la venta neta.

### Paso 6 – Exportación
- Exporté el resultado a `ventas_integradas.parquet` y lo descargué desde Colab.
- Parquet conserva los tipos de datos (la fecha sigue siendo datetime) y ocupa
  menos espacio que un CSV.
- El archivo no se abre con doble clic: se lee con `pd.read_parquet(...)`.

## Archivos del repositorio
| Archivo | Descripción |
|---|---|
| `transacciones.csv` | Transacciones de venta (muestra) |
| `maestro_productos.xlsx` | Catálogo de productos |
| `*.ipynb` | Notebook de Colab con todo el proceso |
| `ventas_integradas.parquet` | Resultado final integrado |

## Limitaciones
- Se usa una muestra de aproximadamente 6 semanas por el límite de tamaño de GitHub.
- El dataset original tiene 639 filas duplicadas exactas que no se eliminaron.
- Los precios están en libras esterlinas.

## Requisitos
pandas, openpyxl, pyarrow (ya incluidos en Google Colab).

## Fuente
Online Retail – UCI Machine Learning Repository:
https://archive.ics.uci.edu/dataset/352/online+retail
