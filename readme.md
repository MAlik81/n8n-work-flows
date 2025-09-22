# testing pre-test: advanced translations — README

A small n8n workflow that, after a DocuSeal **submission.completed** event (or a manual trigger), waits **10 minutes** and then emails the end user with:

* **Mandate\_Signed.pdf** (from DocuSeal URL)
* **Audit\_Log.pdf** (from DocuSeal URL; optional)
* **Terms\_EN.pdf** & **Privacy\_EN.pdf** (from Google Drive, Folder A)
* **Terms\_XX.pdf** & **Privacy\_XX.pdf** if user-locale base language `XX` ≠ `en`
* (Optional) **Terms\_XY.pdf** & **Privacy\_XY.pdf** if legal page language `XY` ≠ `en` and `XY` ≠ `XX`

It also:

* Forwards **Mandate\_Signed.pdf** only to `mandate.to.be.sent@datajoy.eu`
* Creates a **Gmail Draft** containing **all files in Drive Folder B** (“To be sent to data holder”) with:

  * **Subject:** `Placeholder 1`
  * **Body:** `Placeholder 2`

---

## 1) Inputs expected

Provide these fields when you trigger the workflow (Webhook payload or *Execute Workflow* input):

```json
{
  "signer_email": "end.user@example.com",
  "userLocale": "fr-BE",                 // primary user locale
  "userLocales": "fr-BE,en-US",          // optional fallback list
  "legalLang": "nl-2025-09-22 09:47:44", // Country legal dropdown (lang + timestamp)
  "mandate_pdf_url": "https://.../signed.pdf",
  "audit_log_url": "https://.../audit.pdf"
}
```

> If `legalLang` is missing, XY pack is skipped automatically.

---

## 2) Google Drive structure

* **Folder A (end-user pack)**: `<<<DRIVE_FOLDER_A_ID>>>`
  Must contain:

  * `terms_en.pdf`, `privacy_en.pdf`
  * Optional localized pairs: `terms_fr.pdf`, `privacy_fr.pdf`, `terms_ar.pdf`, `privacy_ar.pdf`, …

* **Folder B (data holder pack)**: `<<<DRIVE_FOLDER_B_ID>>>`
  Contains *any* PDFs you want to appear in the **Gmail Draft**.

> **Tip:** Use a **Service Account** credential and share both folders with the SA email (Viewer).

---

## 3) How languages are derived

* **XX** (user locale):

  * From `userLocale` (e.g., `fr-BE` → `fr`), else first of `userLocales`, else `en`.
* **XY** (legal page):

  * From `legalLang` (e.g., `nl-2025-09-22 09:47:44` → `nl`), else falls back to `XX`.
  * XY pack is only added if `XY ≠ en` and `XY ≠ XX`.

---

## 4) What the workflow sends

### End-user email (after 10 minutes)

* Always: **Mandate\_Signed.pdf**, **Terms\_EN.pdf**, **Privacy\_EN.pdf**
* Optional: **Audit\_Log.pdf** (if URL valid)
* Optional: **Terms\_XX.pdf**, **Privacy\_XX.pdf** (if `XX ≠ en`)
* Optional: **Terms\_XY.pdf**, **Privacy\_XY.pdf** (if `XY ≠ en` and `XY ≠ XX`)

**Subject:** `Terms, privacy, mandate and audit logs`
**Body:**

```
Hello,

Please find enclosed, the terms, privacy, mandate and audit logs.

Let us know if you have any questions.

Kr,
Bernard Cornet
Founder - Datajoy - +32497864248
```

### Internal forward

* Sends **Mandate\_Signed.pdf** only to `mandate.to.be.sent@datajoy.eu`.

### Gmail Draft (data holder pack)

* Draft (not sent) with **all files from Folder B**.
* **Subject:** `Placeholder 1`
* **Body:** `Placeholder 2`

---

## 5) Credentials & placeholders to set in n8n

* **Google Drive**: `<<<GDRIVE_CREDENTIAL_NAME>>>`
  (Service Account recommended; share Folder A & B to it)
* **Gmail** (for Draft): `<<<GMAIL_CREDENTIAL_NAME>>>`
* **Email sending** (for end-user & internal): `<<<SMTP_CREDENTIAL_NAME>>>` (or use Gmail Send)
* **Folder IDs**:

  * Folder A → `<<<DRIVE_FOLDER_A_ID>>>`
  * Folder B → `<<<DRIVE_FOLDER_B_ID>>>`

---

## 6) Error handling (concise)

* **Missing localized files (XX/XY)**: The email still goes with EN + Mandate (+ Audit). Missing files are simply not attached.
* **`audit_log_url` empty/404**: Continue without it; optionally add a line in the email body indicating audit log is unavailable.
* **Large email (>20–25 MB)**: (Optional) add a size-guard step before send; drop XY first, then XX; always keep Mandate (+ Audit) + EN.
* **HTTP download errors**: Set node retries (e.g., 3x). If Mandate ultimately fails, include a fallback link and log `missing_mandate=true`.
* **Gmail Draft errors**: Retry once, then alert ops if still failing.

---

## 7) Test checklist

1. Trigger with a **known** `mandate_pdf_url` and `audit_log_url`.
2. Confirm 10-min delay then the end-user email arrives with correct attachments.
3. Confirm internal email gets **Mandate\_Signed.pdf** only.
4. Confirm a Gmail **draft** is created containing all Folder B files.
5. Try with `userLocale="en-US"` (no XX) and with `userLocale="fr-FR"` (adds FR).
6. Try `legalLang="nl-2025-09-22 09:47:44"` to see XY pack when `nl` differs from XX.

---

## 8) Minimal function for language & manifest (for your n8n Function node)

```js
function baseLangFromLocale(s) {
  if (!s) return '';
  s = String(s).trim().toLowerCase().replace(/[^a-z,-]/g, '');
  const first = s.split(',')[0] || '';
  return (first.split('-')[0] || '').trim();
}

function baseLangFromLegal(s) {
  if (!s) return '';
  s = String(s).trim().toLowerCase().replace(/\d{4}-\d{2}-\d{2}.*$/, '').replace(/[-\s]+$/, '');
  const m = s.match(/^[a-z]{2,3}/);
  return m ? m[0] : '';
}

let XX = baseLangFromLocale($json.userLocale) || baseLangFromLocale($json.userLocales) || 'en';
if (!XX) XX = 'en';

const XYraw = $json.legalLang;
let XY = baseLangFromLegal(XYraw) || XX || 'en';

const wantXX = XX !== 'en';
const wantXY = XY !== 'en' && XY !== XX;

const manifest = [
  { key: 'Mandate_Signed.pdf', src: 'url',   url: $json.mandate_pdf_url },
  { key: 'Audit_Log.pdf',      src: 'url',   url: $json.audit_log_url },
  { key: 'Terms_EN.pdf',       src: 'driveA', name: 'terms_en.pdf' },
  { key: 'Privacy_EN.pdf',     src: 'driveA', name: 'privacy_en.pdf' },
];

if (wantXX) {
  manifest.push(
    { key: `Terms_${XX.toUpperCase()}.pdf`,   src: 'driveA', name: `terms_${XX}.pdf` },
    { key: `Privacy_${XX.toUpperCase()}.pdf`, src: 'driveA', name: `privacy_${XX}.pdf` },
  );
}
if (wantXY) {
  manifest.push(
    { key: `Terms_${XY.toUpperCase()}.pdf`,   src: 'driveA', name: `terms_${XY}.pdf` },
    { key: `Privacy_${XY.toUpperCase()}.pdf`, src: 'driveA', name: `privacy_${XY}.pdf` },
  );
}

return [{ json: { ...$json, XX, XY, manifest } }];
```

---

If you want this README as an actual file to download, say the word and I’ll generate a `.md` you can grab.
