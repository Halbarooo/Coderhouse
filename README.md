# Rendimiento por línea y su relación con victorias y derrotas

Análisis exploratorio con **pandas** sobre `MatchStatsTbl.csv`, un dataset de estadísticas de partidas de un videojuego MOBA. Cada fila es el desempeño de un jugador en una partida (posición, oro, daño, kills, objetivos, visión, ítems, runas, resultado).

**Objetivo:** entender el rendimiento de cada línea Lane y su relevancia en las victorias y derrotas, tomando como referencia el **promedio de cada estadística medida**.

## Origen de los datos
- Archivo: `MatchStatsTbl.csv` (112 688 filas × 31 columnas).
- Fuente original: (Kaggle .
- Entorno: Google Colab (Enlace adjunto)

## Estructura del código
| Bloque | Contenido |
|---|---|
| 1 | Carga del dataset |
| 2 | Radiografía técnica: `df.shape`, `.info()`, `.describe()`, `.isnull().sum()` |
| 3 | Limpieza de datos |
| 4 | Agregaciones de negocio con `groupby` |

## Diagnóstico
- **Nulos:** 0 en todas las columnas.
- **Duplicados** (ignorando `MatchStatsId`, que es único por fila): 0.
- **Fechas:** el dataset no tiene columnas de fecha.
- **Valores a corregir:** 11 registros con `Lane = UNKNOWN`.

## Decisiones de limpieza
1. **Renombrar posiciones** a los nombres usados habitualmente por la comunidad: `UTILITY` → `SUPPORT` y `BOTTOM` → `ADC`. El resto (`TOP`, `JUNGLE`, `MIDDLE`) se mantiene en mayúsculas.
2. **`UNKNOWN` → vacío → eliminación.** Los 11 registros se convierten primero en nulos (`where`) y luego se eliminan con `dropna(subset=["Lane"])`. Se eliminan en vez de imputar porque no hay forma de saber qué posición jugó el participante y asignarle la moda sesgaría los promedios por línea. Filas: **112 688 → 112 677**.
3. **`reset_index(drop=True)`** tras eliminar filas, para que el índice vuelva a ser consecutivo.
4. **Nulos y duplicados:** se mantienen `dropna()` y `drop_duplicates()` (excluyendo `MatchStatsId`) como regla defensiva; en este dataset no eliminan ninguna fila.
5. **Tipos de datos:**
   - `Win` → `int8` (flag 0/1; permite que su promedio sea directamente el porcentaje de victorias).
   - Contadores (`MinionsKilled`, `kills`, `deaths`, `assists`, `visionScore`, `DragonKills`, `BaronKills`, `TurretDmgDealt`) → enteros reducidos con `downcast="integer"`.
   - Columnas que son **identificadores o códigos** (`MatchStatsId`, `SummonerMatchFk`, `EnemyChampionFk`, `PrimaryKeyStone`, `PrimarySlot1-3`, `SecondarySlot1-2`, `SummonerSpell1-2`) → `str`, para que no se sumen ni promedien por error.

## Agregaciones realizadas
| # | Pregunta | Código | Resultado |
|---|---|---|---|
| 1 | ¿Cuál es la línea más jugada? | `groupby("Lane").size()` | **SUPPORT** (24 364 partidas); el reparto entre líneas es casi uniforme (21 683 – 24 364). |
| 2 | ¿Cuál es el promedio de oro por línea? | `groupby("Lane")["TotalGold"].mean()` | ADC 13 723 · JUNGLE 12 675 · MIDDLE 12 001 · TOP 11 446 · SUPPORT 8 456 |
| 3 | ¿Cuánto daño inflige cada línea en una muestra de 10 partidas? | `df.head(10).groupby("Lane")["DmgDealt"].sum()` | TOP 133 427 · MIDDLE 32 055 · JUNGLE 31 693 · SUPPORT 19 571 |

### Lectura de los resultados
- SUPPORT es la línea más jugada y la que menos oro obtiene, lo cual es esperable por su rol (casi no farmea minions).
- ADC es la línea que más oro acumula en promedio, con ~5 300 más que SUPPORT.
- La agregación 3 usa solo las primeras 10 filas **en el orden del archivo**, no una muestra aleatoria: ADC no aparece y TOP concentra 6 de las 10 filas, por lo que la comparación entre líneas no es representativa. Sirve como ejemplo de `head()` + `groupby()`, no como conclusión de negocio.

## Próximos pasos: conectar las líneas con victorias y derrotas
Las tres agregaciones actuales describen el rendimiento por línea, pero todavía **no separan victorias de derrotas**, que es el objetivo del análisis. El siguiente paso natural es agrupar por `Lane` y `Win`:

```python
(df.groupby(["Lane", "Win"])[["TotalGold", "DmgDealt", "kills", "deaths",
                              "assists", "visionScore", "MinionsKilled"]]
   .mean()
   .round(1))
```

Y calcular el win rate por línea (`groupby("Lane")["Win"].mean()`) y la diferencia entre el promedio de cada estadística en victorias y derrotas para identificar qué métrica se asocia más con ganar en cada posición.

## Limitaciones
- El dataset no es balanceado: el win rate global es 53,8 % (no 50 %), lo que sugiere que sigue a jugadores concretos.
- Los códigos (`EnemyChampionFk`, ítems, runas, hechizos) no vienen con tabla de equivalencias, así que no se pueden interpretar por nombre.
- Los promedios muestran **asociaciones**, no causalidad (p. ej., quien va ganando acumula más oro).
- Hay registros con poca o nula actividad (por ejemplo, jugadores sin daño ni ítems) que no se eliminaron y pueden bajar los promedios.

## Cómo reproducirlo
1. Abrir el notebook en Google Colab.
2. Ejecutar la celda de carga y subir `MatchStatsTbl.csv`.
3. Ejecutar los bloques 2 a 4 en orden.
