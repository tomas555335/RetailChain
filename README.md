# Consolidador de Inventario — RetailChain (UNION vs. UNION ALL)

Este proyecto analiza el comportamiento y las diferencias operativas entre `UNION` y `UNION ALL` en SQL para consolidar datos de inventario entre distintas sucursales de la cadena **RetailChain**.

---

## Preguntas Técnicas y Análisis de Resultados

### 1. ¿Cuántas filas devuelve cada consulta y por qué son distintas? Explicá con ejemplos concretos de los datos qué filas se eliminaron con UNION.

* **`UNION ALL` devuelve 14 filas:** La Sucursal Norte aporta 7 registros y la Sucursal Sur aporta 7 registros ($7 + 7 = 14$). Al apilar directamente todas las filas sin filtros ni procesamiento extra, conserva la totalidad de las entradas de ambas tablas.
* **`UNION` también devuelve 14 filas:** No elimina ninguna fila porque **no existe ningún registro que sea 100% idéntico** en todas las columnas seleccionadas (`id_producto`, `nombre_producto`, `categoria` y `stock`).

#### Análisis detallado con los datos del ejercicio:
* **Diferencia por stock en mismo producto:** Los productos **103** (`Monitor 4K 27"`), **104** (`Teclado Mecánico`) y **106** (`SSD Externo 1TB`) figuran en ambas sucursales, pero tienen valores distintos en la columna `stock` (ej. en el producto 106, Norte tiene 10 unidades y Sur tiene 7 unidades). Como `UNION` evalúa la fila completa, al diferir el stock las considera filas distintas.
* **Diferencia por ID en mismo nombre:** La `Webcam HD 1080p` se encuentra en ambas sucursales con el mismo nombre y categoría, pero en la Sucursal Norte posee `id_producto = 107` y en la Sur `id_producto = 111`. Esta variación en la clave primaria evita que el operador las identifique como duplicadas.
* **Conclusión:** Para que `UNION` elimine un registro, **todas y cada una de sus columnas deben coincidir de forma idéntica**. Si insertáramos en la Sucursal Sur un registro exacto a uno de la Norte (por ejemplo: `101, 'Laptop Pro 15', 'Computación', 8`), `UNION` devolvería 13 filas al descartar esa repetición absoluta.

---

### 2. ¿Por qué UNION ALL es más eficiente que UNION? ¿Qué operación adicional realiza UNION internamente que consume más recursos?

`UNION ALL` es más eficiente porque realiza una simple concatenación o apilamiento de los bloques de datos a medida que son leídos en memoria o desde el disco.

En cambio, **`UNION` requiere un paso computacional adicional en el plan de ejecución**:
1. **Deduplicación (Sort / Hash Aggregate):** El motor SQL junta los resultados y ejecuta un ordenamiento (*Sort*) o una tabla en memoria (*Hash Match*) para comparar campo por campo y filtrar duplicados.
2. **Consumo de recursos:** En volúmenes masivos de datos (millones de registros), este procesamiento extra exige un alto consumo de CPU y memoria RAM (o escrituras temporales en disco si la RAM se satura), aumentando significativamente el tiempo de respuesta.

---

### 3. ¿En qué casos de negocio usarías cada uno? Dá al menos dos ejemplos reales distintos a los del ejercicio.

#### Casos de uso para `UNION ALL` (Prioridad en volumen total y rendimiento):
1. **Consolidación de transacciones históricas:** Combinar la tabla `ventas_2025` con `ventas_2026` para calcular la facturación anual acumulada. Como cada transacción es válida y única, no se debe perder tiempo en deduplicar.
2. **Extracción masiva en pipelines ETL:** Cargar datos en bruto de múltiples logs web desde distintos servidores hacia un Data Lake o Data Warehouse, donde la deduplicación y limpieza se delegan a etapas posteriores.

#### Casos de uso para `UNION` (Prioridad en catálogos e identificadores únicos):
1. **Padrón unificado de clientes (CRM):** Unir la lista de clientes del sistema de e-commerce con la del sistema POS de sucursales físicas para obtener un listado único de correos electrónicos y lanzar una campaña de marketing sin enviar correos repetidos.
2. **Maestro de proveedores corporativos:** Consolidar los registros de proveedores de distintas filiales regionales para generar una lista única de proveedores homologados a nivel central.

---

### 4. ¿Qué pasa si las columnas de ambas consultas no coinciden en número o tipo? ¿Qué error genera SQL?

Para ejecutar operativamente `UNION` o `UNION ALL`, los conjuntos de datos deben ser compatibles estructuralmente:

1. **Distinto número de columnas:** Si el primer `SELECT` devuelve 4 columnas y el segundo 3, la consulta detiene su ejecución inmediatamente por error de sintaxis.
   * *Error en PostgreSQL:* `ERROR: EACH UNION query must have the same number of columns`.
   * *Error en SQL Server:* `All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.`
2. **Incompatibilidad de tipo de datos:** Si la columna N del primer `SELECT` es de tipo numérico (`INT`) y la posición N del segundo es de tipo texto (`VARCHAR`) y el motor no puede aplicar una conversión implícita, se genera un error de conversión de tipos.
   * *Ejemplo de error:* `Conversion failed when converting the varchar value 'Laptop' to data type int.`
