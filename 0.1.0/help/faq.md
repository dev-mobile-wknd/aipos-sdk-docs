# FAQ

## General

### Does the SDK provide ready-made UI?

No. The SDK only provides logic, data, and the transaction flow. All UI belongs to
your app, so the design can match your store's brand.

### Do I have to use every feature?

No. Choose the artifact that matches your needs — see [Choosing an artifact](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/choosing-artifacts).

### Can the SDK run without internet?

Cart and card payment don't need internet on the SDK's side (the MineSec terminal
has its own network requirements). Product suggestions, the cashier agent, and
online payment need internet.

### Does the SDK conflict with Koin or Hilt in my app?

No. The SDK uses its own Koin container, not a global `startKoin`.

## Data

### Where is cart and transaction data stored?

In local device storage (`SharedPreferences` on Android, `NSUserDefaults` on
iOS). The cart survives the app being closed. The SDK never sends transactions
to any server — send them to your own backend yourself if needed.

### Does the SDK reduce stock after a transaction?

No. The SDK only validates against the `stock` number you provide. See
[Product catalog](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/catalog#stock-stays-your-responsibility).

### What currency is supported?

Rupiah. `Money.format()` produces `"Rp 25,000"`, and online payment only accepts
whole-Rupiah amounts.

## AI features

### What language model is used?

OpenAI GPT-5.4 mini, either directly through the OpenAI API or a compatible
proxy.

### What data is sent to the language-model provider?

For product suggestions: the conversation transcript and catalog data (name,
category, price, description, stock). For the cashier agent: the cashier's
messages, product search results, and cart contents. No payment card data is
ever sent.

### What language is understood?

Indonesian, including the Indonesian–English mix common in stores. Replies and
suggestion reasoning are written in Indonesian.

### Does the SDK record audio?

No. The SDK never accesses the microphone. Your app records audio and converts
it to text.

## Payment

### Which cards does the terminal accept?

Visa, Mastercard, American Express, JCB, Maestro, and UnionPay — depending on the
acquiring configuration in your MineSec profile.

### Can an iPhone accept card payment?

Not yet. See [Introduction](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/introduction#platform-support). Use online
payment on iOS instead.

### How do I try payment without real money?

- **Card:** the simulator variant approves every transaction.
- **Online:** prototype mode (`baseUrl = ""`) or the built-in beta environment.

### Why do I have to verify online payment with a backend?

The paid status is read from the payment page on the cashier's device, not from
a server. See [Online payment](https://dev-mobile-wknd.github.io/aipos-sdk-docs/features/online-payment#verifying-payment-on-the-backend).

### Can the cashier choose the channel (VA or QRIS) before the page opens?

No need. The customer chooses it themselves on the payment page. You can only
restrict which channels are available via `allowedPaymentMethods`.
