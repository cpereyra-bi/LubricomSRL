## Ingesta en la base de datos

### Carga de datos en MySQL

* Se creó el schema en MySQL para poder cargar cada una de las tablas a utilizar para el análisis de preguntas de negocios.

**Creación del Schema proyecto\_final:**

```sql
CREATE SCHEMA proyecto_final;
```

**Creación de tablas y asignación de claves primarias:**

```sql
-- Tabla combustibles
CREATE TABLE combustibles (
  grade_id INT,
  tipo_combustible VARCHAR(30),
  categoria VARCHAR(30),
  PRIMARY KEY (grade_id)
);

-- Tabla playeros
CREATE TABLE playeros (
  playero_id BIGINT PRIMARY KEY,
  intial VARCHAR(50)
);

-- Tabla surtidores
CREATE TABLE surtidores (
  pump_id BIGINT PRIMARY KEY,
  tipo_vehiculos VARCHAR(25)
);

-- Tabla fidelizacion_ON
CREATE TABLE fidelizacion_ON (
  ON_id BIGINT PRIMARY KEY,
  dato_axion VARCHAR(30)
);

-- Tabla pump_sales
CREATE TABLE pump_sales (
  sale_id INT NOT NULL PRIMARY KEY,
  end_date DATE NOT NULL,
  end_time TIME NOT NULL,
  pump_id INT NOT NULL,
  hose_id INT NOT NULL,
  grade_id INT NOT NULL,
  volume DECIMAL(10,3) NOT NULL,
  money DECIMAL(15,2) NOT NULL,
  ppu DECIMAL(10,2) NOT NULL,
  initial_volum DECIMAL(15,2) NOT NULL,
  final_volum DECIMAL(15,2) NOT NULL,
  start_date DATE NOT NULL,
  start_time TIME NOT NULL,
  sale_auth VARCHAR(20) NOT NULL,
  playero_id INT NOT NULL,
  ON_id INT NOT NULL
);
```

### Importación de datos

**Modificación para permitir carga de datos:**

```sql
ALTER TABLE pump_sales MODIFY sale_id VARCHAR(100) NOT NULL;
```

**Carga desde archivo CSV:**

```sql
LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 9.0/Uploads/pump_sales_3erT24.csv'
INTO TABLE pump_sales
FIELDS TERMINATED BY ';'
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS (
  sale_id, end_date, end_time, pump_id, hose_id, grade_id, volume,
  money, ppu, initial_volum, final_volum, start_date, start_time,
  sale_auth, playero_id, ON_id
);
```

### Comprobaciones de consistencia

```sql
-- Revisión general por tabla
SELECT * FROM combustibles;
SELECT COUNT(*) FROM combustibles;
SELECT * FROM playeros;
SELECT COUNT(*) FROM playeros;
SELECT * FROM surtidores;
SELECT COUNT(*) FROM surtidores;
SELECT * FROM fidelizacion_ON;
SELECT COUNT(*) FROM fidelizacion_ON;
SELECT COUNT(*) FROM pump_sales;

-- Validación de duplicados
SELECT sale_id, COUNT(*) FROM pump_sales GROUP BY sale_id HAVING COUNT(*) > 1;

-- Fechas
SELECT MAX(end_date), MIN(end_date) FROM pump_sales;

-- Ticket promedio
SELECT AVG(money) AS Prom_Venta FROM pump_sales;

-- Inconsistencias
SELECT * FROM pump_sales WHERE start_date > end_date;
SELECT COUNT(*) AS filas_inconsistentes FROM pump_sales WHERE ROUND(money / volume, 2) != ppu;
SELECT FORMAT(SUM(ppu * volume),2) AS ventas_totales,
       FORMAT(SUM(money),2) AS ventas_totales2,
       FORMAT((SUM(ppu * volume) - SUM(money)),2) AS diferencia
FROM pump_sales;
```

> *Nota: No se utilizará la columna `money` para análisis debido a las inconsistencias encontradas.*

### Carga de datos en Power BI

* Conexión a través de archivos CSV exportados.

* Se aplicaron transformaciones con Power Query:

  * Transformación de columnas `start_time` y `end_time`:

    ```m
    = Table.TransformColumns(#"Tipo cambiado", {
    {"end_time", each Text.PadStart(Text.From(_), 6, "0"), type text},
    {"start_time", each Text.PadStart(Text.From(_), 6, "0"), type text}
    })
    ```
  * Columnas calculadas:

    ```DAX
    Fecha_hora_inicio = FORMAT(pump_sales_3erT24[start_date], "yyyy-MM-dd") & " " & FORMAT(pump_sales_3erT24[start_time], "HH:mm:ss")
    Fecha_hora_finalizacion = FORMAT(pump_sales_3erT24[end_date], "yyyy-MM-dd") & " " & FORMAT(pump_sales_3erT24[end_time], "HH:mm:ss")

    Turno =
    IF(HOUR([end_time]) >= 6 && HOUR([end_time]) < 14, "Mañana",
        IF(HOUR([end_time]) >= 14 && HOUR([end_time]) < 22, "Tarde", "Noche"))
    ```

* Creación de tabla de fechas y horas.

* Relaciones gestionadas dentro del modelo gráfico.

## Resolución de las preguntas de negocios (Extracto 4/27)

### Análisis del negocio a través de MySQL

Pregunta 1: ¿Cuál es el lubricante más vendido? ¿Y el menos vendido?
Se ejecutó una consulta SQL que suma las cantidades vendidas agrupando por producto. El objetivo fue identificar el producto con mayor y menor volumen de ventas.

```sql
SELECT p.producto_nombre, SUM(fp.cantidad) AS cantidad_total
FROM factura_producto fp
JOIN producto p ON fp.producto_id = p.producto_id
GROUP BY p.producto_nombre
ORDER BY cantidad_total DESC;

Pregunta 2: ¿Cuál es la sucursal que más vende? ¿Y la que menos vende?
Se utilizó una consulta que agrupa las ventas por sucursal y calcula el total de productos vendidos.

```sql
SELECT s.nombre_sucursal, SUM(fp.cantidad) AS total_vendido
FROM factura_producto fp
JOIN factura f ON fp.factura_id = f.factura_id
JOIN sucursal s ON f.sucursal_id = s.sucursal_id
GROUP BY s.nombre_sucursal
ORDER BY total_vendido DESC;

Pregunta 3: ¿Cuál es el mes con mayor volumen de ventas?
Se realizó una agregación por mes a partir de las fechas de facturación.

```sql
SELECT MONTH(f.fecha) AS mes, SUM(fp.cantidad) AS cantidad_total
FROM factura_producto fp
JOIN factura f ON fp.factura_id = f.factura_id
GROUP BY mes
ORDER BY cantidad_total DESC;

Pregunta 4: ¿Cuál es el cliente que más compró en volumen y en valor monetario?
Se analizaron tanto las cantidades compradas como el monto total facturado por cliente.

```sql
SELECT c.nombre_cliente, SUM(fp.cantidad) AS total_cantidad, SUM(fp.cantidad * p.precio_unitario) AS total_valor
FROM factura_producto fp
JOIN factura f ON fp.factura_id = f.factura_id
JOIN cliente c ON f.cliente_id = c.cliente_id
JOIN producto p ON fp.producto_id = p.producto_id
GROUP BY c.nombre_cliente
ORDER BY total_valor DESC;


*Continúa con el análisis de datos y visualización en Power BI.*

