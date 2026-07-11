# “Precio, Uso y Oferta: Análisis del Mercado de Autos Usados”

## Contexto y Origen de los Datos

- **Objetivo:** Practicar SQL con un caso real intentando responder preguntas básicas sobre autos usados en Ecuador: precios promedio, rankings de los más baratos, cómo cae el precio según el kilometraje, y qué tan disponible está cada rango de kilometraje en el mercado actual.
- **Fuente:** Usé un dataset público de Kaggle (el autor indica que los datos son extraídos de PatioTuerca).
- **Aclaración y Mentoría:** Este proyecto lo desarrollé de la mano de la IA como una guía, no como alguien que lo resolviera por mí. Mi meta no fue solo tirar código, sino aprender a interpretar los resultados y saber qué herramientas de SQL usar en un escenario real sin instrucciones guiadas.

## Bloque 0: Diagnóstico Inicial (¿A qué me enfrentaba?)

### ¿Qué se buscaba hacer?

Hacer una primera inspección visual a la tabla `Registro_PatioTuerca` para entender la estructura de las columnas, los tipos de datos y detectar anomalías antes de escribir cualquier consulta compleja.

**Enfoque por Modelos Top** En lugar de filtrar por marcas enteras voy a trabajar **únicamente con los modelos más vendidos**, para esto hacemos un filtro primario para ver cual eran esos modelos que encabezaban la lista ordenando de mayor a menor con un limite de solo 5 modelos, los mas top.

```sql
SELECT marca, modelo, count(marca) as cantidad_autos
from Registro_PatioTuerca
group by marca, modelo
ORDER BY cantidad_autos desc
LIMIT 5 ;
```
<img width="647" height="312" alt="image" src="https://github.com/user-attachments/assets/fe147a4f-bff7-49e1-8f7d-1a6ad0256d8e" />

Los modelos más vendidos (Aveo Family, Explorer XLT, Spark GT, Fortuner 2.7, Grand Vitara SZ).

## Bloque 1 — Limpieza (Enfoque de Modelos Top)

Al tener claro el panorama en el terminal, decidí que están serian las directrices conceptuales a ejecutar en este bloque:

- Filtrado Estratégico en la columna “Modelo” (Cláusula WHERE):

  Como ya tengo el top de modelos, a partir de ahora solo esos serian los modelos con los voy a trabajar.

- Tratamiento de Nulos y Vacíos en la columna “Precio” (Clausula CASE WHEN):

  Debo conocer si me enfrento al menos en esos modelos a nulos, o vacíos en esos modelos.

```sql
SELECT 
    COUNT(*) AS total_filas,
    SUM(CASE WHEN Precio = '' THEN 1 ELSE 0 END) AS vacios_Precio,
    SUM(CASE WHEN Precio IS NULL THEN 1 ELSE 0 END) AS nulls_Precio
FROM Registro_PatioTuerca
WHERE Modelo IN ('Aveo Family', 'Explorer XLT', 'Spark GT', 'Fortuner 2.7', 'Grand Vitara SZ');
```
<img width="650" height="252" alt="image 2" src="https://github.com/user-attachments/assets/4e6e1c2a-b775-4abf-af44-d3f55d851ed1" />

**Resultados de la Auditoría**

- **Total de registros para los 5 modelos:** 606 filas.
- **Precios vacíos (`''`):** 5 filas.
- **Precios nulos (`NULL`):** 0 filas.
- **Muestra limpia utilizable:** 601 filas.

**Criterio sobre alterar datos:** Al detectar las 5 filas vacías, evalué si debía rellenar esos valores, pero la decisión correcta fue no modificar la información original para no romper la integridad de la base, el tocar base de datos o valores sin una autorización lo único que lograría es alterar datos para el informe final.

```sql
WHERE Precio IS NOT NULL AND Precio != ''
```

A partir de aquí una de mis clausulas principales además de los modelos… es la adjuntada, de esa manera le pedimos a SQL que no tome en consideración las filas que tengan valores nulos en la columna “Precio”.

## Bloque 2: Métricas Generales (Agregaciones)

### ¿Qué se buscaba hacer?

Obtener un resumen ejecutivo de los 5 modelos: calcular el kilometraje promedio, precio mínimo, precio máximo y precio promedio sobre el universo de las 601 filas válidas.

```sql
SELECT Modelo, 
    ROUND(AVG(Kilómetraje), 2) AS Promedio_KM,
    MAX(Precio) AS Precio_Max, 
    MIN(Precio) AS Precio_Min, 
    ROUND(AVG(Precio), 2) AS Precio_Prom
FROM Registro_PatioTuerca
WHERE Precio IS NOT NULL 
  AND Precio != ''
  AND Modelo IN ('Aveo Family', 'Explorer XLT', 'Spark GT', 'Fortuner 2.7', 'Grand Vitara SZ')
GROUP BY Modelo
ORDER BY Promedio_KM DESC;
```
<img width="938" height="300" alt="image 3" src="https://github.com/user-attachments/assets/d8a9c723-034c-4070-ae96-0be56a0c20bd" />

### Observaciones y Hallazgos de Negocio

**Detección de Outliers (Anomalías en Datos):**

1. El *Aveo Family* arrojó un `Precio_Max` de `104.99` ($104,990 USD). Al consultar ese registro en detalle, era un auto con kilometraje normal pero con un precio absurdo para la gama.
2. **Aprendizaje analítico:** Identifiqué que fue un error de digitación al publicar la oferta. Este tipo de valores distorsiona el promedio general del modelo.


<img width="1040" height="473" alt="001" src="https://github.com/user-attachments/assets/4e87002a-3dae-42dc-87a7-5f6fa42f8120" />



**Lectura de Mercado:**

1. El **Spark GT** es el modelo con menor desgaste promedio (63,340 km) y mantiene un precio promedio de $12,470 USD, muy cercano al *Aveo Family* ($12,240 USD) a pesar de ser un segmento más pequeño, debido a su menor uso.
2. El **Explorer XLT** y la **Fortuner 2.7** lideran el rango de precios altos (~$32,200 USD promedio), pero el Explorer presenta un kilometraje promedio superior (143k km frente a 123k km).

## Bloque 3: Rankings por Modelo (Funciones de Ventana)

### ¿Qué se buscaba hacer?

Generar un ranking interno de precios para cada uno de los 5 modelos de interés. La idea era ordenar las ofertas de menor a mayor precio dentro de cada vehículo para identificar rápidamente las opciones más económicas de cada gama.

```sql
SELECT Modelo, Precio, Kilómetraje, Año,
    DENSE_RANK() OVER (PARTITION BY Modelo ORDER BY Precio ASC) AS Ranking_Precio
FROM Registro_PatioTuerca
WHERE Precio IS NOT NULL 
  AND Precio != ''
  AND Modelo IN ('Aveo Family', 'Explorer XLT', 'Spark GT', 'Fortuner 2.7', 'Grand Vitara SZ');
```

<img width="803" height="455" alt="image 5" src="https://github.com/user-attachments/assets/c40b0db0-1edb-4851-9460-482c25b1b8fa" />

**Diferencia entre `PARTITION BY` y `GROUP BY`:**

- Entendí que `PARTITION BY Modelo` divide los datos en "cajas" independientes (una por cada modelo) para reiniciar el conteo del ranking (1, 2, 3...) dentro de cada grupo, manteniendo intactas las 601 filas del detalle.

**Uso de `DENSE_RANK()` vs `RANK()`:**

- Elegí `DENSE_RANK()` para evitar saltos en la numeración del ranking en caso de que dos o más publicaciones tuvieran exactamente el mismo precio.

## Bloque 4: Segmentación por Uso y Análisis de Depreciación

### ¿Qué se buscaba hacer?

Clasificar los vehículos en tres rangos de uso según su kilometraje (**Poco uso**, **Uso moderado**, **Uso elevado**) para calcular el precio promedio y el total de publicaciones dentro de cada categoría, manteniendo la comparación entre los 5 modelos de interés, y agruparlos para darme un promedio de los autos rankeados.

```sql
SELECT 
 CASE 
    WHEN Kilómetraje < 50000 THEN '1.Poco Uso <50mil'
    WHEN Kilómetraje BETWEEN 50000 AND 150000 THEN '2.Uso Moderado 50-150mil'
    ELSE '3.Uso Elevado >150mil' END AS Vida_util,   
ROUND(AVG(Precio),2) AS Precio_promedio, 
COUNT(*) AS Total_Autos
FROM Registro_PatioTuerca
	WHERE Precio IS NOT NULL 
 		AND Precio != '' 
  		AND Modelo IN ('Aveo Family', 'Explorer XLT', 'Spark GT', 'Fortuner 2.7', 'Grand Vitara SZ')
GROUP BY Vida_util;
```

<img width="688" height="251" alt="image 6" src="https://github.com/user-attachments/assets/3ed18f4f-4017-44af-a68f-741ca7b78a40" />


### Tropiezos y Lecciones con la Sintaxis

- **El tropiezo inicial:** En las primeras pruebas intenté escribir múltiples instrucciones `CASE` independientes para cada condición. Esto me generaba columnas separadas e innecesarias en la terminal llenas de espacios vacíos o valores nulos.
- **La solución:** Aprendí a unificar toda la regla lógica en un solo bloque `CASE WHEN ... THEN ... ELSE ... END`. De esta forma, cada vehículo entra en una sola categoría y el resultado devuelve una columna limpia llamada ‘Vida_util’.

<img width="970" height="268" alt="image" src="https://github.com/user-attachments/assets/e914ff3b-e53e-443f-a537-f0d0dbef373c" />

<img width="972" height="588" alt="image" src="https://github.com/user-attachments/assets/f38adefe-21f5-4d8f-bdcc-0774ba457976" />


### Errores Encontrados y Cosas que Aprendí en el Camino

En esta sección documento los tropiezos reales que tuve durante el proceso y cómo los resolví:

#### Error 1: Confusión entre `WHERE` y `GROUP BY`

- **El tropiezo:** Intenté filtrar modelos específicos cuando ya tenía un `WHERE` puesto y pensaba que no podía agregar más filtros.
- **La lección:** Aprendí que con la palabra `AND` y la cláusula `IN (...)` puedo filtrar varios modelos a la vez dentro del mismo `WHERE`.

#### Error 2: El caso del Aveo de $104

- **El tropiezo:** Al sacar los precios máximos por modelo, vi un *Chevrolet Aveo Family* listado en **104.99**. Me di cuenta de que un auto así no cuesta $104,000 ni tampoco $104 dólares.
- **La lección:** Descubrí que los datos venían en miles de dólares (ej. 10.6 = $10,600) y que ese 104.99 era un error humano de quien subió el anuncio. Entendí que mi rol no es adivinar ni editar el dato a mano, sino filtrarlo o reportarlo para no inflar los promedios.

#### Error 3: Poner `GROUP BY` donde no iba (Funciones de Ventana)

- **El tropiezo:** Para crear un ranking con `DENSE_RANK()`, le puse un `GROUP BY` al final a todas las columnas porque tenía la idea de que "todo lo que no se agrega va al group by".
- **La lección:** Mi mentor me aclaró que las funciones de ventana (`OVER`) trabajan sobre filas individuales y no colapsan la tabla, por lo que usar el `GROUP BY` es un tema de lentitud, es un error de lógica/sintaxis.

#### Error 4: Múltiples `CASE WHEN` generando columnas separadas

- **El tropiezo:** Para clasificar el uso del auto (Poco uso, Uso moderado, Uso elevado), escribí tres instrucciones `CASE` independientes. Esto me generó tres columnas distintas en la terminal llenas de espacios vacíos.
- **La lección:** Aprendí a unificar las condiciones en un solo bloque `CASE WHEN ... THEN ... ELSE ... END` para que me devuelva todo en una sola columna limpia llamada `Vida_util`.


