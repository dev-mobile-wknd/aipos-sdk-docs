# Production checklist

Go through this list before the app is installed on a real cashier device.

## Key security

- [ ] **No OpenAI key in the app.** Use [`llmProxyBaseUrl`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/llm-proxy).
- [ ] **The payment backend password isn't bundled.** `OnlinePaymentConfig.password`
      ends up bundled in the APK/IPA and can't be revoked per device. Fetch it from
      the server when the device is activated and store it in the Android Keystore
      / iOS Keychain.
- [ ] **The payment `apiKey` differs per device**, so a lost device can be revoked
      without disrupting other cashiers.
- [ ] MineSec credentials only live in `~/.gradle/gradle.properties` or CI secrets.

## Card payment

- [ ] A test transaction on the release build does **not** show a `[SIMULATED]`
      message, and `transactionId` doesn't start with `SIM-`.
- [ ] A production MineSec license is in `src/main/assets/` and its name is passed
      to `initPlatform`. Don't use `public-test.license`.
- [ ] `MerchantInfo.profileId` is set to a production profile, not
      `DEFAULT_PROFILE_ID`.
- [ ] `terminalId` and `mid` match the data from the acquirer.
- [ ] A failed `initPlatform` is shown to the cashier, not just logged.

## Online payment

- [ ] `baseUrl`, `email`, `password`, and `apiKey` point at **your production
      backend**, not the beta default.
- [ ] `loopbackUrlReplacement = ""` — the default redirects `localhost` addresses
      to a prototype page.
- [ ] **The paid status is confirmed with your backend** before goods are handed
      over. `WebPaymentStatus.Success` is read from the cashier's device and
      hasn't been verified by a server.
- [ ] **The `POST /v1/transactions/simulate-paid` endpoint isn't active on your
      production backend.** SDK version 0.1.0 calls it about three seconds
      after the `Pending` status, for testing purposes. If that endpoint exists
      and accepts the call, a transaction can be recorded as paid without a real
      payment.
- [ ] `allowedPaymentMethods` matches the channels your store actually accepts.
- [ ] The provider's status codes for failure and expiry have been tested.
      Unrecognized codes show up as `Unknown` — watch the `AiPos/online-payment`
      log during the first week.

## AI features

- [ ] The store's privacy policy discloses that conversations are sent to a
      language-model provider.
- [ ] The LLM proxy has a rate limit and cost monitoring.
- [ ] `Product.description` is filled in — suggestion quality depends heavily on
      it.

## Data and stock

- [ ] Stock is reduced in your own system after a transaction is paid. The SDK
      never reduces it.
- [ ] Transactions are sent to your backend. History in the SDK is only stored on
      the device.
- [ ] `resetSession()` is called every time the customer changes.

## Stability

- [ ] Only one SDK instance per process.
- [ ] `close()` is called once the SDK is no longer needed.
- [ ] Every `PaymentListenerRegistration` and `FlowSubscription` is cancelled when
      the screen is closed.
- [ ] The payment WebView is created once per session (inside `remember` in
      Compose).
