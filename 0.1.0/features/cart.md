# Cart

Cart functionality is available on `AIPosSDK` and `PaymentClient`. Its contents are
stored in local device storage, so they survive the app being closed.

:::warning[`OnlinePaymentClient` doesn't have cart functionality yet]
As of version 0.1.0, `OnlinePaymentClient` charges the cart contents but
doesn't provide `addToCart` and friends, so `startSession()` will fail with
*"Keranjang masih kosong"* (cart is still empty). For online payment, use
`AIPosSDK` from `aipos-sdk`.
:::

## Observing the cart contents

```kotlin
sdk.observeCart().collect { cart ->
    showTotal(cart.total.format())            // "Rp 23.498.000"
    showItemCount(cart.itemCount)             // total quantity, not the number of rows
    cart.items.forEach { item ->
        println("${item.product.name} x${item.quantity} = ${item.subtotal.format()}")
    }
}
```

## Changing the cart contents

Every operation is a `suspend` function that returns `PosResult<Cart>`:

```kotlin
scope.launch {
    when (val result = sdk.addToCart(product.id, quantity = 1)) {
        is PosResult.Success -> Unit                        // observeCart() updates too
        is PosResult.Failure -> showMessage(result.message)  // safe to show to the cashier
    }
}
```

| Operation | Function | Fails when |
|---|---|---|
| Add | `addToCart(productId, quantity = 1)` | Quantity ≤ 0, product not in the catalog, out of stock, or the total exceeds stock |
| Update quantity | `updateCartQuantity(productId, quantity)` | Product not in the cart, or the quantity exceeds stock. Quantity ≤ 0 removes the row. |
| Remove | `removeFromCart(productId)` | Product not in the cart |
| Clear | `clearCart()` | — |

The SDK's failure messages are in Indonesian and are safe to show directly to the
cashier:

- `iPhone 15 128GB sedang habis` (iPhone 15 128GB is out of stock)
- `Stok Google Pixel 8 tidak cukup. Tersedia 5, diminta 6` (Not enough stock for Google Pixel 8. 5 available, 6 requested)
- `Produk tidak ada di keranjang` (Product not in the cart)

:::tip[A concise way to handle the result]
```kotlin
sdk.addToCart(product.id)
    .onSuccess { cart -> haptic() }
    .onFailure { failure -> snackbar(failure.message) }
```
:::

## When the cart is cleared automatically

| Event | Cart |
|---|---|
| Card payment `PaymentState.Success` | Cleared, after the transaction is saved to history |
| Online payment `WebPaymentStatus.Success` | Cleared (once per order) |
| Payment failed or cancelled | **Not** cleared — the customer can try again |
| `resetSession()` on `AIPosSDK` | Cleared, along with the conversation and suggestions |

## The `Cart` model

| Property | Type | Description |
|---|---|---|
| `items` | `List<CartItem>` | Line items, order preserved |
| `subtotal` | `Money` | Sum of all lines before discount |
| `discount` | `Money` | Discount applied to the whole cart |
| `total` | `Money` | `subtotal − discount`, never negative |
| `itemCount` | `Int` | Total quantity across all products |
| `isEmpty` | `Boolean` | `true` when there are no items |

`CartItem` contains `product`, `quantity`, and `subtotal` (price × quantity).

`Cart` is *immutable*: every change produces a new object, so it's safe to use
directly as UI state.
