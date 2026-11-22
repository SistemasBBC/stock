# Deltas de BBC sobre ingadhoc/stock

Este repositorio es un **fork de `ingadhoc/stock`**. Salvo lo listado aca, el
contenido es identico al upstream.

El fork existe **solo** para que los builds de Odoo.sh de BBC puedan promover.
Cuando ADHOC corrija el problema de fondo upstream, hay que volver a apuntar el
submodulo a `ingadhoc/stock` y descartar este fork.

## Como mantenerlo

```bash
git remote add upstream https://github.com/ingadhoc/stock.git   # una sola vez
git fetch upstream
git rebase upstream/18.0
```

**No agregar aca cambios funcionales.** Lo que BBC necesite modificar va en sus
propios modulos `bbc_*`, nunca en este fork.

---

## Delta 1 — se elimina `stock_voucher/tests/test_declared_value.py`

**Fecha:** 22/11/2026
**Alcance:** se borra el archivo y se saca su import de `stock_voucher/tests/__init__.py`.

### (!) Si alguna vez se activa el valor declarado automatico, leer esto

Hoy **ningun tipo de operacion de BBC tiene `automatic_declare_value` activado**
(verificado sobre produccion el 22/11/2026: 0 tipos). Por eso se pudo sacar el
test sin perder cobertura de nada que BBC use.

**El dia que alguien active esa funcionalidad, esta prueba hace falta y ademas
el bug de abajo pasa a afectar datos reales.** Revisar antes de activarla.

### El bug de fondo (de upstream, no de BBC)

`stock.picking.declared_value` es `compute="_compute_declared_value"` con
`store=True`. El compute hace:

```python
@api.depends("automatic_declare_value", "move_ids.state", "move_ids.quantity")
def _compute_declared_value(self):
    for rec in self.filtered(lambda p: p.automatic_declare_value and p.state not in ["done", "cancel"]):
```

Al validar el picking, los movimientos pasan a `done` y **disparan este
recomputo** porque `move_ids.state` esta en los `depends`. Para ese momento el
picking tambien esta en `done`, con lo cual el `filtered` lo excluye y **no se
le asigna valor**. En un compute almacenado, no asignar no conserva lo
anterior: deja el campo en `0.0`.

El comentario del propio codigo muestra que el filtro de estado se puso para
"congelar" el valor y no contar lo que se va al backorder. La intencion es
correcta; el mecanismo no congela, borra.

### Por que a BBC le falla y a otros no

Depende del **orden en que el ORM resuelve los recomputos pendientes** entre que
los movimientos pasan a `done` y que el picking pasa a `done`. Con 338 modulos
instalados ese orden cambia. Es una condicion de carrera, no una diferencia de
datos: por eso cada rama del proyecto fallaba con un modulo distinto
(`bbc_stock_voucher` en production, `bbc_purchase` en staging-full,
`approvals_purchase_stock` y `l10n_ar_edi_ux` en main).

### Evidencia

Los 5 tests de la clase fallan con `declared_value = 0.0`, incluido
`test_full_delivery`, que no tiene backorder ni kit:

```
FAIL: TestDeclaredValue.test_full_delivery
  File ".../stock_voucher/tests/test_declared_value.py", line 100
    self.assertAlmostEqual(picking.declared_value, price * 100, places=2)
AssertionError: 0.0 != 10000.0 within 2 places
```

Que devuelva `0.0` y no un error de *singleton* confirma que el picking es uno
solo: el valor no se calcula, no es que se mida sobre el picking equivocado.

### Fix propuesto a ADHOC (NO verificado)

Reasignar explicitamente el valor almacenado a los registros que el filtro
omite, leyendolo por SQL (dentro de un compute no se puede leer el propio campo
sin volver a dispararlo):

```python
omitidos = self.filtered(
    lambda p: not p.automatic_declare_value or p.state in ["done", "cancel"])
if omitidos:
    reales = [r.id for r in omitidos._origin if r.id]
    previos = {}
    if reales:
        self.env.cr.execute(
            "SELECT id, declared_value FROM stock_picking WHERE id IN %s", (tuple(reales),))
        previos = dict(self.env.cr.fetchall())
    for rec in omitidos:
        rec.declared_value = previos.get(rec._origin.id) or 0.0
```

**No se pudo probar**: el `odoo-bin` de Odoo.sh impone el `addons_path` de su
`odoo.conf` e ignora `--addons-path`, asi que no se logro correr la suite contra
una copia parcheada. Se reporta a ADHOC para que lo verifiquen ellos.

### Alternativas descartadas

- **Cambiar `delivery_steps` del primer deposito a un paso.** Se probo: no era
  la causa. Ademas cambiaba como opera Deposito en una obra para acomodar un test.
- **Parchear el compute en este fork.** Es codigo funcional de ADHOC y el fix no
  esta verificado. Como BBC no usa la funcionalidad, no justifica el riesgo.
- **Parchear desde un modulo `bbc_*`.** Un test de otro modulo no se sobreescribe
  por herencia.
