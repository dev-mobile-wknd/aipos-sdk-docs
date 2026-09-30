# Product catalog

The SDK **doesn't ship any product data**. Your app hands over the catalog through
`ProductCatalogSource`, and the SDK only reads it to:

- draw the product list via `observeCatalog()`,
- search for products when the cashier agent receives a command,
- build product suggestions — only items in the catalog may be suggested,
- validate stock when an item is added to the cart.

## The contract

```kotlin
fun interface ProductCatalogSource {
    fun observeProducts(): Flow<List<Product>>
}
```

It's shaped as a stream. Changes on your side — stock going down, a price going up,
a new product — are visible to the SDK immediately, with no need to rebuild anything.

## Option 1: `MutableProductCatalog`

For a static catalog, or if you'd rather push data:

```kotlin
val catalog = MutableProductCatalog(initialProducts)

val sdk = AIPosSDK.Builder()
    .productCatalog(catalog)
    // ...
    .build()

// Whenever the data changes, e.g. after syncing with the server:
catalog.setProducts(latestProducts)
```

`setProducts` **replaces the entire contents** of the catalog, it doesn't append.
Products missing from the new list immediately disappear from search and
suggestions. Safe to call from any thread.

## Option 2: your own implementation

If your catalog is already a stream — Room, SQLDelight, or a `StateFlow` in a
repository — connect it directly so there's no second copy that can go stale:

```kotlin
val catalog = ProductCatalogSource {
    productDao.observeAll().map { rows -> rows.map { it.toAiposProduct() } }
}

private fun ProductEntity.toAiposProduct() = Product(
    id = ProductId(sku),
    name = name,
    price = Money.fromRupiah(priceRupiah),
    barcode = barcode.orEmpty(),
    category = category,
    stock = stock,
    description = shortDescription.orEmpty(),
    imageUrl = imageUrl.orEmpty(),
)
```

:::warning[Two requirements for a custom implementation]
1. **The flow must emit a first value promptly** — an empty list is fine. The SDK
   waits for the first emission when adding an item to the cart; a flow that never
   emits leaves the cashier waiting forever.
2. **The flow must be cheap to collect.** The SDK re-collects it every time it needs
   the catalog. Use a `StateFlow`, a database flow, or a cache — don't hit the
   network on every collection.
:::

## The `Product` model

| Field | Type | Description |
|---|---|---|
| `id` | `ProductId` | Unique identity. May not be empty. |
| `name` | `String` | Display name. Used by the agent when searching for products. |
| `price` | `Money` | Unit price. Create it with `Money.fromRupiah(...)`. |
| `barcode` | `String` | Barcode (EAN-13), for lookup via a scanner. |
| `category` | `String` | E.g. `"Smartphone"` or `"Laptop"`. |
| `stock` | `Int` | Remaining stock. Zero means out of stock and can't be added to the cart. |
| `description` | `String` | A one-line summary. **Strongly affects the quality of AI suggestions** — fill it with the product's selling points. |
| `imageUrl` | `String` | Image address. The SDK never loads it; display it with your own image loader. |

:::tip[A good description makes for good suggestions]
Product suggestions match customer needs against `name`, `category`, `price`, and
`description`. A description like *"48MP camera, 2-day battery, IP68 water
resistance"* is far more useful than *"Nice phone"*.
:::

## Stock stays your responsibility

The SDK validates requests against the `stock` number you provide, but it **never
reduces stock**. After a transaction is paid, record the sale and update stock in
your own system. Since the catalog is a stream, the new number is picked up by the
SDK immediately:

```kotlin
sdk.processPayment().collect { state ->
    if (state is PaymentState.Success) {
        val sold = sdk.observeTransactions(limit = 1).first().first().cart
        inventory.reduceStock(sold.items)   // your own app's code
    }
}
```
