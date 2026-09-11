# ETL Bancario — Detección de Lavado de Dinero (IBM-AML)

Pipeline de datos end-to-end sobre el dataset sintético **IBM Transactions for Anti Money Laundering (HI-Small)**, simulando el trabajo de un Data Engineer en el área de riesgo/compliance de un banco: más de 5 millones de transacciones bancarias reales en estructura, procesadas con Apache Spark y modeladas en un esquema dimensional consultable con AWS Athena.

## Arquitectura

```
S3 (raw)  →  PySpark (transform: limpieza, tipado, modelado dimensional)  →  S3 (curated, Parquet particionado)  →  Glue Catalog  →  Athena (consultas SQL)
```

Todo el stack corre en **AWS** (sin Redshift, por restricción de cuenta free): S3 como data lake, Glue Catalog para metadata/gobernanza, Athena como motor de consulta sobre el lake — patrón "lakehouse" estándar en entornos regulados donde mezclar proveedores de nube no es deseable por compliance.

## Dataset

- **Fuente:** [IBM Transactions for Anti Money Laundering (AML)](https://www.kaggle.com/datasets/ealtman2019/ibm-transactions-for-anti-money-laundering-aml), variante HI-Small
- **Volumen:** 5,078,345 transacciones, 518,581 cuentas
- **Cobertura temporal:** 18 días (1-18 de septiembre de 2022) — verificado directamente contra los datos, no asumido de la documentación

## Calidad de datos

- **Nulos:** 0 en ambas tablas fuente
- **Duplicados:** 9 filas exactas en `transactions` (0.00018%) — eliminadas por ser volumen insignificante, sin patrón sistémico
- **Inconsistencia de bancos:** 8 de 30,478 `bank_id` (0.03%) con dos nombres de banco asociados en pares simétricos, probable artefacto de generación sintética — resuelto tomando un nombre por ID de forma determinística y documentada

## Decisiones de negocio clave

**Monedas no convertidas a una sola divisa.** El 1.42% de las transacciones son cross-currency (moneda de pago ≠ moneda de recepción), y el dataset incluye 15 monedas distintas (incluyendo Bitcoin). Como es un dataset sintético sin tipos de cambio históricos reales asociados, convertir a una moneda base implicaría inventar una tasa de cambio arbitraria. Se optó por mantener los montos en su moneda original y analizar/agregar siempre particionando por moneda — decisión más honesta que una precisión falsa.

**Particionado por `year`/`month`, no por día.** Aunque el dataset cubre 18 días distintos, se mantiene el particionado a nivel mes por simplicidad del caso de uso actual; queda documentado como posible mejora futura si se necesita mayor "partition pruning" en Athena.

## Modelo dimensional (esquema estrella)

**Tabla de hechos:**
- `fact_transactions` — una fila por transacción, con métricas (`Amount Paid`, `Amount Received`) y llaves hacia cada dimensión

**Dimensiones:**
- `dim_currency` — 15 monedas únicas
- `dim_payment_format` — 7 formatos de pago (ACH, Cheque, Credit Card, Cash, Wire, Bitcoin, Reinvestment)
- `dim_bank` — 30,470 bancos únicos, enriquecidos con nombre vía join con `accounts`
- `dim_account` — cuentas y entidades, usando `Account Number` como llave natural
- `dim_date` — 18 fechas, con año/mes/día/día de la semana

## Hallazgos de negocio

- Solo el **0.1%** de las transacciones están marcadas como lavado de dinero (5,177 de 5,078,336) — dataset fuertemente desbalanceado, relevante si se plantea un modelo de clasificación a futuro
- **ACH concentra el 86.6%** de las transacciones de lavado, muy por encima de cheque, tarjeta, efectivo o Bitcoin
- El banco **"Oasis Thrift" (ID 70)** concentra 633 transacciones de lavado — 8x más que el segundo banco con más incidencia — y domina el top de combinaciones banco/moneda/formato en transacciones sospechosas
- Bancos con ID 119 y 222 destacan por mover montos muy altos (~$40M) en relativamente pocas transacciones vía Riyal Saudí/ACH — señal de que el volumen de transacciones no siempre es el indicador más relevante en AML

## Stack técnico

`AWS S3` · `Apache Spark (PySpark)` · `AWS Glue Catalog` · `AWS Athena` · `Parquet`

## Posibles mejoras futuras

- Orquestación con Apache Airflow (ETL → modelo dimensional como tareas independientes)
- Particionado por día para aprovechar mejor el "partition pruning" en Athena
- Tests automatizados de calidad de datos (nulos, duplicados, consistencia de dimensiones) integrados al pipeline
