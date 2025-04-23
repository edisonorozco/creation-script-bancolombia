# Consultas NoSQL para Sistema Bancario en MongoDB

A continuación se presentan 5 enunciados de consultas basados en las colecciones NoSQL del sistema bancario, junto con las soluciones utilizando operaciones avanzadas de MongoDB.

## 1. Análisis de Saldos por Tipo de Cuenta

**Enunciado:** El departamento financiero necesita un informe que muestre el saldo total, promedio, máximo y mínimo por cada tipo de cuenta (ahorro y corriente) para evaluar la distribución de fondos en el banco.

**Consulta MongoDB:**
```javascript
    db.clientes.aggregate([
      { $unwind: "$cuentas" },
      { $group: {
          _id: "$cuentas.tipo_cuenta",
          saldo_total: { $sum: "$cuentas.saldo" },
          saldo_promedio: { $avg: "$cuentas.saldo" },
          saldo_maximo: { $max: "$cuentas.saldo" },
          saldo_minimo: { $min: "$cuentas.saldo" }
      }},
      { $project: {
          _id: 0,
          tipo_cuenta: "$_id",
          saldo_total: 1,
          saldo_promedio: 1,
          saldo_maximo: 1,
          saldo_minimo: 1
      }}
    ]);
```

## 2. Patrones de Transacciones por Cliente

**Enunciado:** El equipo de análisis de comportamiento necesita identificar los patrones de transacciones de cada cliente, mostrando la cantidad y el monto total de transacciones por tipo (depósito, retiro, transferencia) para cada cliente.

**Consulta MongoDB:**
```javascript
    db.transacciones.aggregate([
      { $group: {
          _id: { cliente: "$cliente_ref", tipo: "$tipo_transaccion" },
          cantidad: { $sum: 1 },
          monto_total:{ $sum: "$monto" }
      }},
      { $group: {
          _id: "$_id.cliente",
          patrones: { 
            $push: {
              tipo: "$_id.tipo",
              cantidad: "$cantidad",
              monto_total:"$monto_total"
            }
          }
      }},
      { $lookup: {
          from: "clientes",
          localField: "_id",
          foreignField: "_id",
          as: "cliente"
      }},
      { $unwind: "$cliente" },
      { $project: {
          _id: 0,
          id_cliente: "$cliente._id",
          nombre: "$cliente.nombre",
          cedula: "$cliente.cedula",
          patrones: 1
      }}
    ]);
```

## 3. Clientes con Múltiples Tarjetas de Crédito

**Enunciado:** El departamento de riesgo crediticio necesita identificar a los clientes que poseen más de una tarjeta de crédito, mostrando sus datos personales, cantidad de tarjetas y el detalle de cada una.

**Consulta MongoDB:**
```javascript
    db.clientes.aggregate([
      { $unwind: "$cuentas" },
      { $unwind: "$cuentas.tarjetas" },
      { $match: { "cuentas.tarjetas.tipo_tarjeta": "credito" } },
      { $group: {
          _id: {
            id:     "$_id",
            nombre: "$nombre",
            cedula: "$cedula"
          },
          cantidad: { $sum: 1 },
          tarjetas: { $push: "$cuentas.tarjetas" }
      }},
      { $match: { cantidad: { $gt: 1 } } },
      { $project: {
          _id:       0,
          id_cliente:"$_id.id",
          nombre:    "$_id.nombre",
          cedula:    "$_id.cedula",
          cantidad:  1,
          tarjetas:  1
      }}
    ]);
```

## 4. Análisis de Medios de Pago más Utilizados

**Enunciado:** El departamento de marketing necesita conocer cuáles son los medios de pago más utilizados para depósitos, agrupados por mes, para orientar sus campañas promocionales.

**Consulta MongoDB:**
```javascript
    db.transacciones.aggregate([
      { $match: { tipo_transaccion: "deposito" } },
      { $addFields: {
          mes: { $dateToString: { format: "%Y-%m", date: "$fecha" } }
      }},
      { $group: {
          _id: { mes: "$mes", medio: "$detalles_deposito.medio_pago" },
          usos: { $sum: 1 }
      }},
      { $group: {
          _id: "$_id.mes",
          medios: { 
            $push: {
              medio: "$_id.medio",
              usos:  "$usos"
            }
          }
      }},
      { $sort: { "_id": 1 } },
      { $project: {
          _id: 0,
          mes:    "$_id",
          medios: 1
      }}
    ]);
```

## 5. Detección de Cuentas con Transacciones Sospechosas

**Enunciado:** El departamento de seguridad necesita identificar cuentas con patrones de transacciones sospechosas, definidas como aquellas que tienen más de 3 retiros en un mismo día con un monto total superior a 1,000,000 COP.

**Consulta MongoDB:**
```javascript
    db.transacciones.aggregate([
      { $match: { tipo_transaccion: "retiro" } },
      { $addFields: {
          dia: { $dateToString: { format: "%Y-%m-%d", date: "$fecha" } }
      }},
      { $group: {
          _id: { cuenta: "$num_cuenta", dia: "$dia" },
          retiros_count: { $sum: 1 },
          monto_total:   { $sum: "$monto" }
      }},
      { $match: {
          retiros_count: { $gt: 3 },
          monto_total:   { $gt: 1000000 }
      }},
      { $group: {
          _id: "$_id.cuenta",
          dias_sospechosos: {
            $push: {
              fecha: "$_id.dia",
              count: "$retiros_count",
              total: "$monto_total"
            }
          }
      }},
      { $project: {
          _id: 0,
          num_cuenta: "$_id",
          dias_sospechosos:  1
      }}
    ]);
```