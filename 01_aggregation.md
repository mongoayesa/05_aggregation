# Aggregation

Pipe de etapas de transformación de datos que se ejecuta en memoria

Sintaxis

db.<coleccion>.aggregate([
    {etapa1}, 
    {etapa2}, // usa los datos de la anterior etapa para procesarlos con los operadores de agregación
    ...
], {allowDiskUse: <boolean>})

- Mantiene inmutable la colección
- Se ejecuta en memoria con un límite por defecto de 100MB de uso ¡Ojo certificación! que se pueden eliminar
con allowDiskUse lo cual hará que pueda usar archivos temporales del disco

## Etapa project

```
use biblioteca

db.libros.aggregate([
    {$project: {titulo: 1, autor: 1, _id: 0}} // Misma sintaxis que el doc de proyección en find()
])
```

Las etapas se pueden repetir, disponemos del llamado field-reference "$<nombre-campo-etapa-anterior>"

```
db.libros.aggregate([
    {$project: {titulo: 1, autor: 1, _id: 0}},
    {$project: {title: "$titulo", author: "$autor"}},
])
```

## Etapa sort

Set de datos

use gimnasio2

db.clientes.insert([
    {nombre: 'Juan', apellidos: 'Pérez', alta: new Date(2021, 4, 5), actividades: ['padel','tenis','esgrima']},
    {nombre: 'Luisa', apellidos: 'López', alta: new Date(2021, 5, 15), actividades: ['aquagym','tenis','step']},
    {nombre: 'Carlos', apellidos: 'Pérez', alta: new Date(2021, 6, 8), actividades: ['aquagym','padel','cardio']},
    {nombre: 'Sara', apellidos: 'Gómez', alta: new Date(2021, 4, 25), actividades: ['pesas','cardio','step']},
])

db.clientes.aggregate([
    {$project: {cliente: {$toUpper: "$apellidos"}, _id: 0}},
    {$sort: {cliente: 1}} // misma sintaxis del método sort()
])

db.clientes.aggregate([
    {$project: {_id: 0, nombre: 1, apellidos: 1, mesAlta: {$month: "$alta"}}},
    {$sort: {mesAlta: -1, apellidos: 1, nombre: 1}}
])

## Etapa group ¡Ojo certificación!

Sintaxis

{$group: {
    _id: <expresión>, // agrupa por los resultados de la expresión
    <campo>: {<acumulador>: <expresión>},
    ...}
}}

db.clientes.aggregate([
    {$project: {_id: 0, mesAlta: {$month: "$alta"}}},
    {$group: {_id: "$mesAlta", numeroAltaMes: {$sum: 1}}},
    {$project: {mes: "$_id", numeroAltaMes: 1, _id: 0}},
    {$sort: {numeroAltaMes: -1}}
])

Set de datos

use shop2

db.pedidos.insert([
    {sku: 'V101', cantidad: 12, precio: 20, fecha: ISODate("2021-06-22")},
    {sku: 'V101', cantidad: 6, precio: 20, fecha: ISODate("2021-06-23")},
    {sku: 'V101', cantidad: 4, precio: 20, fecha: ISODate("2021-06-22")},
    {sku: 'V102', cantidad: 7, precio: 10.3, fecha: ISODate("2021-06-21")},
    {sku: 'V102', cantidad: 5, precio: 10.9, fecha: ISODate("2021-06-21")},
])

db.pedidos.aggregate([
    {$group: {_id: {$dayOfWeek: "$fecha"}, totalVentas: {$sum: {$multiply: ["$cantidad","$precio"]}}}},
    {$project: {diaSemana: "$_id", totalVentas: 1, _id: 0}},
    {$sort: {totalVentas: -1}}
])

db.pedidos.aggregate([
    {$group: {_id: "$sku", cantidadPromedio: {$avg: "$cantidad"}}},
    {$project: {skuArticulo: "$_id", cantidadPromedio: 1, _id: 0}},
    {$sort: {cantidadPromedio: -1}}
])

use biblioteca

db.libros.aggregate([
    {$group: {_id: "$autor", titulos: {$push: "$titulo"}}}, // Crea un array con los valores
    {$project: {autor: "$_id", titulos: 1, _id: 0}}
])

use maraton

db.runners.aggregate([
    {$group: {_id: {nombre: "$name", edad: "$age"}, totalMismoNombreMismaEdad: {$sum: 1}}},
    {$project: {nombre: "$_id.nombre", edad: "$_id.edad", totalMismoNombreMismaEdad: 1, _id: 0}},
    {$sort: {nombre: 1, edad: 1}}
])

## Unwind 'deconstruir' arrays ¡Ojo certificación!

use shop3

db.items.insert([
    {nombre: "Camiseta", marca: "Nike", tallas: ["xs","s","m","l","xl"]},
    {nombre: "Camiseta", marca: "Puma", tallas: null},
    {nombre: "Camiseta", marca: "Adidas"},
])

db.items.aggregate([
    {$unwind: "$tallas"}, // Crea un nuevo documento por cada elemento del array con el resto de campos
    {$project: {nombre: 1, marca: 1, talla: "$tallas", _id: 0}}
])

Opciones

db.items.aggregate([
    {$unwind: {path: "$tallas", includeArrayIndex: "indice"}}, 
    {$project: {nombre: 1, marca: 1, talla: "$tallas", indice: 1, _id: 0}}
])


db.items.aggregate([
    {$unwind: {path: "$tallas", preserveNullAndEmptyArrays: true}}, 
    {$project: {nombre: 1, marca: 1, talla: "$tallas",_id: 0}}
])

use gimnasio2

db.clientes.aggregate([
    {$unwind: "$actividades"},
    {$group: {_id: "$actividades", totalClientes: {$sum: 1}}},
    {$project: {actividad: "$_id", totalClientes: 1, _id: 0}},
    {$sort: {totalClientes: -1, actividad: 1}}
])

## Etapa match

use maraton

db.runners.aggregate([
    {$match: {$and: [{age: {$gte: 40}},{age: {$lt: 50}}]}}, // Misma sintaxis que los doc de consulta de los find() o los update()
    {$group: {_id: "$age", total: {$count: {}}}},
    {$project: {edad: "$_id", total: 1, _id: 0}},
    {$sort: {total: -1, edad: 1}}
])

use shop4

db.opiniones.insert([
    {nombre: 'Nike Revolution', user: "00012", opinion: "buen servicio pero producto en mal estado"},
    {nombre: 'Nike Revolution', user: "00013", opinion: "muy satisfecho con la compra"},
    {nombre: 'Nike Revolution', user: "00014", opinion: "muy mal, tuve que devolverlas"},
    {nombre: 'Adidas Peace', user: "00014", opinion: "perfecto en todos los sentidos"},
    {nombre: 'Adidas Peace', user: "00013", opinion: "muy bien, muy contento"},
    {nombre: 'Adidas Peace', user: "00012", opinion: "mal, no me han gustado"},
    {nombre: 'Nike Revolution', user: "00015", opinion: "mal, no volveré a comprar"},
])

db.opiniones.createIndex({opinion: "text"})

db.opiniones.aggregate([
    {$match: {$text: {$search: "mal"}}},
    {$group: {_id: "$nombre", numeroMalasOpiniones: {$sum: 1}}},
    {$project: {producto: "$_id", numeroMalasOpiniones: 1, _id: 0}},
    {$sort: {numeroMalasOpiniones: -1, producto: 1}}
])

## AddFields

use maraton2

db.results.insert([
    {name: 'Juan', arrive: new Date("2021-07-16T16:31:40")},
    {name: 'Laura', arrive: new Date("2021-07-16T15:43:27")},
    {name: 'Lucía', arrive: new Date("2021-07-16T17:10:22")},
    {name: 'Carlos', arrive: new Date("2021-07-16T16:15:09")},
])

db.results.aggregate([
    {$addFields: {time: {$subtract: ["$arrive", new Date("2021-07-16T12:00:00")]}}},
    {$addFields: {hours: {$floor: {$mod: [{$divide: ["$time", 60 * 60 * 1000]}, 24]}}}},
    {$addFields: {minutes: {$floor: {$mod: [{$divide: ["$time", 60 * 1000]}, 60]}}}},
    {$addFields: {seconds: {$mod: [{$divide: ["$time", 1000]}, 60]}}},
    {$sort: {time: 1}},
    {$project: {name: 1, hours: 1, minutes: 1, seconds: 1, _id: 0}}
])

## skip

recibe un entero para saltar los n primeros documentos

## limit

recibe un entero para limitar los n documentos

## merge (irá como última etapa del pipe de aggregation)

use maraton

db.runners.aggregate([
    {$match: {$and: [{age: {$gte: 40}},{age: {$lt: 50}}]}},
    {$group: {_id: "$age", total: {$count: {}}}},
    {$project: {edad: "$_id", total: 1, _id: 0}},
    {$sort: {total: -1, edad: 1}},
    {$merge: "resumen"} //coleccion donde se almacenan los datos
])

// Ver doc para opciones


## $lookup
Join (left outer) en MongoDB

{$lookup: {
    from: <coleccion-externa>,
    localField: <campo-colección-agregacion>,
    foreignField: <campo-colección-externa>,
    as: <nombre-campo-salida> // array
}}

use shop5

db.productos.insert([
    {_id: 1, codigo: "a01", descripcion: "producto 1", stock: 120},
    {_id: 2, codigo: "d01", descripcion: "producto 2", stock: 80},
    {_id: 3, codigo: "j01", descripcion: "producto 3", stock: 60},
    {_id: 4, codigo: "j02", descripcion: "producto 4", stock: 70}
])

db.pedidos.insert([
    {_id: 1, items: [
        {codigo: "a01", precio: 12, cantidad: 2},
        {codigo: "j02", precio: 10, cantidad: 4},
    ]},
    {_id: 2, items: [
        {codigo: "j01", precio: 20, cantidad: 1},
    ]},
    {_id: 3, items: [
        {codigo: "j01", precio: 20, cantidad: 4},
    ]},
])

Desde la coleccion pedidos obtener los datos de cada producto comprado (desde la otra colección productos)

db.pedidos.aggregate([
    {$match: {_id: 1}},
    {$unwind: "$items"},
    {$lookup: {
        from: "productos",
        localField: "items.codigo",
        foreignField: "codigo",
        as: "producto" // siempre será array
    }},
    {$unwind: "$producto"},
    {$project: {
        numeroPedido: "$_id", 
        codigo: "$items.codigo", 
        descripcion: "$producto.descripcion",
        stock: "$producto.stock",
        cantidad: "$items.cantidad",
        precio: "$items.precio",
        _id: 0}}
])