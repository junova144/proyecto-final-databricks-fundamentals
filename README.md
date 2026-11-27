# proyecto-final-olist-databricks

Pipeline de arquitectura medallón (Bronze → Silver → Gold) en Databricks con Unity Catalog, sobre el dataset público **Olist Brazilian E-Commerce** (Kaggle `olistbr/brazilian-ecommerce`).

## Arquitectura

```
Kaggle (kagglehub + secret scope)
   │
   ▼
olist.bronze.raw_files (volume)  ──▶  olist.bronze   9 tablas, todo texto + auditoría
                                         │
                                         ▼
                                      olist.silver   9 tablas tipadas y con contrato
                                         │           + *_cuarentena + log_calidad
                                         ▼
                                      olist.gold     fact_ventas + 4 tablas de negocio
                                                     + vista pública
```

| Notebook | Capa | Qué hace |
|---|---|---|
| `00_config_secreto.ipynb` | — | Guarda el token de Kaggle en el secret scope `kaggle` (una sola vez) |
| `01bronze/01_ingesta_bronze.ipynb` | Bronze | Descarga los 9 CSV y los carga tal cual como Delta, con `_ingestion_ts` y `_source_file` |
| `02silver/02_transformacion_silver.ipynb` | Silver | Perfilado, tipado (`try_cast`), nombres de negocio, reglas de calidad con cuarentena, deduplicación por clave de negocio, integridad referencial, reconciliación, contratos (`NOT NULL`, `CHECK`, `PRIMARY KEY`), comentarios y etiquetas |
| `03gold/03_modelo_gold.ipynb` | Gold | Control de calidad previo, `fact_ventas` (grano: ítem vendido) con su cuadre, 4 tablas de negocio, vista sin datos personales y permisos de mínimo privilegio |

## Preguntas de negocio (Gold)

1. `ventas_mensuales`: ¿cómo evolucionan pedidos, clientes e ingresos mes a mes?
2. `ranking_categorias`: ¿qué categorías generan más ingresos y cómo se califican?
3. `entregas_por_estado`: ¿en qué estados se entrega más lento y con menos puntualidad?
4. `satisfaccion_por_retraso`: ¿cuánto cae la satisfacción cuando el pedido llega tarde?

## Cómo ejecutarlo (Workflow)

1. Ejecutar `00_config_secreto` una vez para guardar el token de Kaggle.
2. **Jobs & Pipelines → Create job** `pipeline-olist`, con 3 tareas tipo *Notebook* en cadena:
   `bronze` → `silver` (depende de bronze) → `gold` (depende de silver).
3. Parámetro del job: `catalogo = olist` (lo leen los tres notebooks por widget).
4. **Run now**. Todos los notebooks son idempotentes: se pueden ejecutar varias veces sin duplicar datos.

Silver falla a propósito si la reconciliación no cuadra, y Gold no se ejecuta si la última corrida de Silver no cuadró.
