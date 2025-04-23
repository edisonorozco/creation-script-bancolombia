# Consultas SQL para Base de Datos Bancaria

## Enunciado 1: Clientes con múltiples cuentas y sus saldos totales

**Necesidad:** El banco necesita identificar a los clientes que tienen más de una cuenta, mostrar cuántas cuentas tiene cada uno y el saldo total acumulado en todas sus cuentas, ordenados por saldo total de mayor a menor.

**Consulta SQL:**
```sql
    SELECT 
        c.id_cliente,
        c.nombre,
        COUNT(ct.num_cuenta) AS numero_cuentas,
        SUM(ct.saldo) AS saldo_total
    FROM Cliente c
    JOIN Cuenta ct ON c.id_cliente = ct.id_cliente
    GROUP BY c.id_cliente, c.nombre
    HAVING COUNT(ct.num_cuenta) > 1
    ORDER BY saldo_total DESC;
```

## Enunciado 2: Comparativa entre depósitos y retiros por cliente

**Necesidad:** El departamento de análisis financiero necesita comparar los montos totales de depósitos y retiros realizados por cada cliente, para identificar patrones de comportamiento financiero.

**Consulta SQL:**
```sql
    SELECT
        c.id_cliente,
        c.nombre,
        COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'deposito' THEN t.monto END), 0) AS total_depositos,
        COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'retiro' THEN t.monto END), 0) AS total_retiros
    FROM Cliente c
    JOIN Cuenta ct ON c.id_cliente = ct.id_cliente
    JOIN Transaccion t ON ct.num_cuenta   = t.num_cuenta
    GROUP BY c.id_cliente, c.nombre
    ORDER BY c.id_cliente;
```

## Enunciado 3: Cuentas sin tarjetas asociadas

**Necesidad:** El departamento de tarjetas necesita identificar todas las cuentas que no tienen tarjetas asociadas para ofrecer productos específicos a estos clientes.

**Consulta SQL:**
```sql
    SELECT
        ct.num_cuenta,
        ct.id_cliente,
        ct.saldo
    FROM Cuenta ct
    LEFT JOIN Tarjeta ta ON ct.num_cuenta = ta.num_cuenta
    WHERE ta.num_cuenta IS NULL;
```

## Enunciado 4: Análisis de saldos promedio por tipo de cuenta y comportamiento transaccional

**Necesidad:** La gerencia necesita un análisis comparativo del saldo promedio entre cuentas de ahorro y corriente, pero solo considerando aquellas cuentas que han tenido al menos una transacción en los últimos 30 días.

**Consulta SQL:**
```sql
    WITH cuentas_recientes AS (
      SELECT DISTINCT num_cuenta
      FROM Transaccion
      WHERE fecha > now() - INTERVAL '30 days'
    )
    SELECT
      c.tipo_cuenta,
      ROUND(AVG(c.saldo), 2) AS saldo_promedio
    FROM Cuenta c
    INNER JOIN cuentas_recientes cr
      ON c.num_cuenta = cr.num_cuenta
    GROUP BY c.tipo_cuenta;
```

## Enunciado 5: Clientes con transferencias pero sin retiros en cajeros

**Necesidad:** El equipo de marketing digital necesita identificar a los clientes que utilizan transferencias pero no realizan retiros por cajeros automáticos, para dirigir campañas de banca digital.

**Consulta SQL:**
```sql
    SELECT
        c.id_cliente,
        c.nombre
    FROM Cliente c
    JOIN Cuenta ct ON c.id_cliente = ct.id_cliente
    GROUP BY c.id_cliente, c.nombre
    HAVING 
        COUNT(
          DISTINCT CASE 
            WHEN EXISTS (
              SELECT 1 FROM Transaccion t1
              JOIN Transferencia tr ON t1.id_transaccion = tr.id_transaccion
              WHERE t1.num_cuenta = ct.num_cuenta
            ) THEN ct.num_cuenta
          END
        ) > 0
      AND 
        COUNT(
          DISTINCT CASE 
            WHEN EXISTS (
              SELECT 1 FROM Transaccion t2
              JOIN Retiro r ON t2.id_transaccion = r.id_transaccion
              WHERE t2.num_cuenta = ct.num_cuenta
                AND r.canal = 'cajero'
            ) THEN ct.num_cuenta
          END
        ) = 0;
```