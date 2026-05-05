# Crayons Ink — Order Confirmation Sender

A single-page, browser-only tool for sending order confirmation emails to Crayons Ink customers. It pulls orders from a public Google Sheet (the Google Form responses sheet), previews a filled email template per order, and sends through EmailJS — no backend required.

## Files

```
crayons-sender/
├── index.html      ← the entire app
├── template.html   ← the email body (edit freely; placeholders below)
└── README.md
```

## Run locally

The simplest way is **Live Server** in VS Code:

1. Open this folder in VS Code.
2. Install the "Live Server" extension (Ritwick Dey).
3. Right-click `index.html` → **Open with Live Server**.
4. Your browser opens at `http://127.0.0.1:5500/index.html`.

You can also just double-click `index.html`, but some browsers block `fetch('./template.html')` from `file://` URLs. If the template fails to load, switch to Live Server.

## 1) Make your Google Sheet public

Your Google Form already writes responses into a sheet. To expose it as CSV:

1. Open the responses sheet.
2. **File → Share → Share with others.**
3. Set **General access** to **Anyone with the link** → **Viewer**. Click Done.
4. Copy the sheet ID from the URL — it's the long string between `/d/` and `/edit`:
   ```
   https://docs.google.com/spreadsheets/d/THIS_PART_HERE/edit#gid=0
   ```
5. Build the CSV URL:
   ```
   https://docs.google.com/spreadsheets/d/THIS_PART_HERE/export?format=csv
   ```
   If you have multiple tabs, add `&gid=<tab_id>` (the tab id is in the URL when you click that tab).
6. Paste this URL into the app on Screen 1.

> **Column order matters.** The app expects these columns in this exact order (same as your Google Form):
> `Order Id, Name, Email, Phone number, Alternative Phone Number, Full Address, Payment Method, Sender's bKash number, Transaction Id, Reference/Coupon Code, Timestamp`.
>
> The Order Id from the sheet is used as-is. The delivery zone (Dhaka / Outside Dhaka) is auto-guessed from the Full Address — if "dhaka" appears anywhere in the address it defaults to Dhaka. You can override it per order in the preview modal before sending.

## 2) Set up EmailJS (free tier is fine)

1. Sign up at <https://www.emailjs.com>.
2. **Email Services** → **Add New Service**. Pick Gmail (or any provider you'll send from), connect the account `reachus.crayons@gmail.com`, give it a name. Copy the **Service ID** (e.g. `service_abc1234`).
3. **Email Templates** → **Create New Template**.
   - **To Email**: `{{to_email}}`
   - **From Name**: `Crayons Ink`
   - **Subject**: `Order Confirmation — {{order_id}}`
   - **Content** (switch to the *Code* / HTML view): paste exactly:
     ```html
     {{{html_content}}}
     ```
     The triple-brace `{{{ }}}` tells EmailJS not to escape the HTML.
   - Save. Copy the **Template ID** (e.g. `template_xyz9876`).
4. **Account → General**. Copy the **Public Key** (e.g. `pK_aBc123XyZ`).
5. (Recommended) **Account → Security → Allowed Origins**: add the domain you'll deploy to (e.g. `https://crayons-mail.vercel.app`) and `http://127.0.0.1:5500` for local Live Server.
6. Open the app → expand **EmailJS Settings** → paste all three values → **Save Settings**. The values live in your browser's localStorage; you won't need to re-enter them.

### Test it without sending real mail

Flip the **Test Mode** toggle in EmailJS Settings on. Clicking *Send via EmailJS* in the modal will mark the order as Sent locally and show a yellow "Test Mode — email not sent" banner, but won't hit EmailJS.

## 3) How orders flow through the app

- **Screen 1** — paste the CSV URL, click **Load Orders**.
- **Screen 2** — table of all rows. Search by name/district, filter by Pending/Sent. Each row has a **Preview & Send** button.
- **Screen 3 (modal)** — the email is rendered inside an iframe so its styles don't leak. The right-hand summary card shows the parsed totals. Click **Send via EmailJS** to deliver, or **Download HTML** to save the filled message and forward it manually.

Sent state is persisted to `localStorage` under the key `crayonsink_sent` as a JSON array of Order IDs. Clearing site data resets it.

## Order ID format

`CI-YYYYMMDD-NNN` where `YYYYMMDD` is the order's timestamp date and `NNN` is the row's position in the sheet (3-digit, zero-padded). Example: `CI-20260506-004`.

## Pricing & delivery (hardcoded)

| Item                        | Value         |
| --------------------------- | ------------- |
| Product                     | Crayons High Frequency Vocab Flashcards |
| Price                       | 570 BDT (orig. 800 BDT) |
| Delivery — Dhaka            | 80 BDT (3 days) |
| Delivery — outside Dhaka    | 120 BDT (5 days) |

## Email template placeholders

`template.html` is a plain HTML email. The app fills these tokens:

```
{{Customer Name}}    {{Order ID}}        {{Order Date}}      {{Payment Method}}
{{Delivery Date}}    {{Product Name}}    {{Quantity}}        {{Item Price}}
{{Subtotal}}         {{Shipping Cost}}   {{Total Amount}}    {{Address Line 1}}
{{City}}             {{Postal Code}}     {{Country}}         {{Company Address}}
{{Company Name}}     {{Company Email}}
```

Edit `template.html` freely — just keep the placeholders intact for the fields you want filled.

## Deploy to Vercel (static, no config needed)

1. Push this repo to GitHub (already done if you got here from Claude).
2. Go to <https://vercel.com/new>, click **Import** next to the `crayons-mail-automation` repo.
3. **Framework Preset**: *Other*. Leave Build Command and Output Directory empty.
4. Click **Deploy**. ~30 seconds later you get `https://<project-name>.vercel.app`.
5. Add that domain to EmailJS → Account → Security → Allowed Origins.

## Troubleshooting

- **"Sheet fetch failed (401/403)"** — sheet isn't public. Re-do step 1.
- **Template doesn't load when double-clicking `index.html`** — use Live Server. Browsers block `fetch()` from `file://`.
- **EmailJS error 412** — the recipient's domain isn't in your Allowed Origins, or the template's "To Email" isn't `{{to_email}}`.
- **Email arrives with raw HTML showing as text** — your EmailJS template content isn't using triple braces. It must be `{{{html_content}}}`, not `{{html_content}}`.
