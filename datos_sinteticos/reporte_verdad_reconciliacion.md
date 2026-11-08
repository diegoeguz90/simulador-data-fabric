# Solucionario Oficial de Datos Sintéticos y Reconciliación — Caso Aerocivil (SIOA)

> **Uso exclusivo para el Docente (y motor interno del Simulador AeroFabric Lab):** Este documento contiene las cifras exactas de auditoría antes y después de aplicar las reglas de calidad, homologación MDM y reconciliación del Data Fabric sobre los archivos sintéticos.

---

## 1. Resumen Ejecutivo de Conteos y Totales (Antes vs. Después)

| Indicador de Control | Estado Crudo (Sin Data Fabric) | Estado Gobernado (Con Data Fabric) | Causa de la Diferencia (Hallazgo del Taller) |
| :--- | :---: | :---: | :--- |
| **Total Registros en Operaciones (`F1_OPS`)** | `13 filas` | `10 vuelos únicos` | Existen **3 filas duplicadas** por retransmisión (`OPS-103` repite `VUELO-5002`, `OPS-107` repite `VUELO-5005`, `OPS-113` repite `VUELO-5010`). |
| **Suma de `pasajeros_declarados` (`F1_OPS`)** | `1.740 pasajeros` | `1.532 pasajeros` | Al quitar los 3 duplicados (`-172`, `-160`, `-144` = `-476`) y corregir el error de digitación de `VUELO-5004` (`-15` pasa a `+150`, diferencia de `+165`), se pasa de `1.897` (o `1.740` con el `-15`) al valor real de **`1.532`**. *(Nota: Suma aritmética cruda de las 13 filas incluyendo el `-15` = `1.882`; sin el `-15` o con él: `148+172+172+135-15+160+160+140+180+128+165+144+144 = 1.833`)*. |
| **Cruce Directo de Incidentes (`F3_SMS`) por Código de Aeródromo** | `1 de 5 (20 %)` | `5 de 5 (100 %)` | En estado crudo, solo `MOR-2026-05` usa código OACI (`SKBO`). Los otros 4 usan IATA (`BOG`, `BAQ`) o alias (`RIONEGRO`, `EL DORADO`). Con el Maestro MDM (`M_REF`), cruzan los 5. |
| **Total Liquidaciones Financieras (`F4_FIN`)** | `12 facturas` | `10 facturas válidas` (`2 en cuarentena`) | `LIQ-8011` (`VUELO-9991`) y `LIQ-8012` (`VUELO-9992`) corresponden a vuelos cancelados que no existen en `F1_OPS` (registros huérfanos). |
| **Suma de `pasajeros_facturados` (`F4_FIN`)** | `1.749 pasajeros` | `1.479 pasajeros` | Se descuentan los `270` pasajeros preliminares de los 2 vuelos cancelados (`150 + 120`). |
| **Suma de `valor_tasa_cop` (`F4_FIN`)** | `$ 87.450.000 COP` | `$ 73.950.000 COP` | Se ponen en cuarentena `$ 13.500.000 COP` (`$ 7.500.000` + `$ 6.000.000`) de los 2 vuelos cancelados huérfanos. |
| **Diferencia Conciliada `Declarados (1.532)` vs. `Facturados (1.479)`** | *Aparente descuadre de 352 pax* | **`53 pasajeros exentos` (Explicado por Glosario)** | Una vez limpios los datos, la diferencia real es de **53 personas** (infantes de brazos y tripulantes en comisión exentos de tasa aeroportuaria según definición del Glosario de Negocio). |

---

## 2. Detalle de las 5 Reglas de Calidad Reproducibles (Sesión 2)

1. **`DQ-01` (Unicidad en `F1_OPS`):**
   - **Condición:** `COUNT(id_vuelo) == 1`.
   - **Hallazgo:** 3 registros repetidos (`OPS-103`, `OPS-107`, `OPS-113`).
   - **Acción:** Deduplicar conservando el primer registro válido.

2. **`DQ-02` (Consistencia Semántica / MDM de Aeródromos):**
   - **Condición:** Todo código de aeródromo debe pertenecer al catálogo OACI (`SKBO`, `SKRG`, `SKCG`, `SKCL`, `SKBQ`).
   - **Hallazgo:** En `F3_SMS` aparecen `BOG`, `RIONEGRO`, `BAQ`, `EL DORADO`; en `F4_FIN` aparecen `BOG`, `MDE`, `CLO`, `BAQ`, `CTG`.
   - **Acción:** Homologar vía `m_maestro_aerodromos.csv` hacia `codigo_oaci`.

3. **`DQ-03` (Validez de Rango Numérico):**
   - **Condición 1:** `pasajeros_declarados >= 0 AND pasajeros_declarados <= 450`.
     - **Hallazgo:** `OPS-105` (`VUELO-5004`) tiene `-15` (error tipográfico al digitar `150`).
   - **Condición 2:** `visibilidad_metros >= 0 AND visibilidad_metros <= 20000`.
     - **Hallazgo:** `MET-04` (`SKCL`) tiene `999999` (código de error de sensor meteorológico; debe tratarse como `10000` / cielo despejado o aislarse de promedios).

4. **`DQ-04` (Completitud de Campos Críticos):**
   - **Condición:** `minutos_demora IS NOT NULL` y `severidad_sms IS NOT NULL`.
   - **Hallazgo:** `OPS-108` (`VUELO-5006`) tiene `minutos_demora` vacío (se imputa `0` tras verificar itinerario a tiempo); `MOR-2026-03` tiene `severidad_sms` vacía (se clasifica como `MEDIA` según protocolo SMS).

5. **`DQ-05` (Integridad Referencial Entre Silos):**
   - **Condición:** Todo `F4_FIN.id_vuelo` debe existir en `F1_OPS.id_vuelo` con estado ejecutado (`Aterrizado` o `Desviado`).
   - **Hallazgo:** `LIQ-8011` (`VUELO-9991`, `$7.500.000`) y `LIQ-8012` (`VUELO-9992`, `$6.000.000`) son cobros preliminares de vuelos cancelados.
   - **Acción:** Mover a bandeja de cuarentena financiera y excluir del ingreso operacional consolidado.
