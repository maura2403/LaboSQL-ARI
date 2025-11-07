# LaboSQL-ARI
Laboratorio de SQL y Optimizacion de ARI

# Table of Contents
- [Utiles para recordar](#utiles-para-recordar)
- [Obtener las playlists que tienen el mayor numero de tracks de Rock - Tabla TEMP](#playlists-que-tienen-el-mayor-número-de-tracks-de-rock)
  - [Alternativa con Having](#playlists-que-tienen-el-mayor-número-de-tracks-de-rock-alternativa-having)
- [Clientes que han gastado más que el promedio de todos los clientes en sus compras - con TOP 7 porque si](#clientes-que-han-gastado-más-que-el-promedio-de-todos-los-clientes-en-sus-compras-con-top-7-porque-si)
- [Clientes que hayan comprado canciones de más de un género musical distinto - Count(distinct)](#clientes-que-hayan-comprado-canciones-de-más-de-un-género-musical-distinto)
- [Títulos de los álbumes donde todas las canciones tienen una duración mayor al promedio de duración de todas las canciones de la base - NOT IN](#títulos-de-los-álbumes-donde-todas-las-canciones-tienen-una-duración-mayor-al-promedio-de-duración-de-todas-las-canciones-de-la-base)
- [Ejemplo Datepart](#ejemplo-datepart)
- [Datos de género incluyendo la cantidad de álbumes de cada uno (cantidad de álbumes donde hay canciones de ese género)](#datos-de-género-incluyendo-la-cantidad-de-álbumes-de-cada-uno-cantidad-de-álbumes-donde-hay-canciones-de-ese-género)
- [Optimización](#optimización)
- [Juntas](#juntas)


# Utiles para recordar
Ver info de campos de una tabla facilmente
```SQL
EXEC dbo.sp_help 'dbo.Playlist'
```

## Playlists que tienen el mayor número de tracks de Rock
[**⬆️**](#table-of-contents)
```SQL
-- uso de tabla temporal puede servir
WITH PlaylistRock AS (
SELECT pt.PlaylistId, COUNT(*) AS CantidadTracks
FROM PlaylistTrack pt
JOIN Track t ON pt.TrackId = t.TrackId
JOIN Genre g ON g.GenreId = t.GenreId
WHERE g.Name LIKE 'Rock'
GROUP BY pt.PlaylistId
)
SELECT p.PlaylistId, p.Name, r.CantidadTracks
FROM Playlist p
JOIN PlaylistRock r ON r.PlaylistId = p.PlaylistId
WHERE r.CantidadTracks >= (SELECT MAX(r.CantidadTracks) FROM
PlaylistRock r);
```

## Playlists que tienen el mayor número de tracks de Rock (Alternativa Having)
[**⬆️**](#table-of-contents)
```SQL
SELECT p.PlaylistId, p.Name, COUNT(t.TrackId) as RockCount FROM Playlist as p
INNER JOIN PlaylistTrack as pt ON p.PlaylistId = pt.PlaylistId
INNER JOIN Track as t ON pt.TrackId = t.TrackId
INNER JOIN Genre as g ON t.GenreId = g.GenreId
WHERE g.Name = 'ROCK'
GROUP BY p.PlaylistId, p.Name
HAVING COUNT(t.TrackId) >= (
    SELECT TOP 1 COUNT(t.TrackId) as RockMax FROM Playlist as p
    INNER JOIN PlaylistTrack as pt ON p.PlaylistId = pt.PlaylistId
    INNER JOIN Track as t ON pt.TrackId = t.TrackId
    INNER JOIN Genre as g ON t.GenreId = g.GenreId
    WHERE g.Name = 'ROCK'
    GROUP BY p.PlaylistId
);
```

## Clientes que han gastado más que el promedio de todos los clientes en sus compras con TOP 7 porque si
[**⬆️**](#table-of-contents)
```SQL
SELECT [TOP 7]
c.FirstName,
c.LastName,
SUM(i.Total) as TotalGastado
FROM dbo.Invoice i
INNER JOIN dbo.Customer c ON c.CustomerId = i.CustomerId
GROUP BY c.CustomerId, c.FirstName, c.LastName
HAVING SUM(i.Total) > (
    SELECT AVG(SumaTotal) FROM 
    (SELECT SUM(Total) as SumaTotal 
    FROM dbo.Invoice GROUP BY CustomerId) t)
ORDER BY TotalGastado desc
-- saltea primeras 0 filas y devuelve las siguientes 7
OFFSET 0 ROWS FETCH FIRST 7 ROWS ONLY;
;
```

Existen varias formas de anidar una consulta en el WHERE:
- **WHERE valor = (SELECT…)** → solo funciona si la subconsulta devuelve una columna con un valor
- WHERE valor **IN | NOT IN** (SELECT…) → funciona si la subconsulta devuelve sólo 1 columna o si “valor” es una tupla y la consulta devuelve un tupla similar.
- WHERE valor >= **ALL | SOME** (SELECT…) → funciona si la subconsulta devuelve sólo 1 columna
- WHERE valor **EXISTS | NOT EXISTS** (SELECT…) → funciona siempre

▨ Cuando la consulta anidada utiliza columnas de una fila de alguna tabla de la consulta principal, se dice que las consultas de adentro y la de afuera están **correlacionadas**.
- Esto implica que la consulta de adentro deberá ejecutarse una vez por cada fila de la consulta de afuera.
```SQL
SELECT 
a.AlbumId
FROM Album a 
INNER JOIN Track t ON t.AlbumId = a.AlbumId
GROUP BY a.AlbumId
HAVING COUNT(t.TrackId) > 10;

-- lo mismo consulta anidada o correlacionada
SELECT 
*
FROM Album a 
WHERE (SELECT COUNT(*) FROM Track t WHERE t.AlbumId = a.AlbumId) > 10;
```

## Clientes que hayan comprado canciones de más de un género musical distinto
[**⬆️**](#table-of-contents)
```SQL
SELECT
c.FirstName,
c.LastName,
COUNT(distinct t.GenreId) as CantidadGeneros
FROM dbo.Invoice i 
INNER JOIN dbo.InvoiceLine il ON i.InvoiceId = il.InvoiceId
INNER JOIN dbo.Track t ON il.TrackId = t.TrackId
INNER JOIN dbo.Customer c ON i.CustomerId = c.CustomerId
GROUP BY c.CustomerId, c.FirstName, c.LastName
HAVING COUNT(distinct t.GenreId) > 1
ORDER BY CantidadGeneros DESC
;
```

## Títulos de los álbumes donde todas las canciones tienen una duración mayor al promedio de duración de todas las canciones de la base
[**⬆️**](#table-of-contents)
```SQL
SELECT
a.Title
FROM Album a
WHERE a.AlbumId NOT IN (SELECT  -- se usa el NOT IN
    a.AlbumId
    FROM dbo.Track t 
    INNER JOIN dbo.Album a ON t.AlbumId = a.AlbumId
    WHERE t.Milliseconds <= (SELECT AVG(Milliseconds) FROM Track))
ORDER BY a.Title
;
-- o seleccionar todos y sacar aquellos que tienen una cancion con menos duracion
SELECT DISTINCT
a.Title
FROM dbo.Album t 
EXCEPT 
SELECT DISTINCT
a.Title
FROM dbo.Track t 
INNER JOIN dbo.Album a ON t.AlbumId = a.AlbumId
WHERE t.Milliseconds <= (SELECT AVG(Milliseconds) FROM Track)
ORDER BY a.Title
;
```

## Ejemplo Datepart
[**⬆️**](#table-of-contents)
```SQL
WHERE DATEPART(year,i.InvoiceDate) > 2010
```

## Datos de género incluyendo la cantidad de álbumes de cada uno (cantidad de álbumes donde hay canciones de ese género)
[**⬆️**](#table-of-contents)
```SQL
SELECT
g.GenreId,
g.Name,
COUNT(DISTINCT a.AlbumId) as CantAlbumes
FROM Genre g 
LEFT JOIN Track T ON g.GenreId = t.GenreId
INNER JOIN Album a ON t.AlbumId = a.AlbumId
GROUP BY g.GenreId, g.Name;
```

# Optimización
[**⬆️**](#table-of-contents)

Tabla como:
- Heap Table --> sin clustered index, SQL Server usa el IAM para localizar las paginas, procesa por orden de asignacion (en que se reservaron los extents)
- Árbol B --> si tiene Clustered Index definido

Puede ser:
- Primary Key --> Clustered Index a no ser qaue se especifique NONCLUSTERED
- Unique Index --> Unclustered index a menos que se especifique CLUSTERED

Cada página ocupa 8KB --> grupos de 8 páginas son un extent

SQL Server usa páginas en el buffer pool (caché).

LOB Root Structure --> para tipos de datos grandes (VARBINARY(MAX), VARCHAR(MAX), TEXT, etc.), apunta a Small o Large LOB Data

Prefijos:
- PK_ : Primary Key, Clustered, Único
- AK_ : Unclustered, Único
- IX_ : Unclustered, NO Único


Clustered Index --> indica orden físico de los datos, la tabla en sí, solo uno por tabla

Non-Clustered Index --> orden que es almacenado en estructura separada de los datos, el resto está en Heap Table, con al dirección física, este árbol tiene referencias a los datos.

Scan --> revisa todo el índice

Seek --> busca un valor/rango en el cluster (hay un where generalmente)

RID Lookup/Key Lookup --> buscar todos los datos de X y hacer lookup de todos los datos de ese X

Heap Table --> Table Scan porque no hay índice clustered

Clustered Index Scan --> Unordered u Ordered


Stream Aggregate y Hash Aggregate: para GROUP BY, DISTINCT y funciones de agregación

- Stream Aggregate --> funcion de agregado sin GROUP BY, si tiene group by tienen que ya venir ordenados (Sort antes o ya de por sí).
    - Ej: SELECT MAX(campo) FROM tabla
- Hash Aggregate: como Hash Join, usa **Hash Match**, para tablas grandes **no ordenadas**, pro no hace falta ordenarlos, su cardinalidad se estima en solo unos pocos grupos
- Distinct Sort: cuando hay Distinct se puede usar este o uno de los dos anteriores :p

## Juntas
[**⬆️**](#table-of-contents)

Ver orden y algoritmo de junta

Join tiene Outer input (tabla principal que va escaneando) e Inner input (por cada elemento del outer input busca los de la otra tabla inner input que corresponda)

3 Tipos:
- Nested Loop Join: dos loops anidados
    - Sirve cuando outer input es chico e inner input es grande **indexado**
    - Amplio, cualquier tipo de junta, **los otros dos necesitan una igualdad si o si en la condición**
- Merge Join: va mezclando a medida que llegan, ya tienen que **estar ordenados**
    - Usa ordenamiento previo, por eso suele venir de clustered index scan
- Hash Join: tabla de hash
    - util para entradas GRANDES NO ORDENADAS, no indexadas eficientemente


- Compute Scalar: resuelve operaciones como concatenacion, cambiar tipo de dato, una modificacion sobre los datos que se devuelven.

- Filter: dejo pasar lo que convenga

- Concatenation: Cuando hay una Unión.
    - es más costoso Union (devuelve todo menos duplicados, usa un hash match para sacarlos) o Union ALL (deja duplicados, entonces es menos costoso, devuelve directo resultado de la concatenacion)?

- Integridad referencial: 
    - SELECT a.campo, **b.id** FROM a JOIN B on a.id = b.id WHERE a.campo = valor --> hace Clustered Index Seek (por el where) en ambas tablas y después un Nested Loop

    - SELECT a.campo, **a.id** FROM a JOIN B on a.id = b.id WHERE a.campo = valor --> hace Clustered Index Seek (por el where) solo en tabla a, porque referencia a una clave del otro lado y como es Clustered es NOT NULL y **Foreign Key**, por integridad referencial no tiene que buscarlo

- Parametros:
    - Importa la cardinalidad de lo que buscas con el where:
        - where City = 'Mentor' --> hay pocos registros, es más fácil hacer index scan en el unclustered (IX_) y después Key Lookup en el clustered (PK_)
        - where City = 'London' --> hay muchos más registros, le conviene hacer solo Clustered Index Scan (PK_) en vez de ir al más chico para ir yendo y viniendo por cada registro
        - SET @City = 'Mentor'; SELECT (...) WHERE City = @City --> si antes ejecutaste la de London, queda guardado ese plan en vez de usar el de Mentor, no es el mejor plan para este caso --> **Parameter Sniffing**
        - OPTION( OPTIMIZE FOR (@City = 'Mentor'))

- Argumentos: SARGable o no - Son buscables o no
    - where field in (a,b,c,d) --> Cluster Seek con 4 predicados y 4 scans, no sabe si la lista esta ordenada o no, busca cada uno por separado
    - where field between a and d --> Cluster Seek pero hay un solo predicado y un solo scan, ya ordenados, es el campo del índice
    - where field  = value/2 --> Seek, hace la cuenta y busca el que convenga
    - where field * 2 = value --> hace Scan porque no puede buscar porque involucra al campo

- Covertura del índice: 
    - where ID = 42 --> busca en el indice uncluster que tiene ese id y busca en otra tabla el dato que pida (si es otro que no es id ni está en el indice)
    - 

## Ejemplos:

(...) where index_field = "123" --> Index Seek

(...) where index_field = 123 --> Index Scan --> el campo es nvarchar, tiene que hacer la conversion al final de la query

(...) count(\<campo no nulleable\>) --> index scan porque cantidad que hay es la cantidad de registros que hay en la tabla, cuenta cualquier indice (el más chico, algun smallint, etc.) en vez de toda la tabla, traerse de memoria la menor cantidad de cosas

(...) count(\<nulleable\>) --> Clustered index scan porque cuenta solo los que no son null, tiene que ir fila por fila revisando eso (la claster tiene la info en sí)

(...) count(*) --> usa un compute scalar porque tiene que convertirlo a bigint

(...) COUNT_BIG(*) --> no hace compute scalar porque no tiene que convertirlo

(...) where field like '1%' --> index seek + key lookup

(...) select no_en_indice where field like '1%1 --> clustered index scan (porque por la cantidad de registros igual tiene que buscar el no_en_indice)

(...) where field NOT like '1%' --> index scan

Si la **selectividad** es muy baja, no gano mucho usando el indice (ej: ver genero (M,F) solo separa en 2 la cantidad a revisar)

(...) distinct(\<campo con pocos distintos\>) --> Clustered index scan + Hash Match (ya sabe que son pocos valores, compara con eso) (ej: Card Type)

(...) distinct(\<campo con muchos distintos\>) --> Index Scan (AK_) (ej: Card Number)