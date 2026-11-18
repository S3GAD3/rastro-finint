# rastro-finint

Módulo de investigación financiera del kit **rastro**. Cuatro herramientas para análisis de cuentas bancarias, tarjetas, criptoactivos y trazabilidad de movimientos, pensadas para investigación policial y ciberinteligencia.

**100% del lado del cliente. Sin backend, sin APIs externas, sin telemetría.** Todo el procesamiento ocurre en el navegador de quien lo usa; nada se envía a ningún servidor y nada se persiste entre sesiones.

**Enlace a la herramienta:** https://s3gad3.github.io/rastro-finint

## Por qué existe

Las tareas de análisis financiero básico (validar un IBAN, identificar una red de tarjeta, comprobar el formato de una dirección cripto, reconstruir un grafo de movimientos a partir de un extracto) suelen depender de herramientas online que exigen subir datos sensibles a terceros. `rastro-finint` es un único archivo HTML autocontenido que hace todo eso localmente, para poder usarse con datos de un caso real sin que salgan del equipo del investigador.

## Módulos

### 01 · Cuentas / IBAN
- Valida el dígito de control de un IBAN (algoritmo ISO 7064 mod 97) o de un CCC español de 20 dígitos.
- Identifica la entidad bancaria española a partir del código de 4 dígitos, con una **tabla editable** desde la propia interfaz.
- Convierte CCC ↔ IBAN.

### 02 · BIN / Tarjetas
- **Identifica la entidad emisora a partir del BIN** (6-8 primeros dígitos) con una base incrustada de 238.947 prefijos (159.529 con emisor, 12.496 entidades), sin conexión: emisor, país de emisión, tipo (débito/crédito/prepago), producto, web y teléfono del emisor. Busca siempre el prefijo más largo (8 → 6 → 5 dígitos).
- Indica la **entidad actual** cuando el emisor español fue absorbido (p. ej. Banco Popular → Santander, Caja Madrid/Bankia → CaixaBank, CatalunyaCaixa → BBVA), que es a quien hay que dirigir el requerimiento.
- Señala las tarjetas **prepago** y los BIN de **procesadores o entidades centrales** (Servired, Sistema 4B, Euro 6000, Banco Cooperativo…), cuyo emisor real puede ser cualquier entidad adherida.
- Acepta números completos, **enmascarados** (`454881******1234`) o solo el BIN. Valida el número con el **algoritmo de Luhn** y detecta la red (Visa, Mastercard, Amex, Discover, Diners, JCB, UnionPay, Maestro, Mir) por rango IIN público.
- **Análisis masivo**: extrae de cualquier texto todos los números de tarjeta (completos o enmascarados) y los BIN sueltos, identifica cada emisor y exporta a CSV. Los números completos se muestran y exportan siempre enmascarados.
- Permite **cargar un CSV propio de rangos BIN** (columnas `bin`, `entidad`, `pais`, `tipo`), que tiene prioridad sobre la base incrustada.
- **Consulta en línea opcional** del emisor, solo al pulsar y enviando únicamente el BIN (nunca el número completo). Las fuentes se prueban en cascada hasta obtener datos: HandyAPI → Binlist directo → Binlist vía proxy CodeTabs → Binlist vía proxy AllOrigins → CardBinCheck (vía CodeTabs). «Contrastar todas las fuentes» las consulta todas y las muestra en paralelo, marcando si la entidad coincide con la base incrustada o si las fuentes discrepan. Incluye importación manual del JSON de Binlist (útil al abrir el archivo como `file://`, donde algunos servidores bloquean la petición por CORS), las respuestas originales y un informe JSON descargable.
- En el análisis masivo, **«Completar en línea los BIN sin emisor»** (o «Consultar todos en línea») consulta cada BIN distinto una sola vez, añade la columna *Emisor en línea* y la incluye en el CSV exportado.

### 03 · Cripto — BTC / EVM / Tron / LTC / DOGE
- Verifica formato y checksum de direcciones Bitcoin (Base58Check con SHA-256 vía Web Crypto API, y Bech32/Bech32m incluyendo SegWit v0–v16 y Taproot), EVM (0x + 40 hex, **checksum EIP-55 con Keccak-256 incluido**), Tron, Litecoin y Dogecoin, y reconoce hashes de transacción.
- Análisis masivo de direcciones pegadas en bloque, con exportación CSV.
- **No consulta saldo ni histórico en vivo** — eso exigiría un nodo o un explorador externo, y se ha evitado a propósito para no requerir red. En su lugar, permite **importar un export CSV/JSON de un explorador** (Etherscan, mempool.space, Blockchair...) para reconstruir el grafo de trazabilidad offline.
- Directorio local editable de direcciones conocidas (exchanges, sospechosos identificados) para etiquetar nodos en el grafo.
- Enlaces directos a exploradores públicos para consulta manual (se abren solo al pulsarlos).

### 04 · Normalizador + Grafo de movimientos
- Carga extractos bancarios en **CSV, XLSX, TXT, texto copiado de un PDF, Norma 43 (AEB), SWIFT MT940/MT942 e ISO 20022 CAMT.053 / CAMT.052 / CAMT.054**.
- Detecta el formato automáticamente; para CSV/XLSX ofrece un mapeo de columnas corregible a mano.
- **MT940/MT942**: interpreta los campos `:61:`/`:86:`, incluido el `:86:` estructurado con subcampos `?NN` (formato alemán/austriaco) o con códigos `/NAME/`, `/IBAN/`, `/REMI/`, `/EREF/`… (SWIFT, bancos neerlandeses y belgas), anulaciones (`RC`/`RD`) y varios extractos por fichero. Comprueba que cada extracto cuadra (saldo inicial + movimientos = saldo final) y que no hay huecos entre extractos consecutivos de la misma cuenta.
- **CAMT.053/052/054**: lee cualquier versión del esquema (v2 a v8+), con ordenante/beneficiario y su IBAN, referencia EndToEnd y concepto. Desglosa los apuntes agrupados en lote cuando cada operación trae su importe, deja fuera los apuntes no contabilizados (p. ej. `PDNG`) indicándolo, y contrasta saldos de apertura/cierre y totales declarados (`TxsSummry`).
- Calcula el **SHA-256 del fichero original** para la cadena de custodia.
- Normaliza todo a un esquema común (fecha, cuenta, contraparte, concepto, importe, saldo).
- Redacta un **resumen del análisis en lenguaje natural**: periodo cubierto, entradas/salidas/neto, evolución del saldo, contraparte de mayor volumen, movimiento más alto.
- Señala **patrones que pueden merecer revisión** (heurísticas, no conclusiones): movimientos justo por debajo de 3.000 €, grupos de importes similares repetidos en pocos días con la misma contraparte, alto porcentaje de contrapartes sin identificar.
- Construye un **grafo dirigido** de la cuenta investigada (nodo central fijo, admite varias cuentas propias) frente a sus contrapartes:
  - Por defecto, **una flecha por movimiento** — si hay 10 transferencias a la misma cuenta, se ven 10 flechas en abanico, no una agregada. Puede activarse el modo agregado para extractos muy voluminosos.
  - Selector para incluir **todos los movimientos, solo entradas, solo salidas, o una selección manual** marcada en la tabla.
  - Nodos etiquetados con los **dígitos finales** de cada cuenta (más útil que el inicio, que suele compartir entidad/oficina).
  - Ranking de principales contrapartes por volumen.
- Exporta el CSV normalizado.

## Cómo usarlo

No requiere instalación ni build. Es un único archivo HTML.

1. Descarga `index.html`.
2. Ábrelo con cualquier navegador moderno (Chrome, Edge, Firefox).
3. Listo — funciona sin conexión a internet desde el primer momento: las librerías JS van incrustadas en el propio archivo (ver [Dependencias](#dependencias)).

También puede publicarse con GitHub Pages para tenerlo accesible desde una URL propia, sin que eso implique enviar datos a ningún sitio: el procesamiento sigue siendo enteramente local en el navegador de quien lo visita.

## Privacidad y custodia de la evidencia

- Ningún dato introducido (cuentas, tarjetas, direcciones, movimientos importados) se guarda entre sesiones ni se envía a ningún servidor.
- Al recargar la página se pierde todo — es una decisión de diseño deliberada para no dejar rastro de datos de un caso en el propio equipo o en almacenamiento del navegador.
- No usa `localStorage`, `sessionStorage`, cookies ni ningún backend propio.

## Dependencias

Tres librerías JS de terceros, **incrustadas en el propio HTML** (sin llamadas de red, apto para equipos aislados / air-gapped):

- [PapaParse](https://www.papaparse.com/) — parseo de CSV.
- [SheetJS / xlsx](https://sheetjs.com/) — lectura de Excel.
- [D3.js](https://d3js.org/) v7.9.0 — el grafo de trazabilidad.

El resto (IBAN, Luhn, Base58Check, Bech32/Bech32m, Keccak-256, Norma 43, MT940, CAMT) está implementado en el propio archivo.

### Datos incrustados

- **Base BIN**: derivada de [iannuttall/binlist-data](https://github.com/iannuttall/binlist-data), licencia [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/). Se incluye comprimida (gzip + base64, unos 750 KB) y se descomprime en el navegador la primera vez que se usa. La fuente traía un error sistemático que sustituía «L?N» por «lan» en los nombres (`BARCElanA`); 678 nombres se han reconstruido a mano y aparecen marcados como tales en la interfaz.
- **Base de entidades IBAN**: derivada del proyecto [schwifty](https://github.com/mdomke/schwifty) (MIT).

## Limitaciones conocidas

- **BIN/tarjetas**: la base incrustada se actualizó por última vez en 2020 y los rangos BIN se reasignan con el tiempo (sobre todo tras fusiones bancarias). La tabla de entidades actuales españolas es orientativa. Confirma siempre el emisor con la marca de tarjeta o con la propia entidad antes de dirigir un requerimiento; si dispones de un listado más reciente, cárgalo como CSV propio.
- **Cripto**: no hay consulta de saldo/histórico en vivo por diseño (sin red). Una dirección EVM escrita toda en minúsculas o toda en mayúsculas no lleva checksum EIP-55, así que solo puede validarse su formato.
- **MT940**: el campo `:86:` no está normalizado entre bancos; cuando viene en texto libre, el nombre y la cuenta de la contraparte se deducen del concepto igual que en los demás formatos y conviene revisarlos.
- **CAMT**: los lotes sin importe por operación se muestran como un único movimiento (se avisa en las incidencias de lectura).
- **Norma 43**: el parser interpreta los anchos de campo estándar de la especificación AEB, pero algunos bancos usan variantes ligeras. Revisa siempre los primeros resultados contra el extracto original antes de usarlos como prueba.
- **Identificación de entidad por IBAN**: la tabla de códigos de banco españoles es parcial y editable; para certeza total, contrastar con el registro del Banco de España.
- Las señales de "posible fraccionamiento" u otros patrones destacados son heurísticas orientativas, no conclusiones periciales.

## Estructura del repositorio

```
index.html   # herramienta completa (HTML + CSS + JS + librerías incrustadas), archivo único
README.md
```

## Contribuir

Pull requests bienvenidas, especialmente para:
- Ampliar la tabla de códigos de entidad española (o añadir tablas de otros países).
- Mejorar el parser de Norma 43 con variantes específicas de bancos.
- Añadir variantes de MT940 / CAMT de bancos concretos o otros formatos de extracto (OFX, QIF…).

## Autor

S3GAD3 — autor original y responsable del proyecto RASTRO · COMPANY.

## Licencia
Este proyecto se distribuye bajo la licencia MIT.

Copyright © 2026 S3GAD3.

La licencia permite usar, copiar, modificar, publicar, distribuir y crear forks, siempre que se conserve el aviso de copyright y el texto de la licencia, manteniendo así la referencia al autor original.

## Aviso

Herramienta pensada como apoyo al análisis, no como prueba pericial concluyente por sí sola. Los resultados de validación de checksums, detección de patrones y normalización de datos deben revisarse y contrastarse siempre con las fuentes originales.
