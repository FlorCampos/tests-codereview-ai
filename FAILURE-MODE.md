# Modos de falla — `amounts.py`

Inventario de modos de falla de `src/payments_svc/amounts.py`, agrupados por frontera, equivalencia, null/vacío y contrato de negocio. No es una suite de tests: es el mapa de qué comportamientos están confirmados como intencionales y cuáles dependen de una decisión de negocio todavía no tomada. Donde el contrato no se puede inferir del código, se marca "pendiente de decisión" en vez de asumir que el comportamiento actual es la especificación.

## Frontera

### FM-01
- **Categoria:** Frontera
- **Riesgo:** Medio
- **Entrada que lo desestima:** `amount = Decimal("100000.00")` vs `Decimal("100000.01")`
- **Comportamiento actual observado en el código:** `validate_amount` usa `amount > MAX_AMOUNT` (amounts.py:64), así que el límite superior es inclusivo (100000.00 pasa, 100000.01 no).
- **Contrato esperado recomendado:** igual al actual.
- **Estado del contrato:** confirmado — la comparación estricta es explícita y deliberada.
- **Por que importa para pagos:** define el techo exacto de transacción permitido; un off-by-one aquí cambiaría silenciosamente cuántos pagos se rechazan en el límite superior.

### FM-02
- **Categoria:** Frontera
- **Riesgo:** Bajo
- **Entrada que lo desestima:** monto donde `amount * FEE_RATES[currency] == MINIMUM_FEES[currency]` exactamente (p. ej. USD ≈ 10.34)
- **Comportamiento actual observado en el código:** `max(percentage_fee, MINIMUM_FEES[currency])` (amounts.py:80) resuelve el empate sin ambigüedad porque ambos valores son iguales.
- **Contrato esperado recomendado:** igual al actual.
- **Estado del contrato:** confirmado.
- **Por que importa para pagos:** el punto de cruce entre "aplica mínimo" y "aplica porcentaje" es donde más fácil se introduce un bug de redondeo; aquí no lo hay, pero es el primer lugar a revisar si cambian las tasas.

### FM-03
- **Categoria:** Frontera
- **Riesgo:** Alto
- **Entrada que lo desestima:** `amount = Decimal("0.00")`
- **Comportamiento actual observado en el código:** `validate_amount` permite 0 (solo rechaza `< 0`), y `calculate_fee` tiene un caso especial que retorna `Decimal("0.00")` de comisión cuando `amount == 0` (amounts.py:76-77), saltándose el `MINIMUM_FEES` por completo.
- **Contrato esperado recomendado:** no inferible del código — no hay comentario ni prueba de negocio que indique si un pago de $0 es una operación válida o si debería rechazarse aguas arriba, ni si la comisión mínima debe aplicar incluso a monto cero.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** si $0 no debería ser un "pago" real, este código lo procesa silenciosamente como válido y sin comisión, lo que puede ocultar errores de integración (p. ej. un campo de monto no poblado) en lugar de rechazarlos.

## Equivalencia

### FM-04
- **Categoria:** Equivalencia
- **Riesgo:** Alto
- **Entrada que lo desestima:** `raw = 19.99 * 3` (float) u otro resultado de aritmética binaria antes de llegar a `parse_amount`
- **Comportamiento actual observado en el código:** la firma acepta `float` explícitamente (amounts.py:31) y hace `Decimal(str(raw).strip())` (amounts.py:36); si el float ya trae error de representación binaria (p. ej. `59.96999999999999`), ese error queda congelado como Decimal "exacto".
- **Contrato esperado recomendado:** no inferible del código — no hay indicación de si `float` es un tipo de entrada soportado deliberadamente para montos de dinero o solo quedó en la firma por comodidad.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** aceptar `float` como clase de equivalencia válida para dinero es un riesgo clásico de precisión; si es intencional debería documentarse, y si no, debería rechazarse explícitamente en `parse_amount`.

### FM-05
- **Categoria:** Equivalencia
- **Riesgo:** Bajo
- **Entrada que lo desestima:** `Decimal("-0.00")`
- **Comportamiento actual observado en el código:** `amount < Decimal("0")` es `False` para `-0.00` (igualdad de valor en Decimal), y `amount == Decimal("0")` también es `True`, así que cae en la misma rama que el cero "normal" en amounts.py:76.
- **Contrato esperado recomendado:** igual al actual (tratar -0.00 como 0).
- **Estado del contrato:** confirmado — es consecuencia directa y correcta de la semántica de `Decimal`, no un bug.
- **Por que importa para pagos:** bajo riesgo real, pero vale documentarlo porque un desarrollador nuevo podría asumir que `-0.00` dispara la rama de "negativo rechazado".

### FM-06
- **Categoria:** Equivalencia
- **Riesgo:** Medio
- **Entrada que lo desestima:** `raw = "10.123456"` (más de 2 decimales)
- **Comportamiento actual observado en el código:** `parse_amount` no valida la escala/precisión del número, solo que sea finito (amounts.py:40-41); `validate_amount` tampoco la revisa. El redondeo a centavos (`round_money`) solo se aplica al final, sobre la comisión y el total — el "amount" original circula con precisión sub-centavo durante todo el flujo.
- **Contrato esperado recomendado:** no inferible del código — no está claro si un monto debe truncarse/rechazarse a 2 decimales en el momento del parseo, o si es aceptable que el monto "crudo" tenga más precisión mientras la comisión y el total sí se redondeen.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** si el "amount" original se persiste o se envía a un procesador con precisión sub-centavo, puede generar descuadres contables entre lo que el usuario autorizó y lo que efectivamente se registra.

### FM-07
- **Categoria:** Equivalencia
- **Riesgo:** Bajo
- **Entrada que lo desestima:** `currency = " usd "`, `"Usd"`, `"USD"`
- **Comportamiento actual observado en el código:** `normalize_currency` hace `.strip().upper()` antes de comparar contra `FEE_RATES` (amounts.py:50), tratando estas variantes como equivalentes.
- **Contrato esperado recomendado:** igual al actual.
- **Estado del contrato:** confirmado — normalización explícita e intencional.
- **Por que importa para pagos:** asegura que variaciones de formato del cliente (mayúsculas/espacios) no generen rechazos espurios de un pago por lo demás válido.

## Null / vacío

### FM-08
- **Categoria:** Null/vacío
- **Riesgo:** Bajo
- **Entrada que lo desestima:** `raw = None`
- **Comportamiento actual observado en el código:** chequeo explícito `if raw is None: raise AmountError("amount is required")` (amounts.py:32-33).
- **Contrato esperado recomendado:** igual al actual.
- **Estado del contrato:** confirmado.
- **Por que importa para pagos:** un monto ausente debe fallar con un mensaje claro de "requerido", no con un error genérico — y aquí así ocurre.

### FM-09
- **Categoria:** Null/vacío
- **Riesgo:** Medio
- **Entrada que lo desestima:** `raw = ""` o `raw = "   "`
- **Comportamiento actual observado en el código:** no hay chequeo de cadena vacía antes del `Decimal(...)`; `""` y `"   "` llegan a `Decimal(str(raw).strip())` (amounts.py:36), fallan con `InvalidOperation` y terminan devolviendo `AmountError("amount must be numeric")` — el mismo caso de "no me diste un monto" que `None` reporta con un mensaje distinto ("amount is required").
- **Contrato esperado recomendado:** no inferible del código si el mensaje debe unificarse (tratar `""`/`"   "` como "amount is required" igual que `None`) — es una decisión de UX de error, no solo de validación.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** mensajes de error inconsistentes para la misma clase de problema (ausencia de monto) complican el manejo de errores aguas arriba (p. ej. un cliente que distingue "campo vacío" de "formato inválido" para mostrar mensajes distintos al usuario final).

### FM-10
- **Categoria:** Null/vacío
- **Riesgo:** Bajo
- **Entrada que lo desestima:** `currency = None`, `""`, o `"   "`
- **Comportamiento actual observado en el código:** los tres casos colapsan consistentemente en `CurrencyError("currency is required")` (amounts.py:47-52).
- **Contrato esperado recomendado:** igual al actual.
- **Estado del contrato:** confirmado — a diferencia de `amount`, aquí sí está unificado.
- **Por que importa para pagos:** consistencia correcta; sirve de referencia de cómo debería resolverse FM-09 para `amount` si se decide unificar.

## Contrato de negocio

### FM-11
- **Categoria:** Contrato de negocio
- **Riesgo:** Alto
- **Entrada que lo desestima:** llamar `calculate_fee("10.00", "USD")` o `total_with_fee` con un `amount` que no sea `Decimal` (p. ej. se saltó `parse_amount`)
- **Comportamiento actual observado en el código:** `validate_amount` asume que `amount` ya es un `Decimal` (amounts.py:60-65) sin verificarlo; el type hint no se aplica en runtime. `amount < Decimal("0")` con un `str` lanza `TypeError`, no `AmountError`.
- **Contrato esperado recomendado:** no inferible completamente del código si el contrato pretendido es "el caller SIEMPRE debe pasar por `parse_amount` primero" (y por tanto el `TypeError` es aceptable como error de programación) o si `validate_amount`/`calculate_fee` deberían ser defensivos y convertir cualquier entrada no válida en `AmountError`.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** el módulo declara `AmountError`/`CurrencyError` como la familia de errores de dominio esperada; una ruta que en cambio deja escapar un `TypeError` sin capturar puede tumbar un endpoint o saltarse el manejo de errores diseñado para pagos.

### FM-12
- **Categoria:** Contrato de negocio
- **Riesgo:** Medio
- **Entrada que lo desestima:** agregar una moneda nueva solo a `FEE_RATES` sin agregarla también a `MINIMUM_FEES` (o viceversa)
- **Comportamiento actual observado en el código:** `normalize_currency` solo valida membresía contra `FEE_RATES` (amounts.py:54); `calculate_fee` indexa `MINIMUM_FEES[currency]` por separado (amounts.py:80). Hoy ambos diccionarios tienen las mismas tres llaves, así que no falla, pero nada impone que sigan sincronizados.
- **Contrato esperado recomendado:** no inferible del código — no hay mecanismo (ni test, ni assert, ni estructura unificada) que garantice que toda moneda soportada tenga tasa Y mínimo definidos.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** si en el futuro alguien agrega una moneda de forma incompleta, el fallo no sería un `CurrencyError` controlado sino un `KeyError` sin capturar dentro de `calculate_fee`, rompiendo el contrato de errores del módulo justo en el cálculo de comisión.

### FM-13
- **Categoria:** Contrato de negocio
- **Riesgo:** Medio
- **Entrada que lo desestima:** cualquier monto que dispare redondeo con dígito exactamente en 5 (p. ej. `Decimal("10.005")`)
- **Comportamiento actual observado en el código:** `round_money` usa `ROUND_HALF_EVEN` ("redondeo bancario") (amounts.py:68-69) en lugar de `ROUND_HALF_UP`.
- **Contrato esperado recomendado:** no inferible del código — muchos sistemas de pago/regulaciones exigen redondeo half-up para efectivo/comisiones, pero no hay ningún comentario o referencia normativa que confirme que half-even es la elección deliberada para este sistema.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** el modo de redondeo en cálculos de comisión es, en algunos marcos regulatorios o de conciliación con procesadores, una regla de negocio explícita y auditable — no un detalle técnico libre; vale la pena confirmarlo con quien define las reglas de comisión antes de asumir que el código actual es la especificación.

### FM-14
- **Categoria:** Contrato de negocio
- **Riesgo:** Medio
- **Entrada que lo desestima:** `amount = "100.50"`, `currency = "COP"`
- **Comportamiento actual observado en el código:** COP se trata igual que USD/EUR, con `CENT = Decimal("0.01")` (amounts.py:7) aplicado de forma global vía `round_money`, permitiendo centavos de COP.
- **Contrato esperado recomendado:** no inferible del código — no hay indicación de si el sistema debe tratar COP como moneda con subunidades de 2 decimales (como aquí) o como moneda de unidad entera, que es una convención común en procesadores de pago para COP.
- **Estado del contrato:** pendiente de decisión.
- **Por que importa para pagos:** si el procesador o el regulador local espera montos COP sin decimales, aceptar "100.50" COP como válido podría generar un monto que ningún backend real procesa correctamente, descubierto solo en producción.
