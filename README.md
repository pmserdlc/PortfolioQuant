# Optimización de Inversiones — Cartera Personal

Sistema de análisis cuantitativo y optimización de cartera **100% local y reproducible**.
Cubre análisis de posiciones, optimización Media-Varianza (Markowitz) y planificación fiscal
sobre carteras multiperfil: fondos indexados, monetarios, ETFs, criptomonedas y acciones
individuales en múltiples entidades y divisas.

> **Entorno:** Python 3.14 · Windows · `.venv`  
> **Fecha última actualización:** 14 de abril de 2026
> **Estado:** ✅ Datos de mercado reales · ✅ Demo pública

---

## Resultados de la última ejecución (13 abr 2026)

| Métrica | Cartera real | Demo ficticia |
|---|---|---|
| Capital analizado | **252.852 €** | 184.270 € |
| Sharpe actual | **0.692** | 0.610 |
| Sharpe óptimo (Máx Sharpe) | **0.894 (+0.203)** | 0.705 (+0.095) |
| Rendimiento esperado actual | 13.76% | 14.65% |
| Rendimiento óptimo | **18.96%** | 15.96% |
| Volatilidad actual | 15.35% | 18.85% |
| Plusvalía neta latente | **40.557 €** | 32.470 € |
| IRPF estimado si se realiza todo | ~8.563 € | ~6.699 € |
| Activos descargados (datos reales) | **26 tickers · 5 años** | — |
| VIX (contexto macro) | 19.7 (Normal, pct 73%) | — |
| Brent crudo | ~$101/barril (+56% YTD) | — |
| Score tensión macro | **90/100 — NIVEL ALTO** | — |
| Régimen S&P500 | Alcista +2.8% vs SMA200 | — |
| Yield 10Y USA | 4.33% | — |

**Escenarios de estrés:**

| Escenario | Cartera actual | Cartera Máx Sharpe |
|---|---|---|
| COVID Crash (Mar 2020, 1 mes) | -20.0% (-47.114€) | -25.9% (-60.864€) |
| Bear 2022 — tipos al alza (año completo) | -16.6% (-38.953€) | -19.2% (-45.278€) |
| Corrección Tech 30% (hipotético 2026) | -13.0% (-30.502€) | -14.7% (-34.669€) |

---

## Índice

1. [Estructura del proyecto](#1-estructura-del-proyecto)
2. [Datos de entrada](#2-datos-de-entrada)
3. [Módulos implementados](#3-módulos-implementados)
4. [Notebooks interactivos](#4-notebooks-interactivos)
5. [Resultados generados](#5-resultados-generados)
6. [Dependencias](#6-dependencias)
7. [Flujo de trabajo y explotación](#7-flujo-de-trabajo-y-explotación)
8. [Limitaciones técnicas conocidas](#8-limitaciones-técnicas-conocidas)
9. [Roadmap — Funcionalidades futuras](#9-roadmap--funcionalidades-futuras)

---

## 1. Estructura del proyecto

```
optimizacion-inversiones/
│
├── datos/
│   ├── cartera/
│   │   ├── posiciones_actuales.csv          → 14 posiciones con coste/valor/ganancia
│   │   ├── posiciones_bolsa_americana.csv   → 28 tickers USA con P&L en USD
│   │   ├── movimientos_myinvestor.csv        → historial de 49 movimientos de caja
│   │   ├── estrategia_activa.csv             → 2 transferencias periódicas activas
│   │   └── cartera_actual.csv                → snapshot simplificado (legacy)
│   ├── perfil/
│   │   ├── perfil_inversor.json              → perfil completo de riesgo (2 inversores)
│   │   └── objetivos_financieros.json        → capital objetivo, horizonte, aportaciones mensuales
│   ├── productos/
│   │   ├── catalogo_productos.csv            → catálogo de productos disponibles
│   │   ├── universo_productos.json           → 24 ETFs/fondos UCITS con TER, Sharpe 5a y plataformas
│   │   └── carteras_referencia.json          → 7 filosofías: Dalio, Browne, Bogle, Swensen, Buffett, ARK, España
│   └── historico/                            → series de precios + JSONs generados por `01_descarga_datos.py`
│       ├── tendencias_sector.json            → scoring momentum de 35+ sectores
│       ├── noticias_sectores.json            → buzz score por tema (Google News RSS)
│       ├── oportunidades_detectadas.json     → cruces precio × noticias (EMERGENTE/CONFIRMADO/AGOTADO)
│       ├── autodescubrimiento.json           → sectores auto-detectados por EMA baseline
│       └── baseline_terminos.json            → baseline de frecuencia de términos (EMA α=0.3)
│
├── modelos/
│   ├── 01_descarga_datos.py                  → descarga de precios + scanner tendencias + noticias + alertas (10 pasos)
│   ├── 02_analisis_cartera.py                → análisis completo de la cartera actual
│   ├── 03_optimizador.py                     → optimización Markowitz (3 escenarios)
│   ├── 03b_optimizador_hrp.py                → Hierarchical Risk Parity (López de Prado 2016)
│   └── 04_recomendador.py                    → recomendador de cartera por perfil y filosofía (HRP + proyección)
│
├── notebooks/
│   ├── analisis_cartera.ipynb                → análisis interactivo ejecutado
│   └── optimizacion_cartera.ipynb            → optimización interactiva ejecutada
│
├── resultados/
│   ├── informe_cartera.txt                   → informe análisis (texto)
│   ├── informe_optimizacion.txt              → informe optimización (texto)
│   ├── pesos_optimos.csv                     → tabla de pesos por escenario
│   ├── informe_hrp.txt                       → comparativa HRP vs Markowitz + contribución al riesgo
│   ├── hrp_pesos.csv                         → pesos HRP puro, HRP acotado y Markowitz comparados
│   ├── informe_recomendacion.txt             → cartera recomendada con TER, proyección y filosofía seleccionada
│   ├── recomendacion_cartera.json            → estructura JSON para integraciones futuras
│   ├── pesos_recomendados.csv                → pesos finales de la cartera recomendada
│   ├── alertas_tendencias.txt                → alertas accionables con productos concretos
│   ├── alertas_tendencias.json               → estructura JSON de alertas para integraciones
│   └── graficos/
│       ├── 01_tipo.png                       → distribución por tipo de activo
│       ├── 02_entidad.png                    → distribución por entidad/broker
│       ├── 03_riesgo.png                     → distribución por nivel de riesgo
│       ├── 04_plusvalias.png                 → mapa de plusvalías/pérdidas latentes
│       ├── 05_estrategia.png                 → simulación estrategia activa
│       ├── 06_usa_sectores.png               → cartera acciones USA por sector
│       ├── 07_frontera_eficiente.png         → frontera Markowitz (script)
│       ├── 08_comparativa_pesos.png          → pesos actual vs óptimos (script)
│       ├── 09_recomposicion_agresivo.png     → plan de rebalanceo (script)
│       ├── 10_mapa_riesgo_rentabilidad.png   → universo activos riesgo/rend (notebook)
│       ├── 11_frontera_eficiente_nb.png      → frontera interactiva (notebook)
│       ├── 12_comparativa_pesos_nb.png       → comparativa 4 escenarios (notebook)
│       └── 13_plan_rebalanceo_nb.png         → plan rebalanceo detallado (notebook)
│
└── requirements.txt                          → dependencias Python
```

---

## 2. Datos de entrada

### Cartera actual (`posiciones_actuales.csv`)
Lista de posiciones vivas con coste histórico y valoración actual.

**Columnas requeridas:**

| Campo | Descripción | Ejemplo |
|---|---|---|
| `producto` | Nombre del fondo/activo | `MSCI World Index P ACC EUR` |
| `isin` | ISIN o identificador | `LU1737652583` |
| `entidad` | Broker/entidad custodia | `MyInvestor` |
| `tipo` | Categoría contable | `indexado` |
| `subtipo` | Subcategoría de riesgo | `rv_global` |
| `coste_eur` | Precio de coste en EUR | — |
| `valor_mercado_eur` | Valor de mercado actual en EUR | — |
| `ganancia_perdida_eur` | Plusvalía/pérdida latente en EUR | — |

Los subtipos reconocidos por el analizador son: `monetario`, `indexado`, `rv_global`, `rv_usa`,
`btc`, `crypto`, `tematico`, `inversion`, `pension`, `liquido`, `renta_variable`.

### Cartera acciones USA (`posiciones_bolsa_americana.csv`)
Posiciones en acciones individuales cotizadas en USD.

**Columnas requeridas:** `ticker`, `empresa`, `sector`, `coste_usd`, `valor_mercado_usd`, `pyg_pct`

Los sectores reconocidos son: `large_cap_tech`, `ai_cloud`, `crypto_mining`, `biotech`,
`micro_cap_especulativo`. El módulo detecta automáticamente posiciones con pérdidas severas
(>80%) y posiciones sin cotización activa (`valor_mercado_usd = 0`).
El tipo de cambio EUR/USD se configura en la constante `TC` del módulo de análisis.

### Perfil inversor (`perfil_inversor.json`)
Parámetros que condicionan los escenarios de optimización y las alertas fiscales.

**Campos relevantes para el optimizador:**
- `horizonte_anos` — años hasta el objetivo de liquidez (condiciona `vol_max`)
- `perfil_riesgo` — `conservador` / `moderado` / `agresivo`
- `tipo_interes_hipoteca` — tipo efectivo anual de deuda (umbral vs. amortización anticipada)
- `pais_fiscal` — régimen IRPF aplicable (`ES` = tramos ahorro españoles)
- `liquidez_necesaria_anos` — años sin necesidad de disponer del capital

---

## 3. Módulos implementados

### `01_descarga_datos.py` — Descarga de precios y contexto macro
Descargador propio basado en `urllib` con acceso directo a la API de Yahoo Finance.

**Pipeline completo en 10 pasos (una sola ejecución):**
- **Pasos 1-4**: 26 activos financieros (ETFs, BTC, acciones USA) · 5 años de histórico, 8 indicadores macro (VIX, EUR/USD, yields, Brent…), score de tensión geopolítica (0-100), alertas automáticas de mercado.
- **Paso 5**: `actualizar_metricas_universo()` — descarga métricas reales de los 24 ETFs del universo de productos vía Yahoo Finance.
- **Paso 6**: `autodetectar_sectores_emergentes()` — descubre automáticamente nuevos sectores emergentes con EMA baseline (α=0.3). Sin intervención manual. Guarda `autodescubrimiento.json`.
- **Paso 7**: `scanner_tendencias()` — scoring de momentum (Sharpe, SMA50, benchmark, retornos) sobre 35+ sectores. Guarda `tendencias_sector.json`.
- **Paso 8**: `descarga_noticias_financieras()` — Google News RSS scraper. Buzz score 0-100 por tema (15 topics + descubrimientos). Sin API key. Guarda `noticias_sectores.json`.
- **Paso 9**: `cruzar_precios_noticias()` — cruza señales de precio con buzz mediático → EMERGENTE / CONFIRMADO / AGOTADO. Guarda `oportunidades_detectadas.json`.
- **Paso 10**: `generar_alertas_accionables()` — para cada tendencia detectada: productos concretos del universo, pesos sugeridos por 3 perfiles de riesgo, aviso fiscal IRPF y plataformas disponibles. Guarda `alertas_tendencias.txt` + `alertas_tendencias.json`.

```bash
python modelos/01_descarga_datos.py   # ~60s · ejecutar 1 vez/semana
```

### `02_analisis_cartera.py` — Análisis de cartera
Análisis completo de la cartera actual. Genera 6 gráficos y un informe de texto.
Usa el caché de `parametros_mercado.json` (generado por `01_descarga_datos.py`) para
la sección de rendimientos históricos de referencia — **sin dependencia de yfinance**.

**Funciones principales:**
- `distribucion()` — reparto por tipo, entidad y nivel de riesgo
- `analizar_plusvalias()` — mapa fiscal con estimación de IRPF por posición (tramos ahorro 2026)
- `analizar_usa()` — detalle de la cartera USA: sectores, P&L, alertas por pérdida >80%
- `simular()` — proyección de la estrategia de traspasos activa (4 meses)
- `generar_informe()` — informe texto con todas las secciones

```bash
python modelos/02_analisis_cartera.py  # ~5s (usa caché JSON)
```

### `04_recomendador.py` — Recomendador de cartera por perfil y filosofía ✅ IMPLEMENTADO (14 abr 2026)
Construye la cartera óptima personalizada usando HRP como motor de pesos y las siguientes entradas:
- `universo_productos.json` — 24 ETFs/fondos UCITS con métricas reales
- `carteras_referencia.json` — 7 filosofías de inversión (Dalio, Browne, Bogle, Swensen, Buffett, ARK, Clásica España)
- `objetivos_financieros.json` — capital objetivo, horizonte, aportaciones y aversión al drawdown
- `perfil_inversor.json` — bloque `perfil_recomendador` con edad, plataformas disponibles y filosofía preferida

**Funciones principales:**
- Filtra el universo por plataformas disponibles del inversor (MyInvestor, Degiro, IBKR…)
- Prioriza fondos traspasables sin tributar (ventaja IRPF)
- Aplica HRP para ponderar la cartera recomendada
- Proyecta el capital a N años con la cartera recomendada
- Compara la cartera recomendada contra las 7 filosofías de referencia

**Resultados última ejecución (14 abr 2026):**

| Métrica | Valor |
|---|---|
| TER medio ponderado | 0.19% |
| Rendimiento esperado (HRP) | 6.1%/año |
| Sharpe estimado | 0.526 |
| Capital proyectado (18a, +2.000€/mes) | 1.299M€ |

**Salidas generadas:**
- `resultados/recomendacion_cartera.json` — estructura completa (pesos, TER, Sharpe, proyección)
- `resultados/informe_recomendacion.txt` — informe legible con filosofía seleccionada
- `resultados/pesos_recomendados.csv`
- `resultados/graficos/rec_01_cartera.png` — composición de la cartera recomendada
- `resultados/graficos/rec_02_proyeccion.png` — trayectoria del capital a 18 años
- `resultados/graficos/rec_03_filosofias.png` — comparativa TER/Sharpe de las 7 filosofías
- `resultados/graficos/rec_04_diversificacion.png` — diversificación UCITS/plataforma/fiscal

```bash
python modelos/04_recomendador.py  # ~5s
```

---

### `03_optimizador.py` — Optimización Markowitz + métricas avanzadas
Optimización Media-Varianza con 10 clases de activos y restricciones reales.
**Carga automáticamente los parámetros históricos reales** desde `parametros_mercado.json`
(rentabilidades y correlaciones empíricas). Si el fichero no existe, usa estimaciones estáticas.

**Universo de activos — parámetros empíricos reales (5 años):**

| Activo | Rend. hist. | Volatilidad | Sharpe | Sortino | Max DD |
|---|---:|---:|---:|---:|---:|
| Monetario (Groupama) | 2,1% | 0,3% | -3.55 | -6.73 | -0,4% |
| RV Global MSCI (MyInvestor) | 11,9% | 14,9% | 0.59 | 0.78 | -23,6% |
| RV Global MSCI (Openbank) | 11,9% | 14,2% | 0.62 | 0.79 | -21,6% |
| RV USA S&P500 (Vanguard) | 13,6% | 17,0% | 0.62 | 0.85 | -24,5% |
| Roboadvisor Indexa | 11,9% | 14,2% | 0.62 | 0.79 | -21,6% |
| Bitcoin (físico + ETP) | 13,2% | 46,6% | 0.22 | 0.30 | -76,6% |
| Acciones USA Large Cap | 32,7% | 36,1% | 0.73 | 1.09 | -53,5% |
| Acciones USA Especulativas | −1,1% | 46,3% | -0.09 | -0.15 | -77,2% |
| ETFs Temáticos | 4,2% | 25,3% | 0.05 | 0.07 | -47,9% |
| Cash | 2,0% | 0,1% | — | — | — |

**Nuevas métricas avanzadas en el informe:**
- **Sortino ratio** (penaliza solo la volatilidad bajista)
- **CVaR 95%** — pérdida esperada anual en el percentil 5% de peores escenarios
- **Escenarios de estrés**: COVID Mar 2020, Bear 2022, Corrección Tech 30%
- **Sección macro**: VIX, EUR/USD, yield curve, régimen S&P500, alertas automáticas

**Resultados última ejecución (13 abr 2026 · datos reales):**

| Escenario | Sharpe | Rend. | Vol. |
|---|---:|---:|---:|
| Cartera actual | 0.692 | 13.76% | 15.35% |
| Máximo Sharpe | **0.894 (+0.203)** | 18.96% | 17.68% |
| Mínima Volatilidad | 0.733 | 9.22% | 8.28% |
| Máx Rend (vol≤18%) | **0.894 (+0.202)** | 19.24% | 18.00% |

**Método:** SLSQP con 50 puntos de arranque aleatorios (semilla fija).
Correlaciones empíricas desde retornos reales (Yahoo Finance v8).

```bash
python modelos/03_optimizador.py  # ~45s
```

---

## 4. Notebooks interactivos

### `analisis_cartera.ipynb`
Versión interactiva del análisis. 5 secciones con salidas embebidas:
1. Distribución de la cartera (3 gráficos)
2. Análisis fiscal de plusvalías/pérdidas
3. Cartera acciones USA (sectores, P&L, alertas)
4. Simulación de estrategia activa
5. Alertas y oportunidades

### `optimizacion_cartera.ipynb`
Versión interactiva de la optimización. 9 secciones:
1. Universo de activos (tabla de parámetros)
2. Mapa riesgo/rentabilidad de activos individuales
3. Cartera actual (métricas y composición)
4. Optimización — 3 escenarios (tabla resumen)
5. Frontera eficiente de Markowitz (gráfico interactivo)
6. Comparativa de pesos — 4 escenarios
7. Plan de rebalanceo — escenario agresivo
8. Impacto fiscal del rebalanceo (IRPF 2026)
9. Resumen ejecutivo y alertas

**Ejecución de notebooks:**
```bash
py -3 -m jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=180 \
  --output-dir notebooks notebooks/optimizacion_cartera.ipynb
```

---

## 5. Resultados generados

| Archivo | Contenido |
|---|---|
| `resultados/informe_cartera.txt` | Análisis completo de cartera (distribución, fiscal, USA) |
| `resultados/informe_optimizacion.txt` | Informe Markowitz con 3 escenarios, Sortino, CVaR y macro |
| `resultados/pesos_optimos.csv` | Tabla de pesos actual vs 3 escenarios óptimos |
| `resultados/informe_hrp.txt` | Comparativa HRP vs Markowitz + contribución al riesgo por clase |
| `resultados/hrp_pesos.csv` | Pesos HRP puro + HRP acotado + Markowitz + RC% |
| `resultados/informe_recomendacion.txt` | Cartera recomendada: TER, Sharpe, proyección y filosofía de ref. |
| `resultados/recomendacion_cartera.json` | Estructura JSON de la cartera recomendada |
| `resultados/pesos_recomendados.csv` | Pesos finales de la cartera recomendada |
| `resultados/alertas_tendencias.txt` | Alertas con sectores emergentes + productos concretos + pesos |
| `resultados/alertas_tendencias.json` | Estructura JSON de alertas para integraciones |
| `resultados/graficos/01–06_*.png` | Gráficos de análisis (distribución, fiscal, USA, estrategia) |
| `resultados/graficos/07–09_*.png` | Gráficos de optimización Markowitz (script) |
| `resultados/graficos/10–13_*.png` | Gráficos de optimización (notebook, mayor detalle) |
| `resultados/graficos/hrp_01–03_*.png` | Dendrograma Ward · pesos HRP vs Markowitz · contribución al riesgo |
| `resultados/graficos/rec_01–04_*.png` | Composición recomendada · proyección capital · filosofías · diversificación |

---

## 6. Dependencias

```
yfinance>=0.2.50      → descarga de precios
pandas>=2.2.0         → manipulación de datos
numpy>=1.26.0         → álgebra lineal (optimización)
scipy>=1.13.0         → optimización numérica (SLSQP)
matplotlib>=3.8.0     → generación de gráficos
seaborn>=0.13.0       → estilos estadísticos
openpyxl>=3.1.0       → lectura/escritura Excel
jupyter>=1.0.0        → notebooks interactivos
ipykernel>=6.29.0     → kernel Python para Jupyter
```

**Instalación:**
```bash
py -3 -m pip install -r requirements.txt
```

---

## 7. Flujo de trabajo y explotación

### Flujo estándar

```
[1] Actualizar datos de cartera
        datos/cartera/posiciones_actuales.csv
        datos/cartera/posiciones_bolsa_americana.csv
        datos/perfil/perfil_inversor.json
              ↓
[2] Descargar datos de mercado  (1 vez/semana)
        python modelos/01_descarga_datos.py
              ↓
        → datos/historico/precios_historicos.csv    (26 activos · 5 años)
        → datos/historico/retornos_historicos.csv  (retornos log diarios)
        → datos/historico/parametros_mercado.json  (stats + macro + correlaciones + score tensión + geopolítico)
              ↓
[3] Ejecutar análisis de cartera
        python modelos/02_analisis_cartera.py       (usa caché JSON automáticamente)
              ↓
        → 6 gráficos en resultados/graficos/
        → resultados/informe_cartera.txt
              ↓
[4] Ejecutar optimizador Markowitz
        python modelos/03_optimizador.py            (usa parámetros reales del JSON)
              ↓
        → 3 gráficos (07–09) en resultados/graficos/
        → resultados/informe_optimizacion.txt       (incluye Sortino, CVaR, estrés, macro)
        → resultados/pesos_optimos.csv
              ↓
[5] Revisar resultados en notebooks
        python -m jupyter notebook
        → notebooks/analisis_cartera.ipynb
        → notebooks/optimizacion_cartera.ipynb
```

### Cuándo re-ejecutar cada módulo

| Evento | Módulos a re-ejecutar |
|---|---|
| Nuevas posiciones o cambio de valoración | `02` + notebook análisis |
| Compra/venta de activos | `02` + `03` |
| Cambio de perfil de riesgo o horizonte | `03` (ajustar `max_w` y escenarios) |
| Nuevo mes de traspasos activos | `02` (actualizar `estrategia_activa.csv`) |
| Cambio de tipo / Euribor | Actualizar `RF` en `03_optimizador.py` |
| Actualizar datos de mercado (semanal) | `01` (30 segundos) |

### Decisiones de inversión que soporta el sistema

1. **¿Cuánto sobrepeso tengo en monetario?**
   → `informe_cartera.txt` sección 2 · gráficos 01–03

2. **¿Qué posiciones vender para aflorar pérdidas fiscales?**
   → `informe_cartera.txt` sección 3 · gráfico 04

3. **¿Cuál es la cartera óptima para mi perfil agresivo?**
   → `informe_optimizacion.txt` · gráfico 11 (frontera eficiente)

4. **¿Qué comprar y vender, y cuánto?**
   → `pesos_optimos.csv` · gráficos 12–13

5. **¿Cuánto IRPF pagaré si rebalanceo?**
   → `optimizacion_cartera.ipynb` sección 8

---

## 8. Limitaciones técnicas conocidas

| Limitación | Causa | Impacto | Solución actual |
|---|---|---|---|
| Tickers delisted o sin cotización | Posiciones OTC o suspendidas | No contabilizadas en valor total | Registrar con `valor_mercado_usd = 0` |
| TC EUR/USD fijo en módulo 02 | Sin API de divisas activa | Pequeña desviación en valoración USD | Actualizar `TC_EURUSD` manualmente; `01_descarga_datos.py` descarga el tipo real |
| CVaR paramétrico (no simulado) | Asunción de normalidad | Subestima colas gordas | Pendiente: Montecarlo para CVaR empírico |
| Correlaciones en crisis subestimadas | Régimen normal ≠ régimen crisis | Escenarios de estrés son más pessimistas | Shocks históricos calibrados en `ESCENARIOS_ESTRES` |

---

## 9. Roadmap — Funcionalidades futuras

Este roadmap recoge las funcionalidades identificadas para ampliar el sistema,
con evaluación de viabilidad técnica y utilidad práctica para el perfil actual.

---

### Bloque 1 — El Núcleo de Eficiencia (Base Clásica)

#### 1.1 Auditoría de Costes y Solapamientos (Scanner de Duplicados)

**Descripción:** Detectar comisiones ocultas por TER de cada fondo/ETF y calcular
el solapamiento real entre posiciones (porcentaje de empresas compartidas entre
dos fondos que replican el mismo índice desde entidades distintas).

**Viabilidad:** Alta. Los TER son datos públicos (KIID/DFI de cada fondo).
El solapamiento entre índices se obtiene de los fact sheets de MSCI/Morningstar.
Implementable con un CSV de composición de índices + script de intersección.

**Utilidad:** Alta cuando la cartera contiene dos posiciones sobre el mismo subyacente
(p.ej., dos fondos que replican el mismo índice en distintas entidades) o cuando las
acciones individuales USA solapan con los ETFs de renta variable. Cuantificar el solapamiento
orientará la decisión de traspaso o consolidación entre fondos equivalentes.

**Módulo sugerido:** `04_auditoria_costes.py`

---

#### 1.2 Mapeo Dinámico de la Frontera Eficiente (Live Data)

**Estado: ✅ IMPLEMENTADO** (13 abr 2026)

El módulo `01_descarga_datos.py` descarga 26 activos, calcula correlaciones empíricas
reales y las inyecta en el optimizador. Sharpe real mejorado de 0.692 → hasta 0.894
con datos de mercado reales. La frontera eficiente (50 puntos) usa rentabilidades y
correlaciones históricas reales de 5 años, no parámetros estáticos de referencia.

---

#### 1.3 Termostato de Riesgo — Perfil Psicológico

**Descripción:** Cuestionario de tolerancia al riesgo que parametriza de forma
continua el `vol_max` del escenario agresivo. En lugar de un límite fijo de 18%,
el sistema escala entre 8% (conservador) y 25% (muy agresivo) según respuestas.

**Viabilidad:** Alta. Es una capa de configuración sobre el optimizador existente.
El cuestionario puede ser un JSON con preguntas + pesos, o un mini-CLI interactivo.

**Utilidad:** Media-alta. Añade valor si en el futuro los inversores actualizan su
perfil (cambio de trabajo, hijos, prejubilación). Para el perfil agresivo actual
el impacto inmediato es bajo.

**Módulo sugerido:** `datos/perfil/cuestionario_riesgo.json` + lógica en `03_optimizador.py`

---

### Bloque 2 — Automatización y Algoritmia (Motor Tecnológico)

#### 2.1 Rebalanceo por Bandas de Tolerancia

**Descripción:** Definir bandas de ±X% alrededor del peso objetivo para cada activo.
Cuando una posición se desvía de su banda, el sistema genera una alerta con la
operación exacta a ejecutar (cuánto vender/comprar), priorizando por coste fiscal.

**Viabilidad:** Alta. Se implementa sobre `pesos_optimos.csv` ya generado.
Requiere actualizar las posiciones actuales periódicamente (mensual o trimestral).
El algoritmo de detección de bandas es sencillo; la priorización fiscal (no vender
lo que tiene más plusvalía si hay alternativa) es la parte más compleja.

**Utilidad:** Muy alta. Es la funcionalidad más accionable a corto plazo.
Elimina la fricción de decidir cuándo rebalancear y cuánto. Aplica directamente
a cualquier estrategia de traspasos periódicos que esté en marcha.

**Módulo sugerido:** `05_rebalanceo_bandas.py` + alerta por email/texto

---

#### 2.2 Simulaciones de Montecarlo

**Descripción:** Proyectar la cartera N años adelante generando miles de trayectorias
de precios (movimiento browniano geométrico con los parámetros del optimizador).
Calcular percentiles de riqueza terminal y probabilidad de alcanzar objetivos.

**Viabilidad:** Alta. Solo requiere numpy (ya instalado). El modelo de un activo
es trivial; para una cartera multidimensional se usa la descomposición de Cholesky
de la matriz de covarianzas (ya construida en `03_optimizador.py`).

**Utilidad:** Alta para el horizonte de 10+ años. Convierte "esperanza de 15%"
en "¿cuál es la probabilidad de que la cartera doble en 5 años?" — información
mucho más accionable para la toma de decisiones. Muy útil para la decisión
amortización anticipada vs. inversión.

**Módulo sugerido:** `06_montecarlo.py` + notebook con percentiles P10/P50/P90

---

#### 2.3 Cosecha Fiscal Automatizada (Tax-Loss Harvesting)

**Descripción:** Escanear continuamente la cartera para detectar posiciones con
pérdidas realizables que puedan compensarse con ganancias del mismo ejercicio fiscal.
Calcular el ahorro fiscal neto de cada operación y ordenarlas por rentabilidad fiscal.

**Viabilidad:** Alta con datos actualizados. La lógica fiscal (compensación
ganancias/pérdidas en España según IRPF ahorro) ya está parcialmente implementada
en `02_analisis_cartera.py`. Ampliar con regla de los 2 meses (no recomprar
en 2 meses o se niega la pérdida) y tratamiento de dividendos.

**Utilidad:** Muy alta cuando la cartera contiene posiciones con pérdidas latentes
significativas (frecuente en carteras con acciones especulativas). Aflorar esas pérdidas
puede compensar ganancias de otros activos y reducir el IRPF del ejercicio.

**Módulo sugerido:** `07_tax_harvesting.py` — ampliar análisis fiscal de `02_analisis_cartera.py`

---

### Bloque 3 — Exploración de Satélites y Oportunidades

#### 3.1 Radar de Sentimiento Social y Algorítmico

**Descripción:** Conexión a APIs de Reddit (r/wallstreetbets, r/investing),
X/Twitter y posiblemente Google Trends para medir el sentimiento y detectar
picos de viralidad en tickers de la cartera USA.

**Viabilidad:** Media-baja. La API de X es de pago desde 2023 (mínimo $100/mes).
Reddit tiene API gratuita pero con límites. Google Trends (pytrends) es gratuito.
La señal de sentimiento es ruidosa y requiere calibración cuidadosa.

**Utilidad:** Baja-media para el perfil actual. El sentimiento social es útil
para timing de entrada/salida en posiciones especulativas (pequeñas en esta cartera).
Para la parte core (fondos indexados, Roboadvisor), es irrelevante y potencialmente
contraproducente para un perfil de largo plazo.

**Módulo sugerido:** `08_sentimiento.py` — solo si la parte especulativa crece

---

#### 3.2 Detector de Anomalías de Mercado (Short Squeezes / Opciones)

**Descripción:** Rastrear datos de interés corto (short interest), días para cubrir
y volumen inusual de opciones (open interest calls/puts) para detectar posibles
short squeezes o movimientos de grandes inversores.

**Viabilidad:** Media. Finviz.com tiene datos de short interest scrapeables.
Los datos de opciones requieren APIs especializadas (CBOE, Tradier — gratuita con límites).
El scraping de Finviz está en zona gris legal.

**Utilidad:** Baja para la cartera actual. Las posiciones especulativas USA
representan <3% de la cartera óptima. El riesgo de actuar sobre señales ruidosas
en este segmento supera el beneficio esperado para un inversor de largo plazo.

**Módulo sugerido:** Diferir hasta que la parte satélite tenga peso relevante.

---

#### 3.3 Filtro de Cuarentena — Core/Satellite con Stop Loss Automático

**Descripción:** Separar explícitamente el presupuesto de riesgo entre core (fondos
indexados, monetario) y satélite (acciones especulativas, cripto alta vol).
Definir stop loss y take profit para posiciones satélite y generar alertas cuando
se alcancen.

**Viabilidad:** Alta para las alertas. El sistema ya segrega las clases de activo
en `03_optimizador.py`. Añadir umbrales de precio en un JSON y un script que compare
contra precios actuales es directo. La ejecución de órdenes automáticas requiere
integración con API de broker (Interactive Brokers tiene API Python gratuita).

**Utilidad:** Alta. La ausencia de stop loss es un error frecuente en carteras con
componente especulativo. Un sistema automático de alertas habría prevenido pérdidas
severas en posiciones que se mantienen por inercia. Esencial como disciplina de riesgo.

**Módulo sugerido:** `09_core_satellite.py` + `datos/cartera/umbrales_sl_tp.json`

---

### Bloque 4 — Más Allá de los Clásicos (Nuevos Modelos)

#### 4.1 Hierarchical Risk Parity (HRP) ✅ IMPLEMENTADO (13 abr 2026)

**Descripción:** Algoritmo HRP de Marcos López de Prado (2016) implementado como
complemento al optimizador Markowitz. Usa clustering jerárquico Ward + bisección
recursiva por varianza-inversa para asignar pesos sin invertir la matriz de
covarianzas (más robusto numéricamente en episodios de correlación inestable).

**Módulo:** `03b_optimizador_hrp.py` — ejecuta en ~0.7 segundos

**Resultados reales (datos 5 años, 963 sesiones, 10 clases de activo):**

| Métrica | HRP acotado | Markowitz Máx Sharpe |
|---|---|---|
| Sharpe | 0.457 | 0.676 |
| Sortino | 0.628 | 0.897 |
| Rendimiento anual | +9.0% | +12.7% |
| Volatilidad anual | 12.8% | 14.2% |
| CVaR 95% | -30.4% | -33.6% |
| Max Drawdown | -18.9% | -17.3% |

Markowitz supera en Sharpe histórico (optimiza explícitamente para ello).
HRP aporta mayor robustez estructural y menor volatilidad en entornos de estrés
(score tensión actual: **90/100 ALTO** — Brent +56%, gold vol ratio 1.6x).

**Salidas generadas:**
- `resultados/hrp_pesos.csv` — pesos HRP puro, HRP acotado y Markowitz comparados
- `resultados/informe_hrp.txt` — informe comparativo completo
- `resultados/graficos/hrp_01_dendrograma.png` — árbol jerárquico Ward
- `resultados/graficos/hrp_02_pesos.png` — comparativa de pesos por clase
- `resultados/graficos/hrp_03_riesgo.png` — contribución al riesgo vs peso

**Cuándo usar HRP en lugar de Markowitz:**
- Score tensión > 70/100 (correlaciones inestables, crisis en curso)
- Como segunda opinión frente a la concentración de Markowitz
- Cuando se añadan más activos individuales (>20) y la covarianza se vuelva ruidosa

---

#### 4.2 Inversión por Factores — Modelo Fama-French

**Descripción:** Aplicar los factores del modelo Fama-French (tamaño, valor, momentum,
calidad) para filtrar las acciones USA de la cartera y construir exposición a factores
con mejor relación riesgo/recompensa que la selección actual.

**Viabilidad:** Media. Los datos de factores Fama-French son públicos y descargables
(Kenneth French Data Library). La parte difícil es conectar los factores con ISINs/tickers
específicos disponibles en MyInvestor o el broker USA.

**Utilidad:** Media. Para la parte core (fondos indexados), el MSCI World ya captura
estos factores implícitamente. La utilidad es mayor para la selección activa de acciones
dentro del bucket especulativo, que actualmente tiene rendimiento negativo esperado.
Puede convertir la cartera especulativa en una cartera factorial racional.

**Módulo sugerido:** `10_factores_fama_french.py` — análogo a los factores de los ETFs smart-beta

---

#### 4.3 Adaptación Macroeconómica — Estilo All Weather

**Descripción:** Monitorizar variables macroeconómicas (inflación IPC, tipos BCE/Fed,
spread crédito, curva de tipos) para ajustar dinámicamente la asignación de activos
según el "clima económico": inflación alta, deflación, crecimiento, recesión.

**Viabilidad:** Media. Los datos macro son públicos (FRED API, BCE, INE).
El modelo "All Weather" de Dalio define 4 cuadrantes macro con asignaciones
de activos específicas. Implementar el detector de régimen macro + reglas
de rebalanceo por régimen es viable en ~3-4 semanas de desarrollo.

**Utilidad:** Alta para el largo plazo. En 2022 la correlación acciones/bonos
se rompió (ambos cayeron por inflación), algo que Markowitz estándar no predijo.
La visión macro permite incorporar activos de cobertura de inflación (TIPS, oro,
commodities) que no están en la cartera actual pero podrían añadir resiliencia.

**Módulo sugerido:** `11_macro_all_weather.py` + indicadores en notebook periódico

---

### Resumen de prioridades del roadmap

| # | Módulo | Utilidad | Viabilidad | Prioridad |
|---|---|---|---|---|
| 2.1 | Rebalanceo por bandas | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **P1 — Inmediato** |
| 2.3 | Tax-Loss Harvesting | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | **P1 — Inmediato** |
| 1.1 | Auditoría costes/solapamientos | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | P2 — Próximo sprint |
| 2.2 | Simulaciones Montecarlo | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | P2 — Próximo sprint |
| 3.3 | Core/Satellite + Stop Loss | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | P2 — Próximo sprint |
| 1.2 | Frontera con live data | ⭐⭐⭐⭐ | ⭐⭐⭐ | P3 — Medio plazo |
| 4.3 | Adaptación macro All Weather | ⭐⭐⭐⭐ | ⭐⭐⭐ | P3 — Medio plazo |
| 4.1 | HRP — López de Prado | ✅ | ✅ | **IMPLEMENTADO** · abr 2026 |
| — | Recomendador por perfil y filosofía | ✅ | ✅ | **IMPLEMENTADO** · abr 2026 |
| — | Scanner de tendencias + alertas accionables | ✅ | ✅ | **IMPLEMENTADO** · abr 2026 |
| 1.3 | Termostato de riesgo (cuestionario) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | P4 — Largo plazo |
| 4.2 | Factores Fama-French | ⭐⭐⭐ | ⭐⭐⭐ | P4 — Largo plazo |
| 3.1 | Radar sentimiento social | ⭐⭐ | ⭐⭐ | P5 — Diferido |
| 3.2 | Detector anomalías / opciones | ⭐⭐ | ⭐⭐ | P5 — Diferido |

---

## Notas técnicas de entorno

```powershell
py -3 modelos/02_analisis_cartera.py
py -3 modelos/03_optimizador.py

# Ejecutar notebooks
py -3 -m jupyter notebook

# Ejecutar notebook en batch (sin abrir navegador)
py -3 -m jupyter nbconvert --to notebook --execute \
  --ExecutePreprocessor.timeout=180 \
  --output-dir notebooks \
  notebooks/optimizacion_cartera.ipynb

# Instalar dependencias
py -3 -m pip install -r requirements.txt
```

> **Disclaimer:** Este sistema es una herramienta de apoyo cuantitativo.
> Los rendimientos esperados son estimaciones históricas de largo plazo y no
> garantizan rentabilidad futura. Consultar con asesor financiero antes de
> ejecutar cualquier rebalanceo.
