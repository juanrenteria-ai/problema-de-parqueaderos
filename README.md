# Simulación de Sistema de Pago de Parqueaderos
## Centro Comercial Supercentro — Modelo de Colas M/M/1

**Módulo 4 — Actividad Didáctica 2 | Simulación**

---

## Descripción del Proyecto

Este proyecto implementa una simulación de eventos discretos del sistema de pago del
parqueadero del Centro Comercial Supercentro. El sistema cuenta con **3 cajeros
independientes** en cada punto de salida, modelados como colas **M/M/1** (llegadas
Poisson, servicio Exponencial, 1 servidor por cajero), con filas independientes, sin
deserción ni cambio de cola.

La simulación responde cinco preguntas de análisis (A–E) y sigue el proceso sistemático
de 8 pasos para simulación de eventos discretos.

---

## Parámetros del Sistema

### Tipos de usuario

| Tipo | Tiempo de Servicio | λ individual | Proporción | ρ individual |
|:----:|:------------------:|:------------:|:----------:|:------------:|
| Rápido    | 1 min  (μ = 1.000) | 0.333 clt/min | 25.0 % | 0.333 |
| Normal    | 3 min  (μ = 0.333) | 0.333 clt/min | 20.0 % | 1.000 |
| Lento     | 4 min  (μ = 0.250) | 0.200 clt/min | 27.5 % | 0.800 |
| Muy Lento | 6 min  (μ = 0.167) | 0.143 clt/min | 27.5 % | 0.857 |

> **Nota sobre el tipo Normal:** su ρ individual = 1.0 indica que, si todos los usuarios
> fueran de este tipo, el sistema estaría en el límite de saturación. Esto eleva
> significativamente los tiempos de espera en el sistema mixto.

### Parámetros efectivos calculados (mezcla de tipos)

| Parámetro | Valor | Fórmula |
|:--|:--:|:--|
| E[S] ponderado | 3.60 min | Σ pᵢ × (1/μᵢ) |
| μ efectiva | 0.2778 clt/min | 1 / E[S] |
| ρ efectivo | 0.7389 | Σ pᵢ × ρᵢ |
| λ por cajero | 0.2053 clt/min | ρ × μ |
| λ del sistema | 0.6158 clt/min | λ_cajero × 3 |
| E[S²] hiperexponencial | 32.70 min² | Σ pᵢ × 2 × (1/μᵢ)² |

### Referencias analíticas

| Referencia | Wq | W |
|:--|:--:|:--:|
| M/M/1 (aprox.) | 10.19 min | 13.79 min |
| M/G/1 Pollaczek-Khinchine | 12.85 min | 16.45 min |

> La distribución real del servicio es **hiperexponencial** (mezcla de exponenciales),
> no exponencial pura. Por eso el modelo es estrictamente M/H/1, y la fórmula
> P-K de M/G/1 es la referencia analítica más precisa.

---

## Diseño de la Simulación

### Motor: SimPy (eventos discretos)

- Cada cajero es un `simpy.Resource` con capacidad 1.
- Los usuarios llegan al **sistema** con tasa λ_sistema (proceso Poisson).
- Cada usuario es clasificado con probabilidades (25/20/27.5/27.5 %) y asignado a
  un cajero **aleatoriamente** con distribución uniforme.
- Por el teorema de bifurcación de Poisson, cada cajero recibe en expectativa
  λ_cajero = λ_sistema / 3 llegadas por minuto.

### Métricas recolectadas por réplica

| Métrica | Símbolo | Descripción |
|:--|:--:|:--|
| Tiempo en cola | Wq | Espera desde llegada hasta inicio del servicio |
| Tiempo de servicio | Ws | Duración efectiva del servicio |
| Tiempo en sistema | W | Wq + Ws |
| Conteo por cajero | n | Usuarios atendidos por cajero |
| Conteo por tipo | — | Usuarios por tipo en todos los cajeros |

### Parámetros de ejecución

| Parámetro | Valor | Descripción |
|:--|:--:|:--|
| `DUR_REPLICA` | 480 min | Duración de cada réplica (8 horas) |
| `BASE_SEED` | 2024 | Semilla base (cada réplica usa `BASE_SEED + i×13`) |
| `N_PILOTO` | 200 | Réplicas para análisis de warm-up |
| `N_WARMUP` | 37 | Réplicas descartadas (período transitorio) |
| `N_FINAL` | 163 | Réplicas usadas en el análisis final |
| `NIV_CONF` | 0.95 | Nivel de confianza para intervalos |

---

## Resultados Principales

### Punto A — Cajero más y menos eficiente

| Cajero | Wq media | IC 95 % |
|:--:|:--:|:--|
| Cajero 1 | 10.46 min | [9.35, 11.57] |
| Cajero 2 | 10.87 min | [9.57, 12.18] |
| Cajero 3 | **9.95 min** | [8.79, 11.10] |

- **Más eficiente:** Cajero 3 (Wq = 9.95 min)
- **Menos eficiente:** Cajero 2 (Wq = 10.87 min)
- Los IC se solapan: la diferencia es atribuible a variabilidad estocástica.

### Punto B — Proporciones de usuarios por tipo

Prueba χ²: χ² = 6.86, p-value = 0.076 → **no se rechaza H0** (proporciones
observadas consistentes con las especificadas).

### Punto C — Comparación de escenarios

| Escenario | Wq media | Criterio (≤ 3 min) |
|:--|:--:|:--:|
| A — 3 cajeros (actual) | 10.43 min | No cumple |
| B — 4 cajeros | 5.01 min | No cumple |
| C — Cajeros especializados | 40.65 min | No cumple (inestable) |

**Recomendación:** instalar un cuarto cajero. El Escenario C (especialización sin
rebalanceo de carga) sobrecarga el cajero lento (ρ > 1) y empeora el sistema.

### Punto D — Verificación, Calibración y Validación

- Verificación: error vs. M/M/1 en caso trivial = **3.26 %** (< umbral del 5 %)
- Calibración: Wq simulado vs. M/M/1 = **2.32 %** de error
- Validación: comportamiento acorde con la teoría M/G/1 P-K

### Punto E — Estado transitorio

- Warm-up: primeras **37 réplicas** descartadas
- Diferencia de medias CON vs. SIN transitorio: **0.21 min (1.94 %)**

---

## Prerrequisitos

### Versión de Python

**Python 3.9 o superior.** El notebook se probó con Python 3.14 en Windows 11.

### Dependencias

El notebook instala automáticamente los paquetes faltantes al ejecutarse.
Para instalarlos manualmente con anterioridad:

```bash
pip install simpy numpy scipy matplotlib seaborn pandas plotly bokeh nbformat
```

| Librería | Versión mínima recomendada | Uso |
|:--|:--:|:--|
| `simpy` | 4.0 | Motor de simulación de eventos discretos |
| `numpy` | 1.23 | Cálculos numéricos y generación aleatoria |
| `scipy` | 1.9 | Estadística (t-Student, chi-cuadrado) |
| `matplotlib` | 3.6 | Visualizaciones estáticas |
| `seaborn` | 0.12 | Boxplots y gráficas estadísticas |
| `pandas` | 1.5 | Análisis y tablas de datos |
| `plotly` | 5.0 | Gráficas interactivas (proporciones) |
| `bokeh` | 3.0 | Gráficas interactivas (escenarios) |
| `nbformat` | 5.0 | Solo para `generar_notebook.py` |

---

## Formas de ejecución

### Opción 1 — Jupyter Notebook (recomendada)

Es la forma canónica. Las celdas de Plotly y Bokeh muestran gráficas interactivas.

```bash
jupyter notebook Valencia_Renteria_Juan_Felipe_problemaParqueaderos.ipynb
```

Ejecutar las celdas **en orden secuencial** (Kernel → Restart & Run All).

### Opción 2 — Script no-interactivo

Genera todos los archivos (figuras PNG + informe Word) sin abrir el navegador.
Las gráficas interactivas de Plotly y Bokeh no se muestran, pero el análisis
numérico y las figuras estáticas de Matplotlib/Seaborn se generan completamente.

```bash
python patch_and_run.py
```

### Opción 3 — Regenerar el notebook desde cero

Si se modifica `generar_notebook.py` para cambiar parámetros, ejecutar:

```bash
python generar_notebook.py
```

Esto sobreescribe `simulacion_parqueaderos.ipynb` con los cambios aplicados.

---

## Consideraciones Importantes de Ejecución

### Tiempo de ejecución

La simulación ejecuta un total de **~580 réplicas** (200 piloto + 163 finales × 2
para el escenario B + 163 para el escenario C). En hardware moderno (CPU estándar)
el tiempo total ronda los **3–5 minutos**. No se usa paralelismo; SimPy es
single-threaded por diseño.

### Reproducibilidad

Todas las réplicas usan semillas deterministas derivadas de `BASE_SEED = 2024`:

```python
seed_réplica_i = BASE_SEED + i * 13
```

Cambiar `BASE_SEED` produce resultados distintos pero estadísticamente equivalentes.
Los resultados del notebook son **completamente reproducibles** entre ejecuciones.

### Orden de ejecución de celdas

Las celdas tienen dependencias secuenciales estrictas:

```
Celda 2 (imports/constantes)
    ↓
Celda 8 (funciones SimPy)
    ↓
Celda 12 (200 réplicas piloto)  ←── define N_WARMUP
    ↓
Celda 13 (detección warm-up)    ←── define wq_serie, wq_estab
    ↓
Celda 15 (criterio N réplicas)  ←── define N_FINAL
    ↓
Celda 17 (corrida final)        ←── define res_final, wq_fin
    ↓
Celdas de análisis A, B, C, D, E
```

Ejecutar celdas fuera de orden produce `NameError`. Usar siempre
**Kernel → Restart & Run All**.

### Comportamiento de N_FINAL

El criterio para determinar N réplicas es que la semi-amplitud del IC(95 %) de
Wq sea ≤ 5 % de la media. Dado que la alta varianza del sistema hiperexponencial
(E[S²] = 32.7 min²) impide alcanzar este umbral con solo 163 réplicas disponibles
del piloto, `N_FINAL` toma automáticamente el valor máximo de réplicas en estado
estable (163). Para reducir el IC, se puede aumentar `N_PILOTO` en la Celda 12.

### Bokeh en entornos no-Jupyter

La llamada `output_notebook()` de Bokeh falla fuera de Jupyter. El script
`patch_and_run.py` la desactiva automáticamente. Si se ejecuta el `.ipynb`
fuera de un servidor Jupyter (p. ej., VSCode sin extensión Jupyter activa),
las celdas de Bokeh pueden lanzar advertencias; las demás celdas no se ven
afectadas.

### Compatibilidad de codificación en Windows

La consola de Windows (cmd/PowerShell) usa cp1252 por defecto, que no soporta
caracteres Unicode como μ, λ o ρ. Al ejecutar el script no-interactivo se debe
establecer:

```bash
set PYTHONIOENCODING=utf-8
python _sim_ejecutar.py
```

El notebook en Jupyter no presenta este problema.

---

## Arquitectura del Modelo SimPy

```
                 ┌─────────────────────────────────┐
                 │  generador_llegadas()            │
                 │  Poisson: λ_sistema = 0.616/min  │
                 └────────────────┬────────────────┘
                                  │  np.random.choice(TIPOS, p=props)
                      ┌───────────┼───────────┐
                      │           │           │
                 Cajero 1    Cajero 2    Cajero 3
               Resource(1) Resource(1) Resource(1)
                      │           │           │
               proceso_usuario(cid, tipo)
               ├─ t_espera = env.now - t_llegada
               ├─ t_serv   ~ Exp(service_mean[tipo])
               └─ registra esperas[], servicios[], en_sistema[]
```

---

## Aspectos Teóricos Clave

### Por qué el modelo no es M/M/1 puro

La distribución de tiempos de servicio es una **mezcla de exponenciales**
(distribución hiperexponencial H₄), no una exponencial simple. Esto implica:

- **Coeficiente de variación > 1** → mayor variabilidad que M/M/1
- **Wq real > Wq_MM1** (el modelo M/M/1 subestima la espera)
- La fórmula correcta es la de **Pollaczek-Khinchine (M/G/1)**:
  `Wq = λ·E[S²] / (2·(1−ρ))`

### Por qué el Escenario C (cajeros especializados) es peor

Concentrar todos los usuarios Lentos y Muy Lentos en un solo cajero genera:

```
ρ_cajero_lento = λ_lentos × E[S_lentos]
               = (0.616 × 0.55) × 5.0
               = 1.69  >  1  →  Sistema INESTABLE
```

La cola crece sin límite. La especialización sin rebalanceo de capacidad empeora
el sistema en lugar de mejorarlo.

---

## Resultados de Salida

Al completar la ejecución se generan los siguientes archivos:

| Archivo | Contenido |
|:--|:--|
| `figuras/01_diagrama_sistema.png` | Diagrama del sistema (cajeros, colas, tipos) |
| `figuras/02_distribuciones.png` | PDF de servicio y llegadas por tipo |
| `figuras/03_estado_transitorio.png` | Media acumulada + histograma antes/después |
| `figuras/04_criterio_n_replicas.png` | Convergencia IC-relativo → N réplicas |
| `figuras/05_punto_A_cajeros.png` | Comparación Wq/Ws/W por cajero con IC 95 % |
| `figuras/05b_punto_A_boxplot.png` | Boxplot Seaborn de Wq por cajero |
| `figuras/06_punto_B_usuarios.png` | Proporción observada vs. esperada por tipo |
| `figuras/07_punto_C_escenarios.png` | Wq media por escenario (boxplot + barras) |
| `figuras/08_punto_D_VV.png` | Diagrama de flujo V&V |
| `figuras/09_punto_E_antes_despues.png` | 4 paneles: serie, media acumulada, histogramas |
