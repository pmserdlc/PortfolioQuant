# PortfolioQuant — Sistema de Análisis y Optimización de Cartera

Sistema cuantitativo de análisis, optimización y recomendación de carteras de inversión para el inversor particular español. Genera un dashboard HTML completamente auto-contenido con todos los datos embebidos, sin necesidad de servidor ni base de datos.

> **Entorno:** Python 3.14 · Windows · `.venv`
> **Última actualización:** 6 de mayo de 2026
> **Estado:** ✅ Producción · ✅ Multiperfil · ✅ Dashboard interactivo

---

## Ejemplo de salida (perfil demo)

El sistema genera automáticamente las siguientes métricas al ejecutar el pipeline completo:

| Métrica | Ejemplo (demo) |
|---|---|
| Capital analizado | 184.270 € |
| Sharpe actual | 0.610 |
| Sharpe óptimo (Máx Sharpe) | 0.705 (+0.095) |
| Rendimiento esperado actual | 14.65% |
| Rendimiento óptimo | 15.96% |
| Volatilidad actual | 18.85% |
| Plusvalía neta latente | 32.470 € |
| IRPF estimado (si se realiza todo) | ~6.699 € |

**Escenarios de estrés incluidos:**

| Escenario | Impacto estimado |
|---|---|
| COVID Crash (Mar 2020, 1 mes) | ~-20% a -26% según escenario |
| Bear 2022 — tipos al alza (año completo) | ~-17% a -19% según escenario |
| Corrección Tech 30% (hipotético) | ~-13% a -15% según escenario |

> Los datos del perfil demo son ficticios y sirven únicamente como referencia de las capacidades del sistema. Cada usuario trabaja con sus propios datos en `datos/cartera/`.

---

## Índice

1. [Arquitectura del sistema](#1-arquitectura-del-sistema)
2. [Datos de entrada](#2-datos-de-entrada)
3. [Pipeline de módulos](#3-pipeline-de-módulos)
4. [Dashboard HTML](#4-dashboard-html)
5. [Motor de alertas de cartera](#5-motor-de-alertas-de-cartera)
6. [Perfiles múltiples](#6-perfiles-múltiples)
7. [Asistente de IA integrado](#7-asistente-de-ia-integrado)
8. [Flujo de trabajo](#8-flujo-de-trabajo)
9. [Dependencias e instalación](#9-dependencias-e-instalación)
10. [Limitaciones conocidas](#10-limitaciones-conocidas)
11. [Roadmap](#11-roadmap)

---

## 1. Arquitectura del sistema

```
optimizacion-inversiones/
├── modelos/                         → pipeline Python (módulos 01-08)
│   ├── 01_descarga_datos.py         → precios, macro, ECB/FRED/CoinGecko, FIGI
│   ├── 02_analisis_cartera.py       → análisis, plusvalías, DCA
│   ├── 03_optimizador.py            → Markowitz (5 escenarios)
│   ├── 03b_optimizador_hrp.py       → HRP (Hierarchical Risk Parity)
│   ├── 03c_bl_optimizer.py          → Black-Litterman + views automáticas
│   ├── 04_recomendador.py           → 7 filosofías de inversión
│   ├── 05_core_satellite.py         → clasificación Core/Satellite por perfil
│   ├── 06_montecarlo.py             → simulaciones Monte Carlo 10k + Cholesky
│   └── 08_stop_loss.py              → monitor stop loss con overlay fiscal IRPF
├── dashboard/
│   ├── generar_dashboard.py         → generador HTML (fuente de verdad)
│   ├── index.html                   → dashboard perfil principal
│   ├── demo.html                    → dashboard perfil demo
│   └── maa3.html                    → dashboard perfil maa3
├── datos/
│   ├── cartera/                     → posiciones, estrategia DCA, movimientos
│   ├── historico/                   → precios, retornos, sentiment, noticias, FIGI cache
│   ├── perfil/                      → perfil_inversor.json, objetivos_financieros.json
│   ├── perfiles/                    → datos por perfil alternativo (demo, maa3)
│   └── productos/                   → universo_productos.json, carteras_referencia.json
├── resultados/
│   ├── principal/                   → JSONs y CSVs del perfil principal
│   ├── demo/                        → JSONs y CSVs del perfil demo
│   └── maa3/                        → JSONs y CSVs del perfil maa3
├── scripts/                         → utilidades de mantenimiento
├── notebooks/                       → análisis interactivo Jupyter
└── .github/                         → instrucciones y skills para GitHub Copilot
    ├── copilot-instructions.md
    ├── skills/                      → 5 dominios especializados
    ├── agents/                      → 4 agentes especializados
    ├── instructions/                → reglas por tipo de archivo
    └── prompts/                     → workflows reutilizables
```

---

## 2. Datos de entrada

### `posiciones_actuales.csv`

Posiciones vivas con coste histórico y valoración actual.

| Campo | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `producto` | texto | Nombre del fondo/activo | `MSCI World Index P ACC EUR` |
| `isin` | texto | ISIN o identificador | `LU1737652583` |
| `tipo` | enum | fondo / etf / acciones / roboadvisor / crypto / efectivo | `fondo` |
| `subtipo` | enum | indexado / activo / monetario / pension / btc / liquido... | `indexado` |
| `entidad` | texto | Broker o entidad custodio | `MyInvestor` |
| `riesgo_1_10` | 1-10 | Nivel de riesgo subjetivo | `7` |
| `coste_eur` | decimal | Precio de coste total en EUR | `25000.00` |
| `valor_mercado_eur` | decimal | Valor de mercado actual en EUR | `31500.00` |
| `ganancia_latente_eur` | decimal | Plusvalía/pérdida latente en EUR | `6500.00` |
| `ganancia_pct` | decimal | Rentabilidad (0.26 = 26%) | `0.26` |
| `notas` | texto | Comentarios libres (entre comillas si hay comas) | |
| `fecha_compra` | YYYY-MM-DD | Fecha de primera aportación (opcional) | `2021-03-15` |
| `divisa_base` | texto | Divisa de cotización (opcional, vacío = EUR) | `USD` |
| `distribucion` | enum | acumulacion / distribucion (opcional) | `acumulacion` |
| `dividendo_pendiente_eur` | decimal | Dividendos sin reinvertir en EUR (opcional) | `0` |
| `anios_minusvalia` | entero | Antigüedad de pérdidas para compensar (opcional) | `2` |

### `estrategia_activa.csv`

Estrategias DCA o traspasos programados activos.

| Campo | Descripción |
|---|---|
| `id` | Identificador único |
| `origen` / `destino` | Producto de origen y destino |
| `importe_mensual_eur` | Aportación periódica |
| `meses` | Duración total en meses |
| `estado` | activa / completada / pausada |
| `fecha_inicio` / `fecha_ultima_ejecucion` | Fechas de control (YYYY-MM-DD) |

### `posiciones_bolsa_americana.csv`

Posiciones en acciones USA cotizadas en USD.

| Campo | Descripción |
|---|---|
| `ticker` | Símbolo de cotización |
| `nombre` / `sector` | Nombre y sector |
| `posicion` | Número de acciones |
| `precio_usd` / `valor_mercado_usd` | Precio y valor actual |
| `base_coste_usd` / `precio_medio_usd` | Coste total y precio medio |
| `pyg_no_realizada_usd` / `pyg_no_realizada_pct` | Plusvalía/pérdida latente |

### Archivos de perfil

| Archivo | Contenido clave |
|---|---|
| `datos/perfil/perfil_inversor.json` | Horizonte, perfil riesgo, hipoteca, plataformas disponibles |
| `datos/perfil/objetivos_financieros.json` | Capital objetivo, aportación mensual, horizonte en años |
| `datos/productos/universo_productos.json` | 24 ETFs/fondos UCITS con TER, Sharpe 5a y plataformas |
| `datos/productos/carteras_referencia.json` | 7 filosofías: Dalio, Browne, Bogle, Swensen, Buffett, ARK, España |

---

## 3. Pipeline de módulos

```
01_descarga_datos.py  →  02_analisis_cartera.py  →  03_optimizador.py
                                                  →  03b_optimizador_hrp.py
                                                  →  03c_bl_optimizer.py
                                                  →  04_recomendador.py
                                                  →  05_core_satellite.py
                                                  →  06_montecarlo.py
                                                  →  08_stop_loss.py
                                                             ↓
                                         dashboard/generar_dashboard.py
```

### `01_descarga_datos.py` — Descarga y contexto macro

Pipeline completo en 10 pasos (~60 segundos):

| Paso | Función | Output |
|---|---|---|
| 1-4 | Precios históricos + indicadores macro + score geopolítico + alertas | `precios_historicos.csv`, `parametros_mercado.json` |
| 5 | Métricas reales del universo de 24 ETFs/fondos | Actualiza `universo_productos.json` |
| 6 | Autodescubrimiento de sectores emergentes (EMA baseline α=0.3) | `autodescubrimiento.json` |
| 7 | Scanner de momentum en 35+ sectores (Sharpe, SMA50, benchmark) | `tendencias_sector.json` |
| 8 | Google News RSS — buzz score 0-100 por tema (15 topics) | `noticias_sectores.json` |
| 9 | Cruce precio × noticias → EMERGENTE / CONFIRMADO / AGOTADO | `oportunidades_detectadas.json` |
| 10 | Alertas accionables con productos concretos y pesos por perfil | `alertas_tendencias.txt` + `.json` |

**Nuevas fuentes de datos integradas (sin coste):**

| Fuente | Función | Output |
|---|---|---|
| **BCE (API pública)** | Tipo libre de riesgo real desde Eonia/€STR diario | `rf` dinámico en `parametros_mercado.json` |
| **FRED (St. Louis Fed)** | Macro USA: CPI, Fed Funds Rate, M2 | `fred_macro` en `parametros_mercado.json` |
| **CoinGecko (REST)** | Precio BTC/EUR sin API key | `btc_eur` en `parametros_mercado.json` |
| **Open FIGI (Bloomberg)** | Resolución ISIN → ticker para fondos europeos | `datos/historico/figi_cache.json` |

```bash
python modelos/01_descarga_datos.py   # ejecutar 1 vez/semana
```

### `02_analisis_cartera.py` — Análisis completo

Análisis de distribución, plusvalías fiscales, cartera USA y estrategia DCA. Genera 6 gráficos y un informe de texto. Usa caché de `parametros_mercado.json` — no requiere conexión a internet.

```bash
python modelos/02_analisis_cartera.py   # ~5s
```

### `03_optimizador.py` — Markowitz Media-Varianza

Optimización con 10 clases de activos, correlaciones empíricas reales (5 años) y restricciones de peso (2%-30% por activo). Genera 3 escenarios: Máximo Sharpe, Mínima Volatilidad, Máximo Rendimiento con vol ≤ 18%.

Métricas incluidas: Sharpe, Sortino, **CVaR 95%**, **Calmar ratio**, **Risk Contribution (RC%)** por activo.

Escenarios de optimización:
- `escenario_max_sharpe()` — Máximo ratio de Sharpe
- `escenario_min_vol()` — Mínima volatilidad
- `escenario_max_rend_riesgo_controlado(vol_max=0.18)` — Perfil agresivo
- `escenario_min_cvar()` — Minimizar Expected Shortfall 95% (Basel III/UCITS)
- `escenario_risk_parity_aproximado()` — Igual contribución fraccional al riesgo

```bash
python modelos/03_optimizador.py   # ~45s
```

### `03b_optimizador_hrp.py` — Hierarchical Risk Parity

Implementación de HRP (López de Prado 2016). Pipeline de 3 pasos: clustering jerárquico Ward → quasi-diagonalización → bisección recursiva. Genera HRP puro, HRP acotado y contribución al riesgo por clase.

```bash
python modelos/03b_optimizador_hrp.py   # ~10s
```

### `03c_bl_optimizer.py` — Black-Litterman

Prior bayesiano de equilibrio CAPM (Π = λΣw_mkt, λ=2.5, τ=0.05) combinado con views generadas automáticamente desde régimen de mercado, VIX y factor scores. Optimización MaxSharpe sobre la distribución posterior BL.

```bash
python modelos/03c_bl_optimizer.py   # ~5s
```

### `scripts/backtester.py` — Walk-forward backtester

Compara 6 modelos fuera de muestra (OOS) con ventanas expanding: Markowitz MaxSharpe, MinVol, MinCVaR, HRP, 1/N y benchmark IWDA. Parámetros: `--train-min 126 --test 21 --embargo 3`. Métricas OOS: Sharpe, Sortino, Max DD, hit rate, CAGR.

```bash
python scripts/backtester.py   # ~30s · 6 modelos
```

### `04_recomendador.py` — Cartera recomendada por filosofía

Combina HRP con el universo de 24 productos reales disponibles en las plataformas del inversor. Filtra por plataformas disponibles, prioriza fondos traspasables (IRPF), y proyecta el capital a N años.

Compara la cartera recomendada contra 7 filosofías: All Weather, Permanent Portfolio, Bogle 3-Fund, Yale Endowment, Buffett Value, ARK Disruptive, Clásica España.

```bash
python modelos/04_recomendador.py   # ~5s
```

### `05_core_satellite.py` — Clasificación Core/Satellite

Clasifica automáticamente cada posición en uno de cuatro buckets (CORE, SATELLITE, ESPECIAL, LIQUIDEZ) según `tipo` y `subtipo`. Compara la distribución actual con el objetivo del perfil de riesgo y genera alertas de rebalanceo si la desviación supera ±5pp.

Targets por perfil de riesgo:

| Perfil | CORE | SATELLITE |
|---|---|---|
| Conservador | 90% | 10% |
| Moderado | 75% | 25% |
| Moderado agresivo | 70% | 30% |
| Equilibrado | 70% | 30% |
| Agresivo | 65% | 35% |
| Muy agresivo | 55% | 45% |

```bash
python modelos/05_core_satellite.py --perfil principal   # ~2s
```

Output: `resultados/<perfil>/core_satellite.json`

### `06_montecarlo.py` — Simulación Monte Carlo patrimonial

10.000 trayectorias de capital a 5, 10 y 15 años usando rendimientos lognormales correlacionados (Cholesky de la covarianza Ledoit-Wolf). Genera percentiles P10/P25/P50/P75/P90, probabilidad de batir inflación (2.5%) y 200 trayectorias de muestra para visualización.

```bash
python modelos/06_montecarlo.py --perfil principal   # ~8s
python modelos/06_montecarlo.py --perfil principal --n-sim 50000   # alta precisión
```

Output: `resultados/<perfil>/montecarlo.json`

### `08_stop_loss.py` — Monitor de stop loss con overlay fiscal

Monitor continuo de stop loss y trailing stop para cada posición. Umbrales diferenciados por `riesgo_1_10`. Calcula el coste fiscal IRPF 2026 estimado si el stop se activa con ganancia, y advierte de la regla de los 2 meses en caso de minusvalía.

| Estado | Significado |
|---|---|
| `ACTIVADO` | La pérdida ya supera el hard stop |
| `CRITICO` | Pérdida entre 75% y 100% del umbral hard stop |
| `ALERTA` | Precio < trailing stop calculado |
| `EN_GANANCIAS` | Sin riesgo de stop; muestra la plusvalía latente |
| `OK` | Dentro de márgenes normales |
| `NO_APLICA` | Roboadvisors, planes de pensiones, efectivo |

Ejecutabilidad: `ACCIONABLE` (ETFs, acciones) vs `INFORMATIVA` (fondos, T+1/T+2).

```bash
python modelos/08_stop_loss.py --perfil principal   # ~3s
```

Output: `resultados/<perfil>/stop_loss.json`

---

## 4. Dashboard HTML

El dashboard se genera como un único archivo HTML auto-contenido con todos los datos embebidos (sin servidor, sin llamadas externas).

### Estructura de pestañas

| Pestaña | Contenido |
|---|---|
| **Perfil** | Situación hipoteca, motor de alertas (23 checks), tabla de posiciones, Core/Satellite, Stop Loss |
| **Optimización** | Pesos Markowitz (3 escenarios), HRP, comparativa y rebalanceo, Monte Carlo patrimonial |
| **Recomendaciones** | Cartera recomendada, 7 filosofías de inversión, proyección de capital |
| **Mercado** | Sentiment Reddit, noticias por sector, tendencias y oportunidades |
| **Estrategia** | DCA activos, movimientos MyInvestor, plusvalías USA |

### Nuevos paneles del dashboard

| Panel | Pestaña | Descripción |
|---|---|---|
| **Core / Satellite** | Perfil | Gráfico doughnut con distribución actual vs objetivo por perfil; alertas de rebalanceo |
| **Monte Carlo** | Optimización | Bandas de percentiles P10/P25/P50/P75/P90 a 15 años; probabilidades por horizonte |
| **Stop Loss Monitor** | Perfil | Tabla de estados por posición con nivel de alerta, ejecutabilidad y nota fiscal IRPF |

### Generación

```bash
python dashboard/generar_dashboard.py                  # → dashboard/index.html
python dashboard/generar_dashboard.py --perfil demo    # → dashboard/demo.html
python dashboard/generar_dashboard.py --perfil maa3    # → dashboard/maa3.html
```

---

## 5. Motor de alertas de cartera

La pestaña **Perfil** incluye 23 checks automáticos agrupados en 6 bloques temáticos:

| Bloque | N | Ejemplos |
|---|---|---|
| **K** — Concentración | 3 | Magnificent 7 >25%, RV+Crypto >85%, USD no cubierto >40% |
| **F** — Fiscal | 4 | Escalonamiento IRPF, minusvalías a punto de caducar, ETF+fondo duplicado, dividendos sin reinvertir |
| **T** — Temporal | 3 | Rebalanceo pendiente (SAD >15pp), liquidez <3 meses gastos, DCA sin ejecutar >45d |
| **O** — Diversificación | 2 | Más de 12 posiciones, posiciones zombi <1.500€ |
| **C** — Contraparte | 3 | Sin diversificación jurisdicciones, suma entidad >FOGAIN 100k€, brokers omnibus |
| **M** — Macro | 2 | Sin activos anti-inflación (oro/TIPS/REITs), sin exposición materias primas |

Severidades: 🔴 Urgente · 🟠 Atención · 🟡 Informativo · 🔵 Oportunidad

---

## 6. Perfiles múltiples

| Perfil | Datos | Dashboard | Descripción |
|---|---|---|---|
| `principal` | `datos/` | `dashboard/index.html` | Cartera real del inversor |
| `demo` | `datos/perfiles/demo/` | `dashboard/demo.html` | Demostración con datos ficticios |
| `maa3` | `datos/perfiles/maa3/` | `dashboard/maa3.html` | Perfil alternativo |

Crear nuevo perfil:
```bash
python scripts/nuevo_perfil.py --nombre <nombre> --desde demo
```

---

## 7. Asistente de IA integrado

El proyecto incluye una arquitectura completa de skills y agentes para GitHub Copilot en `.github/`:

### Agentes especializados (`@nombre-agente`)

| Agente | Capacidades | Acceso |
|---|---|---|
| `@quant-analyst` | Markowitz, HRP, métricas riesgo, backtesting | Solo lectura |
| `@tax-optimizer` | IRPF, traspasos fondos, minusvalías, FOGAIN | Solo lectura |
| `@dashboard-builder` | Modifica generar_dashboard.py y regenera | Lectura + edición + ejecución |
| `@data-updater` | Actualiza CSVs y ejecuta el pipeline | Lectura + edición + ejecución |

### Prompts reutilizables (`/nombre-prompt`)

| Prompt | Descripción |
|---|---|
| `/analizar-cartera` | Análisis ejecutivo completo de la cartera |
| `/nueva-alerta` | Guía para añadir una alerta al motor |
| `/nuevo-perfil` | Crea un nuevo perfil de inversor completo |

---

## 8. Flujo de trabajo

### Actualización semanal completa

```bash
# 1. Actualizar datos de cartera si hubo operaciones
#    editar: datos/cartera/posiciones_actuales.csv

# 2. Descargar datos de mercado (~60s)
python modelos/01_descarga_datos.py

# 3. Ejecutar análisis y optimización (~65s total)
python modelos/02_analisis_cartera.py
python modelos/03_optimizador.py
python modelos/03b_optimizador_hrp.py
python modelos/04_recomendador.py

# 4. Nuevos módulos cuantitativos (~15s total)
python modelos/05_core_satellite.py --perfil principal
python modelos/06_montecarlo.py     --perfil principal
python modelos/08_stop_loss.py      --perfil principal

# 5. Regenerar dashboard
python dashboard/generar_dashboard.py

# — o todo de una vez (incluye todos los módulos y todos los perfiles) —
actualizar_todo.bat

# — omitir backtester (más rápido, para actualizaciones frecuentes) —
set SIN_BACKTESTER=1 && actualizar_todo.bat
```

### Cuándo re-ejecutar cada módulo

| Evento | Módulos |
|---|---|
| Nuevas posiciones / cambio de valor | `02` + `05` + `08` + dashboard |
| Compra o venta de activos | `02` + `03` + `03b` + `05` + `06` + `08` + dashboard |
| Cambio de perfil de riesgo | `03` + `05` (target Core/Satellite) |
| Ejecución mensual de DCA | Actualizar `estrategia_activa.csv` + `02` |
| Cambio de tipo hipoteca / Euribor | Actualizar `perfil_inversor.json` |
| Actualización semanal de mercado | `01` → todos → dashboard |

### Publicar en GitHub

```bash
subir_github.bat   # fetch + rebase + commit + push automático
```

---

## 9. Dependencias e instalación

```bash
# Crear entorno virtual
py -3 -m venv .venv
.venv\Scripts\activate

# Instalar dependencias
pip install -r requirements.txt
```

**Dependencias principales:**

```
yfinance>=0.2.50      → descarga de precios históricos
pandas>=2.2.0         → manipulación de datos
numpy>=1.26.0         → álgebra lineal y optimización
scipy>=1.13.0         → optimización numérica (SLSQP)
matplotlib>=3.8.0     → generación de gráficos
seaborn>=0.13.0       → estilos estadísticos
requests>=2.31.0      → API calls (ApeWisdom sentiment)
jupyter>=1.0.0        → notebooks interactivos
```

---

## 10. Limitaciones conocidas

| Limitación | Causa | Solución actual |
|---|---|---|
| Tickers delisted sin cotización | Posiciones OTC o suspendidas | `valor_mercado_usd = 0`; no contabilizados |
| TC EUR/USD fijo en módulo 02 | Sin API de divisas activa | `01_descarga_datos.py` descarga el tipo real |
| CVaR paramétrico (asume normalidad) | Colas gordas subestimadas | Shocks históricos calibrados como proxy |
| Correlaciones en régimen normal ≠ crisis | Régimen de estrés eleva correlaciones | Escenarios de estrés históricos como corrección |
| Sin ejecución de órdenes automática | Sin integración con API de broker | Alertas de rebalanceo manuales |

---

## 11. Roadmap

### Mejoras de modelos cuantitativos

| Estado | Módulo / Mejora | Descripción |
|---|---|---|
| ✅ Implementado | `03_optimizador.py` — Markowitz (MaxSharpe / MinVol / MaxRend / MinCVaR / RiskParity) | 5 escenarios de optimización; métricas Sharpe, Sortino, CVaR 95%, Calmar, RC% por activo |
| ✅ Implementado | `03b_optimizador_hrp.py` — HRP (Hierarchical Risk Parity) | Bisección recursiva sobre clúster jerárquico, acotamiento de pesos |
| ✅ Implementado | `03c_bl_optimizer.py` — Black-Litterman | Prior bayesiano CAPM (π = λΣw_mkt, λ=2.5, τ=0.05) + views automáticas desde régimen/VIX/factor scores |
| ✅ Implementado | `scripts/backtester.py` — Walk-forward backtester | 6 modelos OOS (MaxSharpe, MinVol, MinCVaR, HRP, 1/N, benchmark); train=126d, test=21d, embargo=3d |
| ✅ Implementado | Ledoit-Wolf shrinkage | Regularización de la covarianza muestral en `01_descarga_datos.py`; coeficiente guardado en `parametros_mercado.json` |
| ✅ Implementado | `port_calmar()` + `port_risk_contribution()` | Calmar ratio paramétrico; RC fraccional rc_i = w_i·(Σw)_i / σ_p |
| ✅ Implementado | `05_core_satellite.py` — Clasificación Core/Satellite | Targets por perfil de riesgo (conservador→muy agresivo); alerta rebalanceo ±5pp; multiperfil |
| ✅ Implementado | `06_montecarlo.py` — Monte Carlo patrimonial | 10k simulaciones, Cholesky/Ledoit-Wolf, horizontes 5/10/15a, percentiles P10-P90, 200 trayectorias |
| ✅ Implementado | `08_stop_loss.py` — Stop Loss Monitor | Umbrales por riesgo_1_10, overlay IRPF, ejecutabilidad ACCIONABLE/INFORMATIVA, regla 2 meses |
| ✅ Implementado | APIs gratuitas en `01_descarga_datos.py` | BCE rf dinámico, FRED macro USA, CoinGecko BTC/EUR, Open FIGI resolución ISIN→ticker |
| 📅 P1 | `07_tax_harvesting.py` | Cosecha fiscal automatizada: pérdidas compensables con ganancia del ejercicio |
| 📅 P1 | `04_auditoria_costes.py` | Scanner de solapamientos entre fondos + TER total de la cartera |
| 📅 P2 | `03e_optimizacion_robusta.py` | Optimización robusta: incertidumbre en μ̂ modelada como elipsoide |
| 📅 P2 | `08_sentimiento.py` | Radar de sentimiento social ampliado (Reddit, Google Trends, RSI) |

### Arquitectura IA (GitHub Copilot)

| Estado | Componente | Descripción |
|---|---|---|
| ✅ Implementado | `copilot-instructions.md` | Contexto global del proyecto (esquemas CSV, convenciones, fiscalidad) |
| ✅ Implementado | 8 Skills (`quant-finance`, `financial-advisor-spain`, `dashboard-builder`, `portfolio-alerts`, `data-pipeline`, `data-engineer`, `risk-manager`, `ml-engineer`) | Conocimiento especializado por dominio |
| ✅ Implementado | 4 Agentes (`Quant Analyst`, `Tax Optimizer`, `Dashboard Builder`, `Data Updater`) | Roles con scope de herramientas delimitado |
| ✅ Implementado | 3 Instructions files + 3 Prompts | Contexto automático por tipo de archivo, tareas predefinidas |
