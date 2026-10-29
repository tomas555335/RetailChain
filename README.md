# Consolidador de Inventario — RetailChain (UNION vs. UNION ALL)

Este proyecto analiza las diferencias fundamentales entre el uso de los operadores `UNION` y `UNION ALL` en SQL para la consolidación de datos provenientes de múltiples sucursales de la cadena **RetailChain**.

---

## Preguntas Técnicas y Análisis de Resultados

### 1. ¿Cuántas filas devuelve cada consulta y por qué son distintas? Explicá con ejemplos concretos de los datos qué filas se eliminaron con UNION.

* **`UNION ALL` devuelve 14 filas:** La Sucursal Norte aporta 7 registros y la Sucursal Sur aporta 7 registros. Al no aplicar ningún filtro ni deduplicación, combina directamente la totalidad de las filas ($7 + 7 = 14$).
* **`UNION` devuelve 13 filas:** Elimina únicamente las filas que son **100% idénticas** en todas las columnas seleccionadas (`id_producto`, `nombre_producto`, `categoria` y `stock`).

#### Caso concreto de eliminación en los datos:
* **Producto 106 (`SSD Externo 1TB`):** En la Sucursal Norte tiene `stock = 10` y en la Sur tiene `stock = 7`. Como el valor de la columna `stock` difiere, las filas **no son idénticas**, por lo que `UNION` conserva ambas filas en el resultado.
* **Productos 103 (`Monitor 4K 27"`) y 104 (`Teclado Mecánico`):** Tienen stocks diferentes en cada sucursal (5 vs 3 y 20 vs 18), por lo cual tampoco se eliminan.
* **Producto 107 / 111 (`Webcam HD 1080p`):** Tienen el mismo nombre y categoría, pero sus `id_producto` son distintos (107 en Norte, 111 en Sur), lo que hace que las filas sean técnicamente diferentes para el motor.
* **Producto idéntico eliminado:** Si existiera un registro donde `id_producto`, `nombre_producto`, `categoria` y `stock` coincidieran exactamente en ambas tablas (por ejemplo, dos filas con `106, 'SSD Externo 1TB', 'Almacenamiento', 10`), `UNION` habría descartado la repetición. La diferencia de 1 fila entre ambas consultas ocurre porque `UNION` identifica y elimina únicamente los registros con duplicación absoluta en sus campos.

---

### 2. ¿Por qué UNION ALL es más eficiente que UNION? ¿Qué operación adicional realiza UNION internamente que consume más recursos?

`UNION ALL` simplemente concatena o apila los conjuntos de resultados directamente a medida que los lee de memoria o disco.

En cambio, **`UNION` es computacionalmente más costoso** porque realiza un paso adicional en el plan de ejecución:
1. **Ordenamiento / Hash Match (Deduplicación):** El motor debe almacenar en buffer las filas de ambas consultas y ejecutar un algoritmo de ordenamiento (*Sort*) o una tabla Hash (*Hash Match - Aggregate*) para comparar campo por campo y descartar los duplicados.
2. **Uso de memoria y CPU:** Con volúmenes masivos de datos (millones de filas), este proceso consume recursos significativos de RAM y CPU, e incluso puede forzar escrituras temporales en disco (como `tempdb` en SQL Server), degradando el rendimiento.

---

### 3. ¿En qué casos de negocio usarías cada uno? Dá al menos dos ejemplos reales distintos a los del ejercicio.

#### Casos de uso para `UNION ALL` (Preservar volumen y priorizar rendimiento):
1. **Consolidación de logs o transacciones históricas:** Al unir la tabla de `ventas_2025` con `ventas_2026` para calcular la facturación anual global. Cada transacción es única y válida; no se busca eliminar filas ni perder tiempo procesando deduplicaciones.
2. **Carga masiva de datos a un Data Lake o Data Warehouse:** En procesos ETL donde la deduplicación o limpieza se posterga para etapas posteriores y se requiere extraer y apilar datos crudos lo más rápido posible.

#### Casos de uso para `UNION` (Catálogos limpios e identificadores únicos):
1. **Directorio unificado de contactos/clientes:** Combinar listas de clientes procedentes de dos sistemas distintos (ej. CRM de ventas web y sistema POS de tiendas físicas) para generar un padrón único de mailing sin enviar correos duplicados a la misma persona.
2. **Maestro de código de proveedores o materias primas:** Consolidar proveedores de múltiples filiales para obtener la lista única de empresas habilitadas a nivel corporativo.

---

### 4. ¿Qué pasa si las columnas de ambas consultas no coinciden en número o tipo? ¿Qué error genera SQL?

Para que `UNION` o `UNION ALL` funcionen, las sentencias `SELECT` deben cumplir dos reglas estrictas de estructura:

1. **Distinto número de columnas:** Si el primer `SELECT` solicita 4 columnas y el segundo solicita 3, el motor de base de datos arroja un error de sintaxis antes de ejecutar la consulta. 
   * *Ejemplo de error (PostgreSQL):* `ERROR: EACH UNION query must have the same number of columns`.
   * *Ejemplo de error (SQL Server):* `All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.`
2. **Incompatibilidad de tipos de datos:** Si la columna ordinal N de la primera consulta es de tipo numérico (`INT`) y la columna ordinal N de la segunda consulta es de tipo texto (`VARCHAR`) sin conversión implícita, el motor detendrá la ejecución con un error de conversión.
   * *Ejemplo de error:* `Conversion failed when converting the varchar value 'Laptop' to data type int.`
