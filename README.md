# The Spring: restaurant ordering system

PHP 8.1+ and MySQL/MariaDB, Bootstrap 5, Bootstrap Icons, Chart.js. No Composer needed.

## What it does
- **Customers:** browse the menu (offers shown), order for dine-in or delivery, choose arrival or delivery time, pay the upfront share by M-Pesa STK Push, track the order live, get notifications, pay the balance by STK Push, download receipts, set order reminders.
- **Waiters:** see and move orders through Placed, Preparing, Ready, Dispatched, Completed; request balances by STK Push; search payments; see the balances table.
- **Admins:** everything above plus menu, prices and offers, team accounts, settings (deposit %, M-Pesa and SMS credentials), SMS log, dashboard.

## Requirements
PHP 8.1 or newer with `pdo_mysql`, `curl`, `mbstring`, `openssl`, `fileinfo`; MySQL 5.7+ or MariaDB 10.4+; HTTPS in production (Safaricom only calls back to HTTPS URLs).

## Install
1. Create an empty database and user (cPanel: MySQL Databases).
2. Upload the project. Point the domain or subdomain **document root to the `public/` folder**. Everything else stays outside the web root.
3. Copy `config/config.sample.php` to `config/config.php`. Fill in the database details, set a long random `app_key`, and set `app_url` to your live address (for example `https://orders.thespring.co.ke`).
4. Make `storage/` and `public/uploads/` writable (chmod 775).
5. Open `https://your-site/install.php`, create the admin account, then **delete `public/install.php`**.
6. Sign in, open **Admin, Settings** and fill in the three tabs (below).
7. Add a cron job so reminders fire even when nobody is online: `* * * * * php /path/to/the-spring/bin/cron.php`

Local testing (XAMPP/Laragon or the PHP built-in server): `php -S localhost:8000 -t public`

## Where to put the credentials (Admin, Settings)
**M-Pesa tab** (Safaricom Daraja, M-Pesa Express):
- Environment: *Simulation* (default, no real money, payments auto-succeed after a few seconds), *Sandbox*, or *Production*
- Consumer key, consumer secret, Lipa na M-Pesa passkey, Paybill number (business shortcode)
- Press **Save and test connection** to check the key and secret
- The callback URL is generated for you and shown on the page; each request carries it with a secret token. No registration on the Daraja portal is needed.

**SMS tab** (choose Africa's Talking or SMS Leopard):
- Africa's Talking: username, API key, optional sender ID, Sandbox or Live
- SMS Leopard: API key and API secret (dashboard, Settings, API Keys) and your assigned sender ID (use `SMS_Leopard` for testing until you have one)
- Master switch, per-event switches (order confirmed, ready, dispatched, balance) and editable message texts
- **Save and send test** sends a message to any number

Secrets are encrypted in the database with your `app_key`. Do not change `app_key` after saving credentials, or you will need to re-enter them.

## How payments work
1. Customer places an order. It is saved as *Awaiting deposit* (the kitchen does not see it yet).
2. STK Push is sent for the upfront share (default 30%, change under Settings, General).
3. Safaricom calls back (or the app checks the status after 15 seconds). The payment is recorded once, the order becomes *Placed*, the customer and staff are notified, and a receipt is created.
4. When the order is *Ready* or *Dispatched*, the customer (or a waiter) requests the balance by STK Push. An order cannot be closed until the balance is paid.
All attempts, including failed and cancelled ones, are kept under **Payments, All M-Pesa requests**.

## Search and balances
- **Payments:** search by M-Pesa code, receipt number, phone (07xx or 2547xx), order number, customer name or amount; filter by type and date; export CSV.
- **Balances:** every order that still owes money, with totals, search, filters, an M-Pesa request button and an SMS reminder button; export CSV.

## Roles
| Role | Access |
|---|---|
| Customer | menu, own orders, payments, receipts, reminders, notifications |
| Waiter | orders, balances, payments (search), notifications |
| Admin | everything, including menu, offers, team, settings, SMS log |

## Folder map
`public/` web root (pages, `api/`, `admin/`, `assets/`, `uploads/`) · `app/` classes (Mpesa, Sms, Orders, Reminders, Settings, Auth) · `database/schema.sql` · `bin/cron.php` · `config/` · `storage/` logs and token cache

## Security notes
Prepared statements everywhere, CSRF tokens on every POST, bcrypt passwords, login throttling, encrypted secrets, upload validation, callback token check, role checks on every staff page. Keep PHP and the server updated, use HTTPS, and delete `install.php`.

## Sign in and sign up with Google
Customers can use **Sign in with Google** / **Sign up with Google** on the sign-in and register pages. New Google users get an account straight away with no phone number step. The phone is asked for at checkout (where the M-Pesa prompt needs it) and can be added in **My profile**; until then the account's phone is empty and shows as "no phone yet" in Team and customers. SMS reminders are skipped for accounts without a phone. If a Google email matches an existing account, the two are linked automatically (Google only supplies verified emails). Google accounts have no password until the person sets one in **My profile**.

Set up (Admin, Settings, **Google** tab, which also shows the exact values to copy):
1. Google Cloud Console, create a project, then **OAuth consent screen** (External). Add the **Privacy Policy** and **Terms of Service** links (`/privacy.php`, `/terms.php`) and **Publish** the app.
2. **Credentials, Create credentials, OAuth client ID, Web application**. Under *Authorised redirect URIs* add `https://your-site/google_callback.php` (set `app_url` in `config/config.php` so the address is right; Google requires https, except localhost).
3. Paste the client ID and secret into the Google tab and switch the button on. The secret is stored encrypted.

The flow uses the OAuth authorization-code method with PKCE, a state and nonce check, and verifies the token's issuer, audience, expiry and `email_verified` claim. Sign-ins, links and sign-ups are written to the audit trail. The database upgrades itself (schema version 6 adds `users.google_id` and `users.password_set`).

## Order this again, saved addresses and loyalty points
**Order this again:** every order in *My orders* and on its own page has an **Order this again** button. It puts the same dishes back in the cart at today's prices (quantities never go below what is already in the cart; unavailable dishes are listed and skipped) and opens checkout with the same type, party size, address, landmark, pin and notes filled in.

**Saved delivery addresses** (*My profile* or the account menu, **Delivery addresses**): customers keep up to 10 addresses, each with a label, address, landmark and an optional **map pin** (the "Pin my current location" button uses the phone's location, with the person's permission; it needs https). One address is the default. At checkout, choosing Delivery shows the saved addresses as options (the default is preselected), or "Another address" with a tick-box to save it. The pin travels with the order, and staff see an **Open the customer's pin in Maps** button on the order page (a plain Google Maps link, no API key needed).

**Loyalty points** (off until you switch it on under **Admin, Settings, Loyalty**):
- Earn: when an order is **completed** (so paid in full), points = food total after any discount, per KES 100 (default 2 points per KES 100). The delivery fee does not earn points.
- Use: once a customer has the minimum balance (default 100, shown as a progress bar on their **Rewards** page), a tick-box at checkout uses their points as a discount on the food bill (default KES 1 per point, capped at 30% of the food bill, never more than 50%). The discount is part of the order total, so the deposit, balance, receipts and payments all follow the discounted total, and receipts show a "Loyalty discount" line.
- Safety: points are a ledger (`loyalty_ledger`); the balance is its sum. Redeeming locks the customer's row so the same points cannot be spent twice, earning and returning are idempotent, points used on an order that is cancelled come back, and a refund on a completed order takes back the matching points.
- Admin: **Loyalty points** in the side menu lists customers with balances and lets an admin add or remove points with a reason (audit-logged, and the customer is notified). Settings, Loyalty shows how many points are outstanding and what they are worth.
- The database upgrades itself (schema version 8: `saved_addresses`, `loyalty_ledger`, and `lat`, `lng`, `loyalty_points`, `loyalty_discount` on orders). The Terms page shows a Loyalty points section whenever the programme is on.

## Fingerprint sign-in (passkeys)
People can sign in with their phone's fingerprint, face or screen lock instead of typing a password. It uses the WebAuthn standard: the device keeps the private key and only signs after the person unlocks it, and this server stores just the public key (table `passkeys`, added automatically by schema version 7). No library is needed.
- **Turning it on:** a signed-in person opens **My profile, Fingerprint sign-in, Turn on for this device**. Customers and the admin dashboard also see a dismissible "Sign in faster" banner until they have set it up. People can add several devices and remove any of them.
- **Signing in:** the sign-in page shows **Sign in with fingerprint** on devices that support it. No email is typed: the phone offers the saved passkey for this site. They land on the page they were heading to.
- **Admin switch:** Settings, **Security**, *Allow fingerprint, face and screen-lock sign-in*. It also shows how many accounts use it.
- **Requirements:** HTTPS and a real domain name (not an IP address). A passkey only works on the domain it was created on, so if you change the domain, people add their fingerprint again. Set `app_url` in `config/config.php` to your live address. Works on current Chrome, Safari, Edge and Samsung Internet on phones with a screen lock or biometric.
- **Safety:** each sign-in uses a one-time challenge, checks the site address, requires the fingerprint/PIN check, verifies the signature (ES256 and RS256 keys), and refuses a passkey whose counter goes backwards (possible clone). Failed attempts count towards the normal sign-in lockout and everything is written to the audit trail (`auth.passkey_added`, `auth.passkey_removed`, `auth.login`, `auth.passkey_failed`). Deactivated accounts cannot sign in this way.
- **If someone loses their phone:** they sign in with their password or Google and remove the old device under My profile; an admin can also deactivate the account under Team and customers.

## Privacy Policy and Terms of Service
`/privacy.php` and `/terms.php` are linked in the footer of every customer page. The text is a starting template written for this system (data collected, M-Pesa and SMS providers, Google sign-in, Kenya Data Protection Act rights, deposits, balances, refunds). It reads your restaurant name, address and phone, the deposit percentage, and the **Contact email** under Settings, General. Have it reviewed against your actual practice, and by a lawyer, before relying on it. The "last updated" date sits at the top of each file.

## Receipts
Every payment still gets its own receipt (deposit, balance, cash or manual). In addition, each order has a **combined receipt** (`/order_receipt.php?order=ID`) that lists all its payments in one statement with the order items, the share paid, any refunds taken off, net paid, balance remaining and a *Paid in full* or *Part paid* stamp. Customers find it on their order page, staff on the order page and in the payments list, and each single receipt links to it. If an order's stored total and its payments disagree, staff see a warning on the combined receipt.

## Not included yet
Configurable order cancellation, stock counts, multiple branches.

## Paybill vs Till
Paybill: enter the Paybill as the shortcode, leave "Till number" empty, transaction type Paybill.
Till (Buy Goods): enter the **store (head office) number** as the shortcode, the **till number** in "Till number", transaction type Till. A mismatch gives "The Agent number and Store number entered do not match".

## Refunding deposits (M-Pesa reversal)
Open an order (Admin, Orders) and use **Refund deposit**. Admins only. Choose the amount (full or partial), a reason (customer did not come, delivery not completed, cancelled), and whether to also cancel the order.
- **Automatic:** the app calls Safaricom's Transaction Reversal API on the customer's original M-Pesa code. Set it up under Settings, M-Pesa, *Refunds*: initiator name, initiator password and the Safaricom certificate (.cer) for the environment. The initiator must be an M-Pesa API operator that Safaricom has allowed to reverse transactions, and your M-Pesa account needs enough balance. The result arrives on a protected callback URL and completes the refund; the customer gets an in-app notification and an SMS.
- **Manual record:** if you reversed the payment yourself in the M-Pesa portal, choose "I already refunded it" and enter the reference. The app updates the order and payment records.
- Refunds are searchable under **Payments, Refunds**, and the payments list flags refunded payments. Dashboard figures are net of refunds.
- In Simulation mode refunds complete automatically after a few seconds.
Test automatic reversals in the Daraja sandbox before relying on them: Safaricom enables reversal per shortcode, and the receiver settings (receiver party, identifier type) may need adjusting for your account.

## Forgot password
Customers and staff use **Forgot password?** on the sign-in page. A one-hour, single-use link is emailed to them. Set up outgoing email under Settings, Email (SMTP details of a mailbox on your domain; use the **Save and send test** button). Admins can also reset any password from Team and customers.

## Sign-in lockout
Settings, Security: set how many wrong passwords are allowed and how long the lockout lasts. The same tab lists locked-out addresses with an **Unlock now** button and **Unlock everyone**. If every admin is locked out, run `php bin/unlock.php` on the server.

## Logo and look
Upload the logo under Settings, General; it shows in the header and on receipts. The colour scheme (yellow, leaf green, orange, red with cream and peach neutrals) is set in `public/assets/css/app.css` at the top of the file.

## Upgrading an existing install
Upload the new files over the old ones, but keep `config/config.php`, `public/uploads/` and `storage/`. The database updates itself on the first page load (new tables for password reset and refunds, plus one column on payments).

## Push notifications (alerts when the app is closed)
Order updates, new-order alerts for staff and customer order reminders are delivered as phone and desktop notifications through Web Push, even when the browser or app is closed. No third-party account is needed: the app creates its own keys (Settings, Push).
- **Customers and staff** see a "Turn on notifications" prompt after signing in, and a control panel on the Notifications page (turn on or off for this device, send a test). Signing out removes that device, so alerts do not follow someone onto a shared phone.
- **Needs HTTPS** (localhost is fine for testing). On **iPhone** it needs iOS 16.4 or newer and the app added to the Home Screen (Share, Add to Home Screen); the prompt explains this.
- **Reminders need the cron job** (`* * * * * php /path/to/the-spring/bin/cron.php`). Without it a reminder only fires while someone has the site open. The dashboard warns you if the scheduled task is not running.
- Sending uses the standard (RFC 8291 / VAPID) protocol in plain PHP with the OpenSSL extension. Expired devices are cleaned up automatically. Admins can switch push off or create new keys in Settings, Push.
- The site can also be installed as an app (it ships a web app manifest and icons).

## Order list and cancelling
- On the order list (checkout page) every dish has an **X** that removes it; the + and - buttons still change quantities.
- A customer can **cancel an order whose deposit is unpaid** (button on the order page and on the payment page). It is refused once any deposit is paid (the restaurant refunds those), and while an M-Pesa request is still pending, so a payment in flight cannot be orphaned. If money arrives anyway after a cancellation, staff get an alert to refund it and the event is audited.

## Audit trail (admins only)
Admin, **Audit trail** shows a permanent record of what people did and what the system did by itself: sign-ins and failures, lockouts and unlocks, registrations and password resets, orders and every status change, payment requests and received payments, refunds, menu and price changes, team changes, every settings change (old and new value), data exports, notifications set-up and system jobs.
- Search by person, order number, IP address or detail; filter by area, role and date; export to CSV (the export is itself recorded).
- Passwords, API keys and full phone numbers are never written. A changed secret is recorded as "value hidden".
- Records cannot be edited or deleted in the app. Each is signed (HMAC with your `app_key`) and chained to the previous one; **Verify integrity** detects any record edited or removed directly in the database. Keep `app_key` safe and unchanged.
- Behind Cloudflare or a proxy, set `ip_header` in `config/config.php` (for example `'HTTP_CF_CONNECTING_IP'`) so the trail and the sign-in lockout see the visitor's real address.

## "Invalid Access Token" when refunding
The app now discards a rejected Safaricom token, fetches a new one and retries once, which fixes stale or revoked tokens. If the refund still fails with that message, the Daraja app behind your consumer key does not have the **Transaction Reversal API** enabled for that environment (ask Safaricom, or check My Apps on the Daraja portal), or the key and secret belong to a different app or environment. Until then, refund in the M-Pesa portal and use "I already refunded it" to record it.

## Paying more than the minimum, and cash
- **Deposits:** customers pay at least the set percentage, and can pay more or the whole order (quick buttons for the minimum, 50%, 75%, full, or any amount in between). Every payment stores what it paid, its share of the order total and the balance left after it, and these show on receipts, the order pages and Payments (with a CSV export).
- **Cash:** staff (waiters and admins) use **Record a cash payment** on the order. While the deposit is unpaid, cash counts as the deposit (at least the minimum) and places the order; afterwards it counts against the balance, in parts if needed. Cash receipts show who took the money. A cash deposit can only be refunded by recording how the cash was returned (it cannot be reversed through M-Pesa).

## "DS timeout, user cannot be reached" and money that arrives late
- The customer is told plainly what happened and not to pay twice. A success that arrives **after** a failure or timeout is still recorded.
- The app keeps re-checking such requests with Safaricom for two hours (cron job, and while the site is busy), and **before any retry** it checks whether the earlier attempt actually went through, so a customer is not charged twice.
- Admins can press **Re-check with Safaricom** on a request, and as a last resort **Confirm manually** (admin only): enter the M-Pesa code from the statement or the customer's SMS, the amount and phone. The code must look right, can only be used once, and cannot exceed what is owed. It is stored as a payment "confirmed manually" and audited. If two payments ever cover the same order, staff are alerted to refund the difference.

## Messages
Customers use **Messages** (and the "Message us about this order" button) to write to the restaurant; admins answer from **Messages** in the staff menu. A conversation can go back and forth, be closed and is reopened by a new reply. Unread badges, notifications and push alerts tell each side about new messages, and open conversations update by themselves. Admins can also start a conversation with a customer from an order. Only admins see the inbox; message contents are not copied into the audit trail (only who wrote and when).
