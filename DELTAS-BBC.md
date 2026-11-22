# Deltas de BBC sobre ingadhoc/stock

Este repositorio es un **fork de `ingadhoc/stock`**. Salvo lo listado aca, el
contenido es identico al upstream.

El fork existe **solo** para que los builds de Odoo.sh de BBC puedan promover.
Cuando ADHOC tome el cambio upstream, hay que volver a apuntar el submodulo a
`ingadhoc/stock` y descartar este fork.

## Como mantenerlo

```bash
git remote add upstream https://github.com/ingadhoc/stock.git   # una sola vez
git fetch upstream
git rebase upstream/18.0
```

El delta es de una linea mas comentarios, asi que el rebase no deberia tener
conflictos. **No agregar aca cambios funcionales**: lo que BBC necesita
modificar va en sus propios modulos `bbc_*`, nunca en este fork.

---

## Delta 1 — `stock_voucher/tests/test_declared_value.py`

**Fecha:** 22/11/2026
**Reportado a ADHOC:** si (ver issue en ingadhoc/stock)

### El problema

`TestDeclaredValue.setUpClass` toma el primer deposito de la compania:

```python
cls.warehouse = cls.env["stock.warehouse"].search([("company_id", "=", cls.env.company.id)], limit=1)
cls.warehouse.out_type_id.automatic_declare_value = True
```

y despues mide sobre `order.picking_ids` **como si fuera un unico picking**:

```python
picking = order.picking_ids
...
self.assertAlmostEqual(picking.declared_value, price * 30, places=2)
```

Eso vale solo si el deposito entrega en **un paso**. Si `delivery_steps` es
`pick_ship` o `pick_pack_ship`, el pedido genera ademas el picking interno, la
medicion cae sobre el equivocado y `declared_value` da **0.0**.

### Donde se manifesto

Produccion de BBC: **27 depositos** en la compania 1, y el primero por orden
(`0716.CC.LOS ESPINILLOS`) esta configurado en **`pick_ship`**. Los 5 tests de
la clase fallaban con `declared_value = 0.0`, lo que impedia que Odoo.sh
promoviera cualquier build de produccion.

En una instalacion con un solo deposito en un paso el problema es invisible.

### El cambio

Una linea en `setUpClass`, para que el test no dependa de como tenga
configurados sus depositos quien lo corre:

```python
cls.warehouse.delivery_steps = "ship_only"
```

### Por que asi y no de otra forma

- **No se cambio la configuracion del deposito en produccion.** Poner
  `0716.CC.LOS ESPINILLOS` en un paso hacia pasar el test, pero cambia como
  opera Deposito en esa obra para acomodar una prueba.
- **No se borro el archivo de tests.** Se pierde cobertura y el problema
  reaparece en el proximo cliente multi-deposito.
- **No se parcheo desde un modulo `bbc_*`.** Un test de otro modulo no se
  sobreescribe por herencia.
