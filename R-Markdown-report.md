Análisis de datos meteorológicos del Proyecto de Cosecha de Agua (PCA):
Resultados preliminares
================
Johan Ninanya
2023-06-24

# Introducción

Este reporte R Markdown tiene por objetivo documentar la metodologia
para el análisis de datos meteorológicos de las 8 estaciones automáticas
del **Proyecto de Cosecha de Agua (PCA)**. Los datos y
R-scripts/funciones generados pueden ser descargados en el siguiente
enlace <https://github.com/jninanya/PCA>. El control de calidad de datos
se realizó siguiendo la metodología descrita por el Servicio Nacional de
Meteorología e Hidrología (SENAMHI) cuyo manual técnico puede ser
descargado
[aquí](https://www.senamhi.gob.pe/load/file/00711SENA-54.pdf).

<div class="figure" style="text-align: center">

<img src="../Others/6-Estaciones meteorológicas.jpg" alt="Estaciones meteorológicas del Proyecto Cosecha de Agua." width="65%" />
<p class="caption">
Estaciones meteorológicas del Proyecto Cosecha de Agua.
</p>

</div>

# Datos y R-funciones

Para fines de esta demostración los datos consisten de un archivo XLSX
(`"DATOS ESTACIONES-PCA-NICARAGUA.XLSX"`) que contiene datos
meteorlogicos cada 30 minutos de 8 estaciones meteorológicas. Asimismo,
se incluye las siguientes funciones en R:

-   `XLSXweather_to_csv`: Separa el archivo XLSX en archivos
    individuales CSV para cada estación. Asimismo, selecciona las
    variables meteorológicas de interes (temperaturas, humedad relativa,
    precipitación, y radiación).
-   `hourly_summary_wdata`: Resume los datos por hora y verifica si el
    registro esta completo o no (`check_datetime = TRUE`).
-   `estimate_thresholds`: Estima los umbrales inferior (**p01** –
    percentil 1%) y superior (**p99** – percentil 99%) para definir el
    rango de valores a partir del cual los datos pueden ser sospechosos,
    de acuerdo con la metodologia del
    [SENAMHI](https://repositorio.senamhi.gob.pe/handle/20.500.12542/449).  
-   `check_weather_data`: Realiza un control de calidad de los datos
    siguiendo la metodologia del
    [SENAMHI](https://www.senamhi.gob.pe/load/file/00711SENA-54.pdf). De
    momento incluye las pruebas de límites duros y blandos. Se
    implementará las pruebas de consistencia temporal. El resultado es
    un archivo CSV que te indica si hay o no datos “raros” y cuáles son.
-   `fill_wdata`: Completa datos faltantes usando el valor medio
    (`method = "mean"`). Se planea implementar una rutina para aplicar
    algún método de machine leaning para completar datos faltantes (ver
    seccion XX).

Corramos el siguiente chunk para cargar las funciones:

``` r
#getwd()
#[1] "C:/Users/jninanya/OneDrive - CGIAR/Desktop/PCA/R-project"

source("./R-scripts/XLSXweather_to_csv.R")
source("./R-scripts/hourly_summary_wdata.R")
source("./R-scripts/estimate_thresholds.R")
source("./R-scripts/check_weather_data.R")
source("./R-scripts/fill_wdata.R")
source("./R-scripts/QualityControlData.R")
```

Note que estamos trabajando el la carpeta **Desktop/PCA/R-Project**
dentro del cual hay una carpeta (**/R-scripts**) conteniendo todas las
funciones. Asimismo hay otra carpeta (**/Data**) que contiene el archivo
XLSX con los datos de las estaciones meteorológicas. Adicionalmente
vamos a necesitar las siguientes librerias:

``` r
#install.packages(c("readxl", "lubridate", "dplyr", "openair"))

library(readxl)      # lectura de archivos XLSX
library(lubridate)   # manejo de fechas
library(dplyr)       # manejo de data frames
library(openair)     # generación rosa de viento
```

# Análisis meteorológico: Estación Casa Blanca

## Lectura de datos

Importaremos los datos del archivo XLSX usando la funcion
`XLSXweather_to_csv`. Si queremos trabajar con los datos de todas las
estaciones usamos el argumento `sheet = "ALL"`. Para fines práctico,
solo trabajaremos con los datos de la estación **Casa Blanca** por lo
que usaremos el argumento `sheet = "casa blanca"`. Para ello, corramos
el siguiente chunk:

``` r
xlsx_file <- "./Data/DATOS ESTACIONES-PCA-NICARAGUA.xlsx"

#res <- XLSXweather_to_csv(xlsx_file, sheet = "ALL")
res <- XLSXweather_to_csv(xlsx_file, sheet = "casa blanca")

head(res$casa_blanca)
```

    ##         date  time temp tmax tmin rhum wvel wdir prec srad etpo
    ## 1 2022-01-01 00:30 18.2 18.6 18.2   90    0  ---    0    0    0
    ## 2 2022-01-01 01:00 17.9 18.2 17.9   92    0  ---    0    0    0
    ## 3 2022-01-01 01:30 17.7 17.9 17.7   92    0  ---    0    0    0
    ## 4 2022-01-01 02:00 17.3 17.7 17.3   93    0  ---    0    0    0
    ## 5 2022-01-01 02:30 16.8 17.3 16.8   92    0  ---    0    0    0
    ## 6 2022-01-01 03:00 16.7 16.8 16.7   93    0  ---    0    0    0

Las variables que se estan exportando son: temperatura promedio
(`$temp`), temperatura maxima (`$tmax`). temperatura minima (`$tmin`),
humedad relativa (`$rhum`), velocidad del viento (`$wvel`), direccion
del viento (`$wdir`), precipitacion (`$prec`), y radiacion solar
(`$srad`).

## Resumen horario

La funcion `hourly_summary_wdata` nos permite obtener el promedio
horario de los datos, independientemente del intervalo de registro (30
min en este caso). El argumento `check_datetime = TRUE` verifica si hay
o no datos faltantes. Veamos esto con el siguiente chunk:

``` r
weather <- res$casa_blanca
wd <- hourly_summary_wdata(weather, check_datetime = TRUE)

head(wd$hourly)
```

    ##              datetime missing  temp  tmax  tmin rhum wvel wdir prec srad etpo
    ## 1 2022-01-01 00:00:00   FALSE 18.20 18.60 18.20 90.0    0  ---    0    0    0
    ## 2 2022-01-01 01:00:00   FALSE 17.80 18.05 17.80 92.0    0  ---    0    0    0
    ## 3 2022-01-01 02:00:00   FALSE 17.05 17.50 17.05 92.5    0  ---    0    0    0
    ## 4 2022-01-01 03:00:00   FALSE 16.65 16.80 16.65 93.0    0  ---    0    0    0
    ## 5 2022-01-01 04:00:00   FALSE 16.70 16.75 16.55 93.5    0  ---    0    0    0
    ## 6 2022-01-01 05:00:00   FALSE 17.00 17.00 16.85 94.0    0  ---    0    0    0

Note que ahora los datos estan resumidos de manera horaria. En el caso
de `prec` y `etpo` los datos son *acumulados*, mientras que para `wdir`
se toma la *moda* ya que es una variable categorica. Asimismo, note que
aparece una columna llamada `missing` el cual indica la presencia de
datos faltantes. Veamos cuando `missing` es `TRUE`:

``` r
head(wd$hourly[wd$hourly$missing == TRUE, ])
```

    ##                 datetime missing temp tmax tmin rhum wvel wdir prec srad etpo
    ## 6941 2022-02-16 14:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA
    ## 6942 2022-03-04 13:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA
    ## 6943 2022-03-04 14:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA
    ## 6944 2022-03-22 03:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA
    ## 6945 2022-03-22 04:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA
    ## 6946 2022-03-22 05:00:00    TRUE   NA   NA   NA   NA   NA <NA>   NA   NA   NA

Esta funcion tambien retorna como salida la cantidad de valores
faltantes por variable (`missing_wd$n` y `missing_wd$percentage`).
Veamos con el siguiente chunk:

``` r
wd$missing_wd
```

    ##            n percentage
    ## datetime 320   4.407713
    ## temp     326   4.490358
    ## tmax     326   4.490358
    ## tmin     326   4.490358
    ## rhum     326   4.490358
    ## wvel     320   4.407713
    ## wdir     320   4.407713
    ## prec     320   4.407713
    ## srad     326   4.490358
    ## etpo     320   4.407713

Note que del total de datos (N = 7260) aproximadamente el 4.4% (n = 323)
son datos faltantes. Acontinuacion se determina limites superior e
inferior para las variables meteorlogicas definidas en el argumento
`wd_var`.

``` r
weather <- wd$hourly
loc <- names(res)[1]
wd_var = c("temp", "tmax", "tmin", "rhum", "wvel", "srad")

par(mfrow = c(2, 3))
tsl = estimate_thresholds(weather, loc, graph = TRUE, wd_var)
```

![](R-Markdown-report_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->
