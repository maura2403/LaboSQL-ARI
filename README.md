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
- [Example](#fourth-example)


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

## 
```SQL

```
