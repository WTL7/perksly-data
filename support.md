---
title: Perksly Support
---

# Perksly Support

Welcome! Perksly is built and maintained by an independent developer. Here are answers to common questions and ways to get help.

## Frequently Asked Questions

### How do I add a credit card?

Tap the **+** button in the top-right of the **My Wallet** tab. Search the built-in catalog (200+ U.S. cards) and tap one to add it. If your card isn't in the catalog, tap **Add Your Own Card** at the bottom of the list to create a custom card.

### What's the difference between Benefits and Perks?

- **Benefits** are credits and rewards that reset on a schedule — monthly Uber credits, quarterly hotel credits, annual airline fee credits, free-night certificates, etc. Perksly tracks how much you've used in the current period and reminds you before they expire.
- **Perks** are always-on features that don't expire — lounge access, elite status, cellphone insurance, no foreign transaction fees, concierge service. Perksly lists them on each card so you remember what you have.

### Why isn't my card in the catalog?

The built-in catalog covers most major U.S. consumer credit cards from American Express, Chase, Capital One, Citi, Bank of America, Wells Fargo, Discover, U.S. Bank, and others. New cards launch all the time and we update the catalog regularly.

If your card is missing:
1. Add it as a custom card via **Add Your Own Card**, OR
2. Open an issue at the [perksly-data repository](https://github.com/WTL7/perksly-data/issues) requesting it be added, OR
3. Email [tanning.impacts.5v@icloud.com](mailto:tanning.impacts.5v@icloud.com) with the card name and issuer.

### I'm not getting reminder notifications

Three things to check:

1. **Notification permission.** On your iPhone: Settings → Perksly → Notifications → make sure **Allow Notifications** is on.
2. **Reminder is enabled on the benefit.** Open the card → tap the **bell icon** on the benefit → set a reminder duration (1, 7, or 30 days before).
3. **The benefit isn't already fully used.** Reminders only fire for benefits that still have value left in the current period.

If notifications still don't arrive, force-quit and reopen Perksly — the next foreground re-arms all pending reminders.

### Where is my data stored?

Entirely on your device. Perksly does not have any servers that store user data. Your cards, benefits, and usage history are saved locally using Apple's SwiftData framework and are included in your standard iOS device backup (iCloud Backup or encrypted Mac/PC backup) only if you have those features enabled.

To delete all your Perksly data, simply delete the app from your device.

See the full [Privacy Policy](privacy.html) for details.

### Will Perksly sync between my iPhone and iPad?

Not in the current version. iCloud sync via CloudKit is planned for a future release. For now, Perksly is single-device.

### Is there an Android version?

No. Perksly is iOS only.

### How do I delete a card or benefit?

- **Delete a card:** in My Wallet, swipe left on the card row → tap Delete. This removes the card and all of its benefits and perks.
- **Delete a benefit:** open the card → swipe left on the benefit row → tap Delete.
- **Delete a perk:** open the card → swipe left on the perk row → tap Delete.

Deletes are permanent. There's currently no undo.

### How do I edit a benefit's value or reset rule?

Open the card → swipe right on the benefit row → tap **Edit**. You can change the name, value, frequency, reset rule, and category.

### Why does my catalog show updates available?

When the public card catalog gets new cards or updated benefit values, Perksly checks for updates and offers to apply them to cards you've already added. You can review the changes and tap **Apply** to accept them, or **Skip** to keep your current version.

### How do I request a feature or report a bug?

Email [tanning.impacts.5v@icloud.com](mailto:tanning.impacts.5v@icloud.com) or open an issue at the [perksly-data repository](https://github.com/WTL7/perksly-data/issues).

If you're a TestFlight beta tester, you can also send feedback (including screenshots) directly via the TestFlight app.

## Contact

- **Email:** [tanning.impacts.5v@icloud.com](mailto:tanning.impacts.5v@icloud.com)
- **GitHub Issues:** [perksly-data repository](https://github.com/WTL7/perksly-data/issues)

This is a one-person project, so response times may vary, but every message is read.
