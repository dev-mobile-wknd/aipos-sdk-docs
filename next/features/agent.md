# Cashier agent

The cashier agent accepts commands in everyday language and runs the transaction
itself: searching for products, managing the cart, processing payment, and
composing the receipt. The agent's language understanding targets Indonesian,
matching the SDK's primary market.

```
Cashier: cari iPhone 15, tambah 1, berapa totalnya?
         (find iPhone 15, add 1, what's the total?)
Agent:   iPhone 15 128GB sudah masuk keranjang. Totalnya Rp 13.999.000. Lanjut bayar?
         (iPhone 15 128GB is in the cart. Total is Rp 13,999,000. Proceed to pay?)
Cashier: bayar (pay)
Agent:   Silakan tempelkan kartu ke terminal...
         (Please tap a card on the terminal...)
Agent:   Pembayaran berhasil (VISA). Ini struknya: ...
         (Payment successful (VISA). Here's the receipt: ...)
```

**Available on:** `AIPosSDK` (artifact `aipos-sdk`) only.

## Sending a message

```kotlin
scope.launch {
    val answer: String = sdk.sendMessage("find MacBook Air then add 1")
    // answer comes back in Indonesian
}
```

`sendMessage` is a `suspend` function that only returns once the agent is done —
including after waiting for the customer to tap a card. Don't await it on the UI
thread without an indicator; observe `observeProcessing()` to show an animation.

## Displaying the conversation

```kotlin
sdk.observeMessages().collect { history ->
    history.forEach { message ->
        when (message) {
            is AgentMessage.UserMessage -> rightBubble(message.content)
            is AgentMessage.AssistantMessage -> leftBubble(message.content)
            is AgentMessage.ToolCall -> smallNote("${message.toolName}: ${message.summary}")
            is AgentMessage.ErrorMessage -> errorBubble(message.message)
        }
    }
}

sdk.observeProcessing().collect { busy -> showTypingIndicator(busy) }
```

| `AgentMessage` type | Contents |
|---|---|
| `UserMessage` | The message sent by the cashier (`content`) |
| `AssistantMessage` | The agent's reply (`content`) |
| `ToolCall` | A record that the agent ran an action (`toolName`, `summary`) — useful for audits |
| `ErrorMessage` | A failure while processing the message (`message`) |

Every type has a `timestamp` in epoch milliseconds.

## Payment status while the agent is paying

The agent only replies once the transaction is done. While waiting, show the tap
instructions from `observePaymentState()`:

```kotlin
sdk.observePaymentState().collect { state ->
    when (state) {
        is PaymentState.WaitingTap -> showTapDialog(state.message)
        is PaymentState.Processing -> showTapDialog(state.message)
        else -> dismissTapDialog()
    }
}
```

## What the agent can do

| Action | Example command |
|---|---|
| `search_product` — search for a product by name or barcode | *"ada Samsung S24?"* (any Samsung S24?), *"cek 8991234500011"* (check 8991234500011) |
| `manage_cart` — add, remove, change quantity, view, clear | *"tambah 2"* (add 2), *"hapus Pixel-nya"* (remove the Pixel), *"berapa totalnya?"* (what's the total?) |
| `process_payment` — contactless card payment | *"bayar"* (pay), *"lanjut"* (proceed) |
| `generate_receipt` — compose a 40-character-wide text receipt | *"cetak struknya"* (print the receipt) |
| `transaction_history` — recent transaction history | *"transaksi hari ini apa saja?"* (what transactions happened today?) |

A cart changed by the agent is the same cart changed through `addToCart()` — the
on-screen buttons and chat commands can be used interchangeably.

## Built-in safeguards

- **Never pays without confirmation.** The agent is required to show the total
  first and wait for the cashier to answer *"bayar"* (pay), *"lanjut"* (proceed),
  or *"ok"*. The payment action refuses to run without that confirmation, without
  touching the terminal.
- **Never invents products, prices, or stock.** Every number comes from your
  catalog.
- **Stock is validated** with the same rules as `addToCart()`.

## Starting a new transaction

```kotlin
scope.launch { sdk.resetSession() }
```

`resetSession()` clears the conversation history, suggestions, cart, and payment
status all at once. Call it every time the customer changes.

## Limitations

- Payment through the agent only supports **contactless cards** — not available on
  iOS. Online payment is triggered from an on-screen button via
  `startOnlinePayment()`.
- Requires internet and a valid language-model key.
- Every message calls the language model one or more times, incurring cost
  according to your model provider's rates.
