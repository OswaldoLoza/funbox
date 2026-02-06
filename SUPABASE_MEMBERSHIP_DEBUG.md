# Diagnóstico directo con tu `MembershipsPage` (ya con contexto real)

Gracias, este snippet sí muestra mejor dónde puede salir el `HTTP 400`.

## 1) Qué sí hace `MembershipsPage` y qué NO

En este archivo:

- Sí consulta Supabase: `membership_types` (`select`, `eq`, `order`).
- Sí dispara checkout para clases sueltas con `initializeCheckout`.
- **NO** aparece la inserción directa de membresía (`.from('memberships').insert(...)`) en este componente.

Eso implica que el `400` de membresías probablemente viene de:

1. lógica interna de `MembershipCard`, o
2. backend llamado por `MembershipCard`, o
3. webhook/post-checkout que crea membresía, o
4. contrato de checkout para productos de membresía.

---

## 2) Hallazgos clave en este snippet

### A. Carga de planes desde `membership_types`

```js
const { data, error } = await supabase
  .from('membership_types')
  .select('*')
  .eq('is_active', true)
  .order('price', { ascending: true });
```

Esto no debería producir `400` salvo esquema/policy mal configurado.

### B. Compra rápida usa `variant_id` (clases, no membresías)

```js
const checkoutItems = [{
  variant_id: productId,
  quantity: 1,
  metadata: { description, type: 'single_class' }
}];
```

Si esos IDs de clase funcionan y membresías no, entonces el problema puede ser:

- `variant_id` de membresía incorrecto en `MembershipCard`, o
- metadata requerida faltante para membresías, o
- backend que diferencia `single_class` vs `membership` y valida campos distintos.

### C. `MembershipCard` es la pieza crítica ausente

Aquí renderizas:

```jsx
<MembershipCard
  key={plan.id}
  plan={plan}
  isRenewal={isRenewal}
  isLoggedIn={isAuthenticated}
/>
```

Sin ver ese componente, no se puede confirmar el payload real de membresías.

---

## 3) Hipótesis más probable (con esta evidencia)

El flujo de membresía falla por **payload inválido en checkout o creación posterior** (webhook/backend), no por el render de `MembershipsPage`.

Especialmente probable si:

- Tienda y clases sueltas sí cobran bien.
- Solo falla cuando compras membresía.

Eso apunta a validaciones específicas de membresía (plan_id, user_id, fechas, tipo, duración, estado, etc.).

---

## 4) Cambios mínimos recomendados en `MembershipsPage`

Aunque no sea raíz, puedes mejorar depuración aquí:

1. En `handleQuickPurchase`, loguear payload y respuesta de error estructurada.
2. Asegurar `setProcessingProduct(null)` en `finally` para evitar estado trabado.
3. Mostrar `error.message` en toast para diagnóstico.

### Ejemplo robusto de `handleQuickPurchase`

```js
const handleQuickPurchase = async (productId, description) => {
  if (!isAuthenticated) {
    window.location.href = '/login?returnUrl=/memberships';
    return;
  }

  setProcessingProduct(productId);

  try {
    if (!productId) throw new Error('Producto inválido para checkout');

    const checkoutItems = [{
      variant_id: productId,
      quantity: 1,
      metadata: {
        description,
        type: 'single_class',
      },
    }];

    const response = await initializeCheckout({
      items: checkoutItems,
      successUrl: `${window.location.origin}/checkout-success?type=class`,
      cancelUrl: window.location.href,
    });

    if (!response?.url) throw new Error('No URL returned from checkout');

    window.location.href = response.url;
  } catch (error) {
    console.error('Quick purchase error details:', {
      message: error?.message,
      code: error?.code,
      details: error?.details,
      cause: error?.cause,
      productId,
      description,
    });

    toast({
      title: 'Error',
      description: error?.message || 'No se pudo iniciar el pago.',
      variant: 'destructive',
    });
  } finally {
    setProcessingProduct(null);
  }
};
```

---

## 5) El fix real depende de `MembershipCard` + backend/webhook

Para arreglar tu `400` de forma exacta necesito ver:

1. Código de `MembershipCard` (donde inicia pago de membresía).
2. Código del endpoint/backend o función que procesa checkout de membresía.
3. Si usas webhook para activar membresía, ese handler.
4. Response JSON exacto del `400` en Network (request + response body).
5. SQL de `memberships` y `membership_types` (constraints y RLS policies).

Con eso sí te doy parche final preciso (sin suposiciones).
