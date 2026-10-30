# ConnectaTel — Análisis de comportamiento y segmentación de clientes

## Descripción del proyecto

Este proyecto analiza el comportamiento de los clientes de **ConnectaTel**, una empresa de telecomunicaciones que opera en Latinoamérica. El trabajo utiliza información de clientes, planes y registros de uso para explorar la calidad de los datos, resumir el consumo, identificar posibles valores atípicos y construir segmentos de clientes.

El periodo de referencia indicado para el análisis llega hasta el **31 de diciembre de 2024**. Durante la exploración se detectaron algunos registros con fechas posteriores a ese límite, que fueron tratados como valores faltantes para evitar incorporarlos como fechas válidas del periodo.

El análisis se desarrolló progresivamente en Python y pandas, priorizando código comprensible, decisiones de limpieza justificadas y resultados que puedan reproducirse.

## Objetivos

- Explorar la estructura y calidad de los archivos disponibles.
- Identificar valores faltantes, sentinels, fechas fuera del periodo y posibles inconsistencias.
- Limpiar los valores inválidos confirmados sin eliminar información válida innecesariamente.
- Construir métricas de uso por cliente.
- Describir el perfil de los clientes y la distribución de los planes.
- Visualizar las distribuciones de edad y consumo por plan.
- Detectar posibles outliers y decidir si deben conservarse o investigarse.
- Segmentar a los clientes por edad y nivel de uso.
- Traducir los resultados en oportunidades y recomendaciones para el negocio.

## Datos utilizados

El proyecto trabaja con tres archivos CSV:

- **`plans.csv`**: catálogo de planes y características como precio, minutos incluidos, GB incluidos y costos por uso adicional. De acuerdo con las instrucciones del proyecto, este archivo contiene dos registros y no requería exploración adicional de valores nulos.
- **`users_latam.csv`**: información de los clientes, incluidos identificador, nombre, edad, ciudad, fecha de registro, plan y fecha de abandono (`churn_date`).
- **`usage.csv`**: registros de utilización de los servicios, con identificador de registro, usuario, tipo de uso (`call` o `text`), fecha, duración y longitud.

En el entorno original, los archivos se cargaron desde `/datasets/`:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

plans = pd.read_csv("/datasets/plans.csv")
users = pd.read_csv("/datasets/users_latam.csv")
usage = pd.read_csv("/datasets/usage.csv")
```

Al ejecutar el proyecto fuera de ese entorno, se deben adaptar las rutas a la ubicación local de los archivos. Los datos originales no se incluyen en este README.

## Herramientas utilizadas

- **Python** para el desarrollo del análisis.
- **pandas** para carga, exploración, limpieza, agrupación y combinación de datos.
- **NumPy** para condiciones y segmentación de clientes.
- **Matplotlib** y **Seaborn** para histogramas y boxplots.
- **Jupyter Notebook** para organizar y ejecutar el análisis por etapas.

Instalación básica de las bibliotecas:

```bash
pip install pandas numpy matplotlib seaborn jupyter
```

## Metodología y etapas desarrolladas

### 1. Carga y exploración inicial

Se cargaron los tres archivos en los DataFrames `plans`, `users` y `usage`. Se revisaron las primeras filas, las dimensiones mediante `.shape` y la estructura mediante `.info()`.

La exploración permitió identificar las columnas, los tipos de datos y la cantidad de valores no nulos antes de modificar los archivos.

### 2. Calidad de los datos

#### Valores faltantes

En la revisión inicial se encontraron los siguientes valores faltantes:

- En `users.city`: **469 valores**, equivalentes al **11,725 %**.
- En `users.churn_date`: **3.534 valores**, equivalentes al **88,35 %**.
- En `usage.date`: **50 valores**, equivalentes al **0,125 %**.
- En `usage.duration`: **22.076 valores**, equivalentes al **55,19 %**.
- En `usage.length`: **17.896 valores**, equivalentes al **44,74 %**.

Las demás columnas revisadas de `users` y `usage` no presentaron valores nulos en esa inspección. El archivo `plans` no presentó valores ausentes según la revisión indicada en el proyecto.

#### Valores faltantes de `duration` y `length` según el tipo de uso

Se comparó el porcentaje de valores ausentes según `usage.type`:

- `duration`: **0 %** de nulos en `call` y aproximadamente **99,93 %** en `text`.
- `length`: aproximadamente **99,93 %** de nulos en `call` y **0 %** en `text`.

El patrón muestra que la ausencia está estrechamente relacionada con el tipo de registro. Por el significado de las variables, estos valores se interpretaron como **ausencias estructurales o datos no aplicables**: un mensaje normalmente no tiene duración de llamada y una llamada normalmente no tiene longitud de mensaje. Por eso se conservaron como nulos y no se imputaron artificialmente.

Esta observación es compatible con una ausencia explicada por una variable observada, pero conviene no etiquetarla automáticamente como MAR en sentido estadístico estricto: el patrón parece corresponder principalmente a variables que no aplican a ciertos tipos de uso.

#### Resumen numérico y categorías

En `users` se identificó el valor `-999` en `age`, que no representa una edad válida y se trató como sentinel. En `city` se observaron ocho valores distintos, incluidos errores de escritura y el símbolo `?`. En `plan` se encontraron las categorías Básico y Premium. En `usage.type` se encontraron `call` y `text`.

También se detectaron valores máximos que merecían revisión en las variables de uso: `duration` alcanzó **120** y `length` alcanzó **1.490**. Estos valores se identificaron como sospechosos para inspección, pero no se declaró que fueran errores confirmados ni se eliminaron automáticamente.

### 3. Estandarización y limpieza

Se aplicaron las siguientes reglas:

- En `users.age`, se reemplazó el sentinel `-999` por la mediana calculada excluyendo ese sentinel.
- En `users.city`, se reemplazó el símbolo `?` por `pd.NA`, porque no se conoce la ciudad correcta.
- Las columnas `users.reg_date` y `usage.date` se convirtieron a formato fecha con `pd.to_datetime(..., errors="coerce")`.
- Las fechas posteriores al **31 de diciembre de 2024** se marcaron como valores faltantes (`NaT`).
- No se eliminaron filas por estas correcciones.
- Los nulos de `duration` y `length` se conservaron porque su patrón está asociado al tipo de uso.
- Los outliers identificados en las métricas agregadas por usuario se conservaron, porque no existía evidencia suficiente para considerarlos errores.

En la inspección de fechas se encontraron **40 registros de `users.reg_date` correspondientes a 2026** (1 % de los 4.000 usuarios). Se marcaron como fechas fuera del periodo. En `usage.date`, los **39.950 valores con fecha disponible** correspondían a 2024; los otros 50 registros tenían la fecha ausente.

La columna `users.churn_date` se revisó en cuanto a sus valores nulos, pero no se incluyó en la conversión y estandarización de fechas de esta etapa, cuyo alcance definido fue `users.reg_date` y `usage.date`. Sus valores ausentes se conservaron pendientes de confirmar su significado.

### 4. Agregación del uso por usuario

Se crearon columnas auxiliares para identificar los tipos de uso:

- `is_text`: vale 1 cuando el registro es un mensaje y 0 en caso contrario.
- `is_call`: vale 1 cuando el registro es una llamada y 0 en caso contrario.

Después se agrupó `usage` por `user_id` y se calcularon tres métricas:

- `cant_mensajes`: total de mensajes por usuario.
- `cant_llamadas`: total de llamadas por usuario.
- `cant_minutos_llamada`: suma de `duration` de los registros de tipo `call` por usuario.

La tabla agregada se combinó con `users` mediante `user_id`, conservando todos los clientes con una unión izquierda (`how="left"`). Para los usuarios sin registros de una métrica se asignó cero en las columnas de conteo y minutos del resumen.

El DataFrame resultante se llamó **`user_profile`**.

**Nota sobre el periodo:** la agregación descrita se realizó sobre los registros disponibles de `usage`; no se documentó un filtro explícito que excluyera los 50 registros sin fecha. Por ello, los indicadores de uso deben interpretarse como métricas del archivo analizado y no afirmarse como métricas estrictamente limitadas a 2024 hasta validar ese punto.

### 5. Estadística descriptiva por usuario

El resumen de `user_profile` produjo los siguientes resultados:

- **Clientes analizados:** 4.000.
- **Edad (`age`):** media de **48,14** años; mediana de **48**; mínimo de **18** y máximo de **79**.
- **Mensajes (`cant_mensajes`):** media de **5,523**; mediana de **5**; mínimo de **0** y máximo de **17**.
- **Llamadas (`cant_llamadas`):** media de **4,477**; mediana de **4**; mínimo de **0** y máximo de **15**.
- **Minutos (`cant_minutos_llamada`):** media de **23,31**; mediana de **19,78**; mínimo de **0** y máximo de **155,69**.

La diferencia entre la media y la mediana de minutos, junto con el máximo observado, indicó una dispersión mayor y justificó revisar posibles consumos extremos.

La distribución de planes fue:

- **Básico:** 64,88 % de los clientes.
- **Premium:** 35,12 % de los clientes.

### 6. Visualización de distribuciones

Se generaron histogramas de `age`, `cant_mensajes`, `cant_llamadas` y `cant_minutos_llamada`, diferenciando los planes mediante `hue="plan"` y la paleta `skyblue` y `green`.

Los principales hallazgos visuales fueron:

- **Edad:** los perfiles de edad de Básico y Premium mostraron distribuciones visualmente similares, sin una diferencia marcada entre los planes.
- **Mensajes:** la mayor concentración se observó aproximadamente entre 3 y 8 mensajes. La distribución presentó sesgo a la derecha, con pocos clientes que alcanzaron valores de hasta 17.
- **Llamadas:** la mayor concentración se observó aproximadamente entre 3 y 6 llamadas. La distribución mostró sesgo a la derecha y una cola hacia cantidades mayores.
- **Minutos:** la mayor concentración se ubicó entre 0 y 40 minutos, especialmente alrededor de 10 a 20. Se observó un sesgo pronunciado a la derecha y una cola con consumos de hasta 155,69 minutos. El gráfico sugirió algunos consumos elevados en Premium, aunque esa diferencia visual no demuestra por sí sola una diferencia estadística entre planes.

Para comparar las formas de las distribuciones por plan se utilizó `stat="probability"` y `common_norm=False`, por lo que las barras representan proporciones dentro de cada plan, no cantidades absolutas de clientes.

### 7. Identificación de outliers mediante IQR

Se utilizaron boxplots y el método del rango intercuartílico (IQR) para las métricas agregadas por usuario. Se identificaron posibles outliers superiores:

- **`cant_mensajes`:** 46 clientes por encima del límite superior de **11,5**; máximo observado: **17**. Corresponde al **1,15 %** de los 4.000 clientes.
- **`cant_llamadas`:** 30 clientes por encima del límite superior de **10,5**; máximo observado: **15**. Corresponde al **0,75 %**.
- **`cant_minutos_llamada`:** 109 clientes por encima del límite superior de aproximadamente **61,87**; máximo observado: **155,69**. Corresponde al **2,73 %**.

La mayor cantidad de posibles outliers se encontró en minutos de llamadas. Se decidió **conservar estos registros** porque un consumo alto puede ser un comportamiento legítimo y no se contaba con evidencia para considerarlo un error. Se recomienda revisarlos como perfiles de alto consumo, no eliminarlos automáticamente.

### 8. Segmentación de clientes

#### Segmentación por nivel de uso

Se creó `grupo_uso` utilizando la cantidad de llamadas y mensajes:

- **Bajo uso:** llamadas < 5 y mensajes < 5.
- **Uso medio:** no cumple la condición de bajo uso y, además, llamadas < 10 y mensajes < 10.
- **Alto uso:** todos los demás casos; es decir, cuando al menos una de las dos métricas es 10 o más.

Distribución exacta:

- **Uso medio:** 2.943 clientes (**73,58 %**).
- **Bajo uso:** 779 clientes (**19,48 %**).
- **Alto uso:** 278 clientes (**6,95 %**).

El grupo de Uso medio concentra la mayor parte de la base analizada. El grupo de Alto uso es pequeño y puede ser relevante para estudiar necesidades de capacidad o beneficios adicionales. El grupo de Bajo uso puede ser objeto de investigación y campañas de adopción o fidelización, sin asumir que necesariamente está en riesgo de abandono.

#### Segmentación por edad

Se creó `grupo_edad` con las siguientes reglas:

- **Joven:** edad < 30 años.
- **Adulto:** edad >= 30 y < 60 años.
- **Adulto Mayor:** edad >= 60 años.

Distribución exacta:

- **Adulto:** 2.018 clientes (**50,45 %**).
- **Adulto Mayor:** 1.222 clientes (**30,55 %**).
- **Joven:** 760 clientes (**19,00 %**).

El grupo Adulto representa la mayor parte de los clientes, seguido por Adulto Mayor y Joven.

Las dos segmentaciones son distribuciones independientes. Los resultados disponibles no permiten afirmar cuántos clientes pertenecen simultáneamente a una combinación específica de edad, uso y plan; para responderlo sería necesario cruzar esas variables.

## Principales conclusiones para el negocio

1. La base analizada está compuesta principalmente por clientes del plan Básico (**64,88 %**) y por clientes del grupo de edad Adulto (**50,45 %**).
2. El nivel de uso predominante es Uso medio (**73,58 %**), mientras que Alto uso representa **6,95 %** y Bajo uso **19,48 %**.
3. Los perfiles de Básico y Premium se ven similares en las distribuciones de edad, mensajes y llamadas. En minutos se observa una cola de consumos altos, que conviene analizar con mayor detalle.
4. Los posibles outliers de consumo se conservaron: pueden representar clientes reales con necesidades particulares.
5. Los hallazgos permiten orientar análisis posteriores de adecuación de planes, consumo adicional y retención, pero no prueban por sí solos diferencias de rentabilidad ni riesgo de abandono.

## Recomendaciones

- **Revisar la adecuación del plan Básico**, dado que concentra el 64,88 % de los clientes. Conviene comprobar si sus beneficios corresponden al consumo observado y si existen oportunidades de mejora sin afectar la rentabilidad.
- **Evaluar el consumo del grupo de alto uso**, especialmente los minutos de llamadas. Se pueden estudiar paquetes adicionales o alternativas de mayor capacidad, comprobando primero los costos y márgenes.
- **Investigar el grupo de bajo uso** para entender si se trata de clientes que utilizan poco el servicio por preferencia, por necesidades distintas o por posible desconexión. No debe asumirse riesgo de churn sin analizarlo.
- **Adaptar la comunicación por edad**, teniendo en cuenta la participación de Adultos y Adultos Mayores, y sin descuidar el segmento Joven.
- **Cruzar edad, nivel de uso y plan** para identificar perfiles combinados y diseñar acciones más específicas.
- **Analizar churn por segmento** una vez validado el significado de `churn_date`, para medir tasas de abandono y no inferirlas únicamente a partir del nivel de uso.
- **Comparar consumo con las condiciones de cada plan** (minutos, mensajes si aplica, GB, precio y tarifas por excedentes) antes de recomendar cambios concretos de producto.
- **Mantener controles de calidad** para edades, ciudades, fechas e identificadores en futuras cargas de datos.

## Limitaciones y verificaciones pendientes

- No se ha demostrado que los clientes de alto uso sean los más rentables: faltan ingresos, costos y márgenes por cliente.
- No se ha calculado una tasa de churn por grupo de edad, uso o plan. Los nulos en `churn_date` requieren confirmar su significado antes de utilizarlos como clientes activos.
- Los 50 registros de `usage` sin fecha deben tratarse de forma explícita si se necesita un análisis estrictamente limitado a 2024. La agregación por usuario descrita no documentó su exclusión.
- Las variantes de escritura de `city` fueron identificadas y los signos `?` se reemplazaron por nulos; no se dispone aquí de una validación completa de todas las ciudades normalizadas.
- Los máximos de `duration` (120) y `length` (1.490) requieren validación de unidad y contexto. No se confirmó que sean errores.
- La comparación entre planes en los histogramas es visual y descriptiva; no se realizó una prueba estadística para determinar diferencias significativas.
- Los umbrales de segmentación por edad y uso son reglas definidas para el ejercicio, no segmentos validados mediante un modelo estadístico.

## Estructura sugerida del repositorio

```text
ConnectaTel/
├── README.md
├── notebooks/
│   └── connectatel_analysis.ipynb
├── data/
│   ├── plans.csv
│   ├── users_latam.csv
│   └── usage.csv
├── outputs/
│   └── figures/
└── requirements.txt
```

La estructura es una sugerencia; se puede adaptar a los nombres y carpetas reales del repositorio. Si los archivos contienen información privada o no se cuenta con autorización para publicarlos, no se deben subir a GitHub. En ese caso, se recomienda excluir `data/` mediante `.gitignore` y explicar cómo obtener los archivos por un canal autorizado.

## Cómo ejecutar el proyecto

1. Clonar o descargar el repositorio.
2. Instalar las bibliotecas indicadas en la sección de herramientas.
3. Colocar los archivos CSV en la carpeta de datos autorizada o configurar sus rutas locales.
4. Abrir el notebook en Jupyter.
5. Ejecutar las celdas desde el inicio y en orden para reconstruir los DataFrames, la limpieza, las métricas, los gráficos y las segmentaciones.

**Importante:** el notebook debe ejecutarse desde un Kernel limpio para comprobar que no depende de variables creadas en ejecuciones anteriores. Las rutas `/datasets/...` utilizadas en el entorno original deben adaptarse al entorno local.

## Estado del proyecto

Se completaron las etapas de exploración inicial, revisión de calidad, limpieza básica descrita, agregación por usuario, estadística descriptiva, visualización de distribuciones, identificación de outliers, segmentación por edad y uso, y redacción de insights ejecutivos preliminares.

Antes de considerar cerradas las conclusiones comerciales, se recomienda realizar las verificaciones pendientes indicadas arriba, especialmente el análisis del periodo exacto de uso, la relación entre segmentos y churn, y la comparación del consumo con las condiciones y costos de los planes.
