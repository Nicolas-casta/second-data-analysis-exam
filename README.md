# informe ejecutivo
## analisis de yellow taxi trips — nyc tlc, enero 2026

**estudiante:** nicolas castaño diosa
**fecha:** 08/06/2026

---

## 1. resumen ejecutivo

se procesaron 3,724,889 registros de yellow taxi trips de nueva york de enero 2026 usando dask para procesamiento distribuido. tras una auditoria de calidad y limpieza, se conservo el 65.05% de los datos. se entreno un random forest que alcanzo accuracy de 0.8836 en el conjunto de prueba, superando el umbral requerido. el modelo generalizo correctamente a febrero 2026 con una caida de solo 0.43 puntos porcentuales.

---

## 2. hallazgos de la auditoria

se encontraron anomalias en varias categorias:

- **temporal:** 45,070 viajes con dropoff anterior al pickup y 7 registros fuera de enero 2026
- **duracion:** 83,747 viajes de menos de 1 minuto y 1,543 de mas de 240 minutos
- **distancia:** 125,738 viajes con trip_distance cero y 162 con mas de 100 millas
- **pasajeros:** 1,088,058 registros con passenger_count nulo
- **monetarios:** 39,463 registros con fare_amount negativo
- **categoricas:** 110,864 registros con RatecodeID igual a 99

las columnas passenger_count, RatecodeID, store_and_fwd_flag, congestion_surcharge y Airport_fee comparten exactamente 1,088,058 nulos en las mismas filas, lo que sugiere un proveedor cuyo sistema no captura esos campos.

### estabilidad temporal

se identificaron dos periodos anomalos en las series diarias de duracion, velocidad y volumen:

- **1 de enero:** volumen casi en cero por el feriado de año nuevo
- **25 y 26 de enero:** el volumen cayo a menos de 50,000 trips diarios por una tormenta de nieve documentada por el servicio meteorologico nacional: https://www.weather.gov/okx/20260125_26

---

## 3. limpieza y transformacion

se aplicaron 6 filtros en dask que eliminaron registros con fechas invalidas, duraciones imposibles, trip_distance fuera de rango, passenger_count nulo o invalido y RatecodeID desconocido. el conjunto limpio conservo 2,423,159 filas (65.05%).

para PULocationID y DOLocationID se opto por tratarlas como variables numericas ordinales. se descarto one-hot encoding porque generaria mas de 500 columnas nuevas. se aplico escalado robusto a la regresion logistica por ser la tecnica mas adecuada ante outliers severos.

---

## 4. comparacion de modelos

| modelo | accuracy | f1 macro | roc-auc |
|---|---|---|---|
| linea base clase mayoritaria | 0.4988 | 0.3328 | N/A |
| linea base logistica (solo trip_distance) | 0.8308 | 0.8302 | 0.9110 |
| modelo A: regresion logistica completa | 0.8391 | 0.8390 | 0.9173 |
| modelo B: random forest | 0.8836 | 0.8836 | 0.9559 |
| modelo C: HistGradientBoosting | 0.8825 | 0.8825 | 0.9559 |

se selecciono el random forest como modelo final por tener la accuracy mas alta. la confusion matrix muestra errores similares en ambas clases (26,417 falsos positivos y 29,994 falsos negativos), lo que indica un modelo bien balanceado.

---

## 5. robustez y generalizacion

**evaluacion por dia:** el peor desempeño fue en los dias 27 al 30 de enero (accuracy minima 0.8627), consistente con el periodo post-tormenta del a9.

**generalizacion a febrero:**

| metrica | enero 2026 | febrero 2026 |
|---|---|---|
| filas crudas | 3,724,889 | 3,399,866 |
| filas limpias | 2,423,159 | 2,184,699 |
| accuracy | 0.8836 | 0.8793 |
| f1 macro | 0.8836 | 0.8793 |

**feature importance:** las variables mas influyentes fueron trip_distance (0.2005), log1p_distance (0.1368), hora (0.0481), dia_mes (0.0196) y PULocationID (0.0188). al eliminar trip_distance la accuracy subio levemente a 0.8856 porque log1p_distance captura informacion similar.

---

## 6. limitaciones

- dask no soporta median en groupby, se uso mean como aproximacion en el a9
- la feature importance se calculo sobre una muestra de 5,000 filas por limitaciones de ram en google colab
- trip_distance se incluyo bajo el supuesto de que existe una estimacion de ruta disponible al inicio del viaje
