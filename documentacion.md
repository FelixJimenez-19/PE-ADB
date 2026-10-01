# Documentación del Proyecto — Gestión de Inventario Farmacia MIA

| Campo | Detalle |
|---|---|
| **Tema** | Módulo de Inventario Farmacia MIA |
| **Nombre del sistema** | Farmacia MIA (nombre ficticio) |
| **Grupo** | Los Reptiles de Marte |
| **Motor de base de datos** | Oracle Database |
| **Usuario/Esquema** | `farmaciamia` (matriz, ver `Farmacia MIA.sql`) |
| **Esquema secundario** | `farmaciamia_sucursal` (ejercicio distribuido, ver `FarmaciaMIA_distribuido.sql`) |
| **Sucursales** | Macas, Sucúa, Méndez, Zamora, Gualaquiza, Loja, Guayas, El Pangui (8) |

---

## 1.1 Planteamiento del problema

Farmacia MIA es una empresa ficticia dedicada a la venta de medicamentos,
productos de belleza, cuidado personal y afines, con presencia en ocho
sucursales. Actualmente enfrenta dificultades para gestionar de forma
eficiente la información de su inventario, proveedores, clientes y ventas.

Los principales problemas identificados son:

1. **Control reactivo de existencias:** los productos se reabastecen solo
   cuando el personal detecta que el stock se agotó o está por agotarse,
   lo que provoca pérdidas de ventas y afecta la continuidad del servicio.
2. **Falta de información consolidada:** no se identifican con claridad los
   productos de mayor y menor demanda por sucursal, generando
   sobreabastecimiento de baja rotación y desabastecimiento de alta demanda.
3. **Uso ineficiente de recursos:** se desperdicia espacio de almacenamiento
   y capital inmovilizado en inventario.
4. **Información dispersa:** proveedores, clientes y ventas no están
   centralizados, dificultando reportes confiables para la toma de decisiones.

**Solución propuesta:** diseñar e implementar una base de datos en Oracle que
centralice la información de las ocho sucursales, permita administrar
inventario, proveedores, clientes y ventas, y facilite reportes de productos
de alta/baja demanda, control de existencias y alertas de stock mínimo.

---

## 1.2 Definición del sistema

### 1.2.1 Ámbito y límites del sistema

**Dentro del alcance (lo que el sistema SÍ hace):**

- Registro y administración de **sucursales**, **productos** y **categorías**.
- Control de **inventario por sucursal** (stock actual, stock mínimo,
  stock máximo y punto de reorden).
- Gestión de **proveedores** y del ciclo de **compras/reabastecimiento**.
- Registro de **clientes** y de **ventas** (cabecera y detalle), incluyendo
  descuento de stock.
- **Alertas automáticas** cuando un producto alcanza o baja del stock mínimo.
- **Reportes** de productos de mayor y menor demanda, desempeño por sucursal,
  ventas por período y estado del inventario.
- Control de acceso mediante **usuarios y roles** del motor Oracle.

**Fuera del alcance (lo que el sistema NO hace):**

- Facturación electrónica ni integración tributaria (SRI).
- Contabilidad general, nómina ni gestión financiera.
- Gestión de recursos humanos (excepto datos básicos del empleado que
  registra la venta).
- Portal web de ventas en línea / e-commerce.
- Logística de última milla o transporte entre sucursales.

### 1.2.2 Interacción con otros sistemas

| Sistema externo | Tipo de interacción | Estado |
|---|---|---|
| Sistema de facturación electrónica (SRI) | Exportación de datos de venta | Futuro / no incluido |
| Lector de código de barras (POS) | Entrada de `cod_producto` en ventas | Incluido (entrada de datos) |
| Sistema contable | Reporte de ventas y compras | Futuro / no incluido |
| Correo electrónico / SMS | Notificación de alertas de stock | Futuro / no incluido |

El sistema es **autónomo**: su núcleo (inventario, proveedores, clientes y
ventas) funciona sin depender de terceros; las integraciones listadas son
extensiones opcionales.

### 1.2.3 Usuarios y áreas de aplicación

| Usuario / Rol | Área | Funciones principales |
|---|---|---|
| Administrador del sistema | TI / Gerencia | Gestión de usuarios, roles, respaldos y configuración |
| Gerente de sucursal | Gerencia | Consulta de reportes y desempeño de su sucursal |
| Jefe de inventario / Bodega | Almacén | Productos, stock, mínimos/máximos, alertas y órdenes de compra |
| Encargado de compras | Adquisiciones | Proveedores, órdenes de compra y reabastecimiento |
| Cajero / Vendedor | Ventas | Registro de clientes y ventas |
| Analista / Auditor | Administración | Consulta y extracción de reportes consolidados |

Áreas de aplicación: **Almacén/Inventario, Adquisiciones (Compras), Ventas,
Administración y Gerencia.**

### 1.2.4 Arquitectura a implementar y justificación

**Arquitectura elegida: base de datos centralizada con modelo cliente-servidor
de tres capas (3-tier), sobre Oracle Database.**

```
   Capa de presentación        Capa de lógica (negocio)        Capa de datos
 ┌──────────────────┐        ┌────────────────────────┐     ┌────────────────────┐
 │ POS / Cajero     │        │  Aplicación / API      │     │  Oracle Database   │
 │ Sucursal Macas   │──HTTP──│  - Reglas de negocio   │────│  Esquema farmacia  │
 │ Sucursal Sucúa   │        │  - Validaciones        │     │  mia               │
 │  ... (8)         │        │  - Alertas / reports   │     │  (centralizado)    │
 │ Gerencia         │        └────────────────────────┘     └────────────────────┘
 └──────────────────┘                 │                              │
                            Procedimientos/funciones      Tablas, vistas,
                            y triggers en Oracle          secuencias, triggers
```

**Justificación:**

- **Centralización:** una sola base de datos elimina la dispersión de
  información entre las 8 sucursales y garantiza datos únicos y confiables.
- **Cliente-servidor de 3 capas:** separa presentación, lógica y datos; facilita
  el mantenimiento, la seguridad y la escalabilidad por sucursal.
- **Oracle Database:** motor robusto con soporte para transacciones (ACID),
  integridad referencial, **triggers** (alertas de stock), **procedimientos
  almacenados** (reportes) y control de acceso por roles, requisitos centrales
  del proyecto.
- **Acceso por roles:** cada área (ventas, almacén, gerencia) accede solo a lo
  que necesita, mejorando la seguridad y la trazabilidad.
- **Base normalizada:** reduce redundancia y anomalías de actualización,
  asegurando la consistencia del inventario y las ventas (ver sección 2).

---

## 1.3 Recolección y análisis de requisitos

Recoger y analizar los requerimientos de los usuarios para dar solución a la
problemática planteada en 1.1.

### 1.3.1 Técnicas de recolección aplicadas

| Técnica | Aplicada a | Propósito |
|---|---|---|
| Entrevistas semiestructuradas | Jefe de inventario, cajero, gerente | Descubrir necesidades reales y validar el alcance |
| Observación directa | Punto de venta y bodega de Macas | Ver el flujo actual de venta y control de stock |
| Análisis de documentos | Kardex, facturas y órdenes de compra | Conocer los datos que hoy se registran en papel |

### 1.3.2 Entrevistas realizadas

**Entrevista 1 — Rosa Quituisaca, Jefa de Inventario (sucursal Macas):**

> "El sistema controla el stock, pero para una farmacia le falta lo más
> delicado: las **fechas de caducidad**. Hoy no sé qué lotes están por vencer;
> se me pierden medicamentos vencidos en bodega y temo vender algo caducado.
> Necesito que me avise **con anticipación** qué está por expirar. Además,
> cuando Macas no tiene un producto y Sucúa sí, hago el traslado por WhatsApp y
> anoto en un cuaderno: quiero que quede **registrado en el sistema**. También
> necesito dejar constancia de la **receta médica** de los medicamentos
> controlados. Y a veces el **precio cambia según la sucursal**."

**Entrevista 2 — Byron Santi, Cajero (sucursal Sucúa):**

> "En hora pico no puedo perder tiempo buscando el producto. Necesito buscar
> por **código de barras o nombre**, que el sistema **descuente el stock solo**
> al cobrar y que **me bloquee si el producto está vencido**. También quiero
> registrar al cliente rápido por cédula y que la venta quede a mi nombre y al
> de mi sucursal."

**Entrevista 3 — Ing. Patricia Núñez, Gerente General:**

> "No puedo esperar reportes de fin de mes. Quiero ver **en tiempo real** las
> ventas de las ocho sucursales, comparar **cuál vende más y cuál menos**, saber
> qué productos son de **alta y baja demanda**, y recibir una **sugerencia de
> cuánto y a quién comprar** según el stock mínimo. Y que todo quede
> **auditado**: quién cambió un precio o movió un producto."

### 1.3.3 Requisitos funcionales

| Código | Requisito | Origen |
|---|---|---|
| RF-01 | Gestionar sucursales, categorías y productos | Gerencia |
| RF-02 | Controlar inventario por sucursal (stock, mínimo, máximo) | Jefe de inventario |
| RF-03 | Emitir **alertas de stock mínimo** por producto y sucursal | Jefe de inventario |
| RF-04 | Gestionar **lotes y fechas de vencimiento** y alertar productos próximos a caducar | Jefe de inventario |
| RF-05 | **Impedir o advertir** la venta de productos vencidos | Cajero / Jefe de inventario |
| RF-06 | Registrar la **receta médica** de medicamentos controlados | Jefe de inventario |
| RF-07 | Registrar ventas con detalle y **descontar stock automáticamente** | Cajero |
| RF-08 | Gestionar clientes (búsqueda por cédula) | Cajero |
| RF-09 | Gestionar proveedores y **órdenes de compra** que actualicen el stock al recibirse | Adquisiciones |
| RF-10 | Registrar **transferencias de productos entre sucursales** | Jefe de inventario |
| RF-11 | Manejar **precio por sucursal** cuando aplique | Jefe de inventario |
| RF-12 | Generar reportes de **alta/baja demanda** y de ventas por sucursal y período | Gerencia |
| RF-13 | Administrar **usuarios y roles** con permisos por área | Administrador |

### 1.3.4 Requisitos no funcionales

| Código | Requisito | Categoría |
|---|---|---|
| RNF-01 | Alertas y reportes disponibles **en tiempo real** | Rendimiento |
| RNF-02 | Acceso restringido según **rol** (ventas, almacén, gerencia) | Seguridad |
| RNF-03 | Datos **íntegros y consistentes** (base normalizada, integridad referencial) | Fiabilidad |
| RNF-04 | **Auditoría** de cambios en precios y movimientos de inventario | Trazabilidad |
| RNF-05 | Soporte para las **8 sucursales** y crecimiento futuro | Escalabilidad |
| RNF-06 | Interfaz simple y rápida para el cajero en hora pico | Usabilidad |
| RNF-07 | Respaldo y recuperación de la información (Oracle) | Disponibilidad |

### 1.3.5 Análisis de los requisitos

Las entrevistas confirman la problemática de 1.1 y **amplían el alcance** en
cuatro necesidades nuevas que el modelo inicial no cubría:

1. **Caducidad y lotes** (RF-04, RF-05): se incorporará la entidad `LOTE`
   (`nro_lote`, `fecha_vencimiento`) asociada al inventario. Es el requisito
   más crítico por el **riesgo sanitario y legal** de vender productos vencidos.
2. **Receta médica** (RF-06): se incorporará la entidad `RECETA` para
   medicamentos controlados.
3. **Transferencias entre sucursales** (RF-10): se incorporará
   `TRANSFERENCIA` + `DETALLE_TRANSFERENCIA` (origen y destino).
4. **Precio por sucursal** (RF-11): el precio se moverá a una entidad
   `PRECIO_SUCURSAL` (o al inventario), dejando de ser un valor único en
   `PRODUCTO`.

Estos hallazgos se reflejan en el modelo de la sección 2 y en el script
`FarmaciaMIA_tablas.sql`.

---

## 2. Diseño de la base de datos normalizada

### 2.1 Modelo conceptual inicial

Punto de partida con **6 entidades** principales (4–5 atributos cada una),
alineadas a la problemática de 1.1. En esta etapa el `total` de la venta se
muestra porque es visible para el usuario; se eliminará en el modelo lógico por
ser un dato derivado.

| # | Entidad | Atributos |
|---|---|---|
| 1 | **SUCURSAL** | `cod_sucursal` (PK), `nombre`, `ciudad`, `direccion`, `telefono` |
| 2 | **PRODUCTO** | `cod_producto` (PK), `nombre`, `descripcion`, `precio`, `requiere_receta` |
| 3 | **PROVEEDOR** | `ruc` (PK), `nombre`, `telefono`, `email`, `direccion` |
| 4 | **INVENTARIO** | `cod_inventario` (PK), `stock`, `stock_min`, `stock_max`, `ubicacion` |
| 5 | **CLIENTE** | `cedula` (PK), `nombres`, `apellidos`, `telefono`, `email` |
| 6 | **VENTA** | `num_venta` (PK), `fecha`, `total`, `tipo_pago`, `estado` |

**Relaciones y cardinalidad:**

- **SUCURSAL** (1) → (N) **INVENTARIO**
- **PRODUCTO** (1) → (N) **INVENTARIO**
- **PROVEEDOR** (1) → (N) **PRODUCTO**
- **CLIENTE** (1) → (N) **VENTA**
- **SUCURSAL** (1) → (N) **VENTA**
- **PRODUCTO** ↔ **VENTA** es **M:N** (se resuelve con `DETALLE_VENTA`).

**Diagrama conceptual:** [Ver en Lucidchart](https://lucid.app/lucidchart/8d76abef-bd2c-4846-9387-8cba36f92405/edit?viewport_loc=-1246%2C40%2C2734%2C1660%2C0_0&invitationId=inv_0805e348-6d39-4ac5-8d34-41626f8b2625)

### 2.2 Entidades propuestas (previo a normalización)

Sucursal, Empleado, Categoría, Producto, Proveedor, Inventario/Sucursal,
Cliente, Venta, Detalle_Venta, Compra y Detalle_Compra.

De las entrevistas de 1.3 surgen además: **Lote**, **Receta**,
**Transferencia** (+ detalle) y **Precio_Sucursal**.

### 2.3 Las 4 matrices de normalización

> Se documentarán exactamente **4 matrices** del proceso de normalización.
> (Borrador inicial — se completan/ajustan según el modelo final.)

**Matriz 1 — Dependencias funcionales**
Tabla universal con todos los atributos y sus dependencias funcionales
(X → Y) detectadas a partir de los requerimientos.

**Matriz 2 — Primera Forma Normal (1FN)**
Se eliminan grupos repetidos y atributos multivaluados; cada celda contiene un
valor atómico y se define la clave primaria de cada relación.

**Matriz 3 — Segunda Forma Normal (2FN)**
Se eliminan las dependencias parciales: todo atributo no clave depende de la
clave primaria completa (aplica a relaciones con clave compuesta, como
`Detalle_Venta` y `Detalle_Compra`).

**Matriz 4 — Tercera Forma Normal (3FN)**
Se eliminan las dependencias transitivas: los atributos no clave no dependen de
otros atributos no clave (por ejemplo, separar `Categoría` de `Producto` y
`Sucursal` de `Empleado`).

### 2.4 Modelo relacional resultante (borrador)

| Tabla | Clave primaria | Propósito |
|---|---|---|
| `SUCURSAL` | `cod_sucursal` | Catálogo de las 8 sucursales |
| `EMPLEADO` | `ced_empleado` | Personal y sucursal a la que pertenece |
| `CATEGORIA` | `cod_categoria` | Categorías de productos |
| `PRODUCTO` | `cod_producto` | Medicamentos y productos, con categoría y proveedor |
| `PROVEEDOR` | `ruc_proveedor` | Proveedores |
| `INVENTARIO` | `cod_inventario` | Stock, mínimos/máximos por producto y sucursal |
| `CLIENTE` | `ced_cliente` | Clientes de la farmacia |
| `VENTA` | `num_venta` | Cabecera de venta |
| `DETALLE_VENTA` | `(num_venta, cod_producto)` | Detalle de la venta |
| `COMPRA` | `num_compra` | Cabecera de compra a proveedor |
| `DETALLE_COMPRA` | `(num_compra, cod_producto)` | Detalle de la compra |
| `LOTE` *(nuevo, RF-04)* | `(cod_inventario, nro_lote)` | Lotes y fecha de vencimiento |
| `RECETA` *(nuevo, RF-06)* | `cod_receta` | Recetas de medicamentos controlados |
| `TRANSFERENCIA` *(nuevo, RF-10)* | `num_transferencia` | Traslados entre sucursales |
| `DETALLE_TRANSFERENCIA` *(nuevo, RF-10)* | `(num_transferencia, cod_producto)` | Detalle del traslado |
| `PRECIO_SUCURSAL` *(nuevo, RF-11)* | `(cod_producto, cod_sucursal)` | Precio diferenciado por sucursal |

> **Pendiente:** definir campos exactos, tipos de datos y relaciones, y
> desarrollar el contenido completo de las 4 matrices.

---

## 3. Diseño lógico y físico (implementación)

### 3.1 Scripts SQL del proyecto

| Archivo | Contenido |
|---|---|
| `Farmacia MIA.sql` | Crea el usuario/esquema **`farmaciamia`** (matriz) con DBA, CONNECT, RESOURCE y cuota. |
| `FarmaciaMIA_tablas.sql` | DDL de las **11 tablas**, secuencias, restricciones (PK, FK, UNIQUE, CHECK, DEFAULT) e **índices**. |
| `FarmaciaMIA_datos.sql` | **5 inserts por tabla** (55) + `COMMIT`, para pruebas. |
| `FarmaciaMIA_distribuido.sql` | Segundo esquema (**sucursal**), vista, vista materializada, job y procedimientos/funciones. |

### 3.2 Restricciones e índices

- **PK simple** en todas las tablas; **PK compuesta** en las intermedias N:M
  (`DETALLE_VENTA`, `DETALLE_COMPRA`).
- **FK** en todas las relaciones; opcionalidad declarada con `NULL`/`NOT NULL`
  (único FK opcional: `VENTA.CED_CLIENTE`, por venta a consumidor final).
- Restricciones **UNIQUE** (`INVENTARIO(COD_PRODUCTO, COD_SUCURSAL)`),
  **CHECK** (`REQUIERE_RECETA IN ('S','N')`, `STOCK >= 0`, `CANTIDAD > 0`,
  `ESTADO IN (...)`) y **DEFAULT** (`REQUIERE_RECETA='N'`, `STOCK=0`,
  `FECHA=SYSDATE`, `ESTADO='PENDIENTE'`).
- **Índices**: uno por cada FK, un **compuesto** (`IX_VENTA_SUC_FECHA`) y un
  **simple** (`IX_PRODUCTO_NOMBRE`); cada uno con su comentario justificativo
  en el script.

---

## 4. Arquitectura distribuida (ejercicio)

> Nota: el diseño base es **centralizado**; este apartado es un ejercicio de
> **arquitectura distribuida** solicitado en clase (esquema matriz + esquema
> sucursal). No cambia el diseño centralizado principal.

### 4.1 Esquema sucursal

- Usuario **`farmaciamia_sucursal`** (password `suc12mia`) con tablas propias
  `INVENTARIO`, `VENTA`, `DETALLE_VENTA` y **datos distintos** a la matriz
  (sucursal Sucúa, ventas 101–105).
- Grants cruzados: `farmaciamia` puede leer el esquema sucursal y viceversa.

### 4.2 Vista simple (UNION ALL + JOIN)

- **`farmaciamia.VW_INVENTARIO_GLOBAL`**: une el inventario de matriz y de
  sucursal con una columna `ORIGEN` y calcula `ESTADO_STOCK` (`ALERTA`/`OK`).

### 4.3 Vista materializada + refresco diario

- **`farmaciamia.MV_VENTAS_GLOBALES`**: consolida ventas y detalle de ambos
  esquemas (`REFRESH COMPLETE ON DEMAND`).
- Job **`JOB_MV_VENTAS_DIARIO`** (`DBMS_SCHEDULER`) con
  `FREQ=DAILY;BYHOUR=1;BYMINUTE=0;BYSECOND=0` → **todos los días a la 01:00**.

### 4.4 Procedimientos y funciones

| Objeto | Tipo | Propósito |
|---|---|---|
| `PR_VENTA_PRODUCTO` | Procedimiento | Registra venta de un producto y descuenta stock validando disponibilidad |
| `PR_ACTUALIZAR_STOCK` | Procedimiento | Actualiza (o crea) el stock de un producto/sucursal |
| `FN_TOTAL_VENTA` | Función | Total de una venta = Σ(cantidad × precio) |
| `FN_STOCK_GLOBAL` | Función | Suma el stock de un producto en **ambos esquemas** (distribuido) |

---

## 5. Estado del proyecto y continuidad

### 5.1 Contexto clave (para retomar)

- **Modelo centralizado**: una sola BD; las sucursales son **filas** de
  `SUCURSAL`, no tablas ni esquemas (el esquema distribuido es solo el
  ejercicio de la sección 4).
- En `VENTA` **no se guarda `TOTAL`** (dato derivado; se calcula en vistas).
- `DETALLE_VENTA.PRECIO_UNIT` se guarda por ser **precio histórico**.
- Se documentarán **exactamente 4 matrices**: DF, 1FN, 2FN, 3FN.
- Entidades nuevas detectadas en 1.3 (aún **no** implementadas en SQL):
  `LOTE`, `RECETA`, `TRANSFERENCIA`, `DETALLE_TRANSFERENCIA`,
  `PRECIO_SUCURSAL`.

### 5.2 Orden de ejecución de scripts

1. `Farmacia MIA.sql`
2. `FarmaciaMIA_tablas.sql`
3. `FarmaciaMIA_datos.sql`
4. `FarmaciaMIA_distribuido.sql`

### 5.3 Pendientes

- [ ] Importar el DDL en **SQL Data Modeler** y **capturar el diseño lógico**
      (el proyecto `logicdesign.dmd` está vacío; falta el import).
- [ ] Completar el contenido de las **4 matrices de normalización**.
- [ ] Implementar en SQL las entidades nuevas (`LOTE`, `RECETA`,
      `TRANSFERENCIA`, `PRECIO_SUCURSAL`).
- [ ] Implementar **triggers**: descuento de stock, **alerta de stock mínimo**,
      bloqueo de productos vencidos y auditoría (RNF-04).
- [ ] Crear **vistas/reportes** de alta/baja demanda y ventas por sucursal (RF-12).

### 5.4 Archivos de la carpeta

| Archivo | Estado |
|---|---|
| `Farmacia MIA.sql` | Listo |
| `FarmaciaMIA_tablas.sql` | Listo |
| `FarmaciaMIA_datos.sql` | Listo |
| `FarmaciaMIA_distribuido.sql` | Listo |
| `documentacion.md` | En curso |
| `logicdesign.dmd` + `logicdesign\` | Vacío (pendiente importar DDL) |
| `diagrama conceptual.png` | Captura del modelo conceptual |
