# BikeZ Data Warehouse - Proyecto SSIS

## Descripción General

Este proyecto implementa un **Data Warehouse (DW)** para la empresa **BikeZ** utilizando **SQL Server Integration Services (SSIS)**. El objetivo es extraer, transformar y cargar datos desde la base de datos operacional de BikeZ hacia un data warehouse optimizado para análisis.

## Arquitectura del Proyecto

El proyecto incluye:

- **Dimensiones (Dim)**:
  - `DimCliente`: Información de clientes
  - `DimEmpleado`: Información de empleados
  - `DimGeografia`: Información geográfica (territorios y países)
  - `DimProducto`: Información de productos
  - `DimTiempo`: Dimensión de fechas (calendario)

- **Tabla de Hechos (Fact)**:
  - `FactVentas`: Información de ventas con claves foráneas a todas las dimensiones

## Requisitos Previos

### Software Requerido
- **SQL Server 2019 o superior** (se desarrolló en SQL Server 2022)
- **SQL Server Data Tools (SSDT)** para Visual Studio 2022
- **Visual Studio 2022** (o versión compatible)

### Bases de Datos
- **BikeZ**: Base de datos operacional (fuente de datos)
- **BikeZ_DW**: Base de datos data warehouse (destino)

### Credenciales y Conexiones
- Usuario con permisos para leer de la base de datos BikeZ
- Usuario con permisos para escribir en la base de datos BikeZ_DW

## Pasos para Ejecutar el Proyecto

### 1. Preparación Inicial

#### 1.1 Clonar el Repositorio
```bash
git clone https://github.com/Hewrek/BikeZ_DW.git
cd BikeZ_DW
```

#### 1.2 Crear las Bases de Datos
Ejecute los siguientes scripts en SQL Server Management Studio (SSMS):

```sql
-- Crear base de datos BikeZ_DW si no existe
CREATE DATABASE BikeZ_DW;
```

#### 1.3 Crear las Tablas del DW
En la base de datos `BikeZ_DW`, ejecute los scripts de creación de tablas:

```sql
-- Tabla de Dimensión: Cliente
CREATE TABLE dbo.DimCliente (
    ClienteKey INT PRIMARY KEY IDENTITY(1,1),
    ClienteID INT NOT NULL,
    Nombre NVARCHAR(50),
    Apellido NVARCHAR(50),
    NombreCompleto NVARCHAR(101)
);

-- Tabla de Dimensión: Empleado
CREATE TABLE dbo.DimEmpleado (
    EmpleadoKey INT PRIMARY KEY IDENTITY(1,1),
    EmpleadoID INT NOT NULL,
    Cargo NVARCHAR(50),
    EstadoCivil NVARCHAR(10),
    Genero NVARCHAR(10)
);

-- Tabla de Dimensión: Geografía
CREATE TABLE dbo.DimGeografia (
    GeografiaKey INT PRIMARY KEY IDENTITY(1,1),
    TerritorioID INT NOT NULL,
    Territorio NVARCHAR(50),
    Grupo NVARCHAR(50),
    Pais NVARCHAR(50)
);

-- Tabla de Dimensión: Producto
CREATE TABLE dbo.DimProducto (
    ProductoKey INT PRIMARY KEY IDENTITY(1,1),
    ProductoID INT NOT NULL,
    Producto NVARCHAR(50),
    Color NVARCHAR(15),
    Categoria NVARCHAR(50)
);

-- Tabla de Dimensión: Tiempo
CREATE TABLE dbo.DimTiempo (
    FechaKey INT PRIMARY KEY,
    Fecha DATE,
Ano INT,
    Trimestre INT,
 Mes INT,
    NombreMes NVARCHAR(20),
    Dia INT,
  DiaDeLaSemana NVARCHAR(20)
);

-- Tabla de Hechos: Ventas
CREATE TABLE dbo.FactVentas (
    ClienteKey INT NOT NULL,
    GeografiaKey INT NOT NULL,
    ProductoKey INT NOT NULL,
    EmpleadoKey INT NOT NULL,
    FechaKey INT NOT NULL,
    FechaEnvioKey INT,
 VentaID INT NOT NULL,
    DetalleVentaID INT NOT NULL,
    Cantidad SMALLINT,
    PrecioUnitario MONEY,
    TotalVenta AS (Cantidad * PrecioUnitario) PERSISTED,
    PRIMARY KEY (VentaID, DetalleVentaID),
    FOREIGN KEY (ClienteKey) REFERENCES dbo.DimCliente(ClienteKey),
    FOREIGN KEY (GeografiaKey) REFERENCES dbo.DimGeografia(GeografiaKey),
    FOREIGN KEY (ProductoKey) REFERENCES dbo.DimProducto(ProductoKey),
    FOREIGN KEY (EmpleadoKey) REFERENCES dbo.DimEmpleado(EmpleadoKey),
    FOREIGN KEY (FechaKey) REFERENCES dbo.DimTiempo(FechaKey)
);
```

### 2. Configurar el Proyecto SSIS

#### 2.1 Abrir el Proyecto
1. Abra **Visual Studio 2022**
2. Vaya a **File ? Open ? Project/Solution**
3. Navegue a la carpeta clonada y seleccione `BikeZ_DW.sln`

#### 2.2 Configurar Gestores de Conexión
1. En el **Solution Explorer**, haga clic derecho en **Connection Managers**
2. Edite los siguientes gestores de conexión:
   - **Nefi-pc.BikeZ1**: Apunte a la base de datos operacional `BikeZ`
   - **Nefi-pc.BikeZ_DW1**: Apunte a la base de datos data warehouse `BikeZ_DW`

**Nota**: Si los nombres de servidor son diferentes, actualice los nombres de los gestores de conexión según su configuración local.

### 3. Ejecutar el Paquete SSIS

#### Opción A: Desde Visual Studio
1. Haga clic derecho en **Package.dtsx** en el **Solution Explorer**
2. Seleccione **Execute Package**
3. Observe la ejecución en la ventana de **Output**

#### Opción B: Desde SQL Server Integration Services
1. Abra **SQL Server Integration Services 2022**
2. Naveg a la carpeta donde se encuentra el proyecto
3. Haga clic derecho en el paquete y seleccione **Run Package**

#### Opción C: Desde la Línea de Comandos
```bash
dtexec /F "C:\ruta\al\BikeZ_DW\Package.dtsx"
```

### 4. Flujo de Ejecución del Paquete

El paquete ejecuta las siguientes tareas en orden:

1. **Limpieza de Tablas DW**: Trunca la tabla `FactVentas`
2. **Cargar Dimensiones Paralelas** (se ejecutan en paralelo):
   - ETL_Cargar DimCliente
   - ETL_Cargar DimEmpleado
   - ETL_Cargar DimGeografia
   - ETL_Cargar DimProducto
3. **Cargar DimTiempo**: Carga la dimensión de fechas
4. **Cargar FactVentas**: Carga la tabla de hechos con búsquedas de claves

## Detalles de los Componentes

### Limpieza de Tablas DW
- Trunca la tabla `FactVentas` para cada ejecución
- Permite actualizaciones incrementales del DW

### Cargar Dimensiones (Paralelas)
Cada dimensión utiliza:
- **Origen OLE DB**: Lee de tablas de BikeZ
- **Ordenamiento**: Prepara datos para merge join
- **Merge Join**: Compara con datos existentes en BikeZ_DW
- **Split condicional**: Separa registros nuevos de modificados
- **Destino OLE DB**: Inserta nuevos registros
- **OLE DB Command**: Actualiza registros modificados (solo empleados y productos)

### Cargar DimTiempo
- Lee fechas únicas de la tabla `Ventas`
- Calcula atributos de fecha (año, trimestre, mes, día, etc.)
- Inserta registros nuevos (ignora duplicados)

### Cargar FactVentas
- Lee datos de `Ventas` y `DetalleVentas`
- Realiza búsquedas (Lookup) para obtener claves de dimensiones
- Calcula claves de fecha
- Inserta registros en `FactVentas`

## Validación y Monitoreo

### Después de la Ejecución
1. Abra **SQL Server Management Studio**
2. Ejecute consultas de validación:

```sql
-- Contar registros en cada dimensión
SELECT 'DimCliente' AS Tabla, COUNT(*) AS Registros FROM dbo.DimCliente
UNION ALL
SELECT 'DimEmpleado', COUNT(*) FROM dbo.DimEmpleado
UNION ALL
SELECT 'DimGeografia', COUNT(*) FROM dbo.DimGeografia
UNION ALL
SELECT 'DimProducto', COUNT(*) FROM dbo.DimProducto
UNION ALL
SELECT 'DimTiempo', COUNT(*) FROM dbo.DimTiempo
UNION ALL
SELECT 'FactVentas', COUNT(*) FROM dbo.FactVentas;
```

### Ver Errores en la Ejecución
- Revise la pestaña **Progress** en Visual Studio durante la ejecución
- Consulte el **Execution Results** para detalles de errores
- Verifique que las conexiones a las bases de datos sean correctas

## Solución de Problemas

### Error de Conexión
**Problema**: No se puede conectar a las bases de datos
- **Solución**: Verifique los gestores de conexión y las credenciales

### Error de Integridad Referencial
**Problema**: No se pueden insertar datos en `FactVentas`
- **Solución**: Asegúrese de que todas las dimensiones se cargaron correctamente

### El Paquete Se Ejecuta Lentamente
**Problema**: La ejecución tarda demasiado tiempo
- **Solución**: Las dimensiones se ejecutan en paralelo; verifique la disponibilidad de recursos del servidor

## Mantenimiento y Actualizaciones

### Ejecutar Regularmente
Se recomienda ejecutar este paquete de forma programada (diaria, semanal, etc.) para mantener el DW actualizado.

### Crear un Job de SQL Server
1. Abra **SQL Server Agent**
2. Cree un nuevo **Job**
3. En el paso del job, seleccione **SQL Server Integration Services Package**
4. Configure la programación según sea necesario

## Contacto y Soporte

Para preguntas o problemas con el proyecto, contacte al equipo de data warehouse o cree un issue en el repositorio de GitHub.

---

**Última actualización**: Octubre 2025
**Versión**: 1.0
