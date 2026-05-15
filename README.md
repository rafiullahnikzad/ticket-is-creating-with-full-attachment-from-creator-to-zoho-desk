# Zoho Creator → Zoho Desk: Ticket Creation with Full Attachments

A Zoho Deluge function that reads a submitted **Main Onboarding Wizard** record from Zoho Creator, builds a richly formatted HTML ticket description, creates a ticket in Zoho Desk, and uploads all file attachments from the main record and every subform row (Employee Details, Professional Licenses, and Professional Certifications).

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Function Signature](#function-signature)
- [How It Works](#how-it-works)
  - [Step 1 – Fetch Creator Record](#step-1--fetch-creator-record)
  - [Step 2 – Build HTML Description](#step-2--build-html-description)
  - [Step 3 – Resolve Desk Contact](#step-3--resolve-desk-contact)
  - [Step 4 – Create Ticket](#step-4--create-ticket)
  - [Step 5 – Upload Main Record Attachments](#step-5--upload-main-record-attachments)
  - [Step 6 – Upload Subform Attachments](#step-6--upload-subform-attachments)
- [Ticket Description Sections](#ticket-description-sections)
- [File Fields Handled](#file-fields-handled)
- [Configuration Reference](#configuration-reference)
- [Error Handling](#error-handling)
- [Limitations](#limitations)
- [License](#license)

---

## Overview

When a contractor completes the onboarding wizard in Zoho Creator, this function is triggered (on form submission or via a button/workflow). It:

1. Reads all structured data from the Creator record and its subforms.
2. Builds a detailed, styled HTML description covering every section of the wizard.
3. Finds or falls back to a contact in Zoho Desk by the submitter's email.
4. Creates a new Desk ticket with custom fields pre-populated.
5. Downloads and re-uploads every file attachment — from the main form and all three subforms — directly to the ticket.

---

## Features

- **Full HTML ticket description** — contact info, company details, location, employees, services, insurance, licenses, certifications, and software — rendered as styled tables inside Zoho Desk.
- **Subform data extraction** — iterates Employee Details, Professional License Registration, and Professional Certification subform rows and includes them in the description.
- **Attachment pipeline** — downloads each file from Zoho Creator via the v2.1 API and uploads it to the Desk ticket as a multipart attachment.
- **Graceful fallback** — if the submitter's email is not found as a Desk contact, the function falls back to a test contact rather than failing silently.
- **Per-file error handling** — large or unreadable files (> 5 MB) are caught, logged, and skipped without aborting the rest of the upload process.
- **Detailed logging** — every major step emits `info` statements for easy debugging in the Zoho Creator log viewer.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Zoho Creator app | App name: `your-app` (replace with your actual app name) |
| Zoho Creator connection | Connection name: `zohocreator` (OAuth, with file-download scope) |
| Zoho Desk connection | Connection name: `zohodesk` (OAuth, with ticket + attachment scope) |
| Zoho Desk Department ID | The department where tickets will be created |
| Zoho Desk Team ID | The team that will own the new tickets |
| Creator reports | `All_Main_Onboarding_Wizards`, `All_Employee_Details`, `All_Employee_Professional_License_Report`, `Company_Professional_Certification_Report` must be accessible via API |

---

## Setup

1. **Copy the function** into your Zoho Creator app under **Functions**.
2. **Replace the placeholder values** in the [Configuration Reference](#configuration-reference) section below.
3. **Verify connection names** — ensure `zohocreator` and `zohodesk` connections are authorised in your Creator app under **Settings → Connections**.
4. **Call the function** from a form's On Success workflow action or a custom button, passing the record ID:

```deluge
creatingTicket(input.ID);
```

---

## Function Signature

```deluge
void creatingTicket(int id)
```

| Parameter | Type | Description |
|---|---|---|
| `id` | `int` | The Creator record ID of the submitted Main Onboarding Wizard record |

---

## How It Works

### Step 1 – Fetch Creator Record

The function fetches the record from the `Main_Onboarding_Wizard` form using the Deluge data object:

```deluge
recordData = Main_Onboarding_Wizard[ID == recordID];
```

All field values are extracted with `ifnull(record.FieldName, "")` to prevent null-related errors during string concatenation.

---

### Step 2 – Build HTML Description

A multi-section HTML string is assembled covering every wizard section (see [Ticket Description Sections](#ticket-description-sections)). Each section is rendered as a bordered HTML table with bold labels in the left column.

Subform data (employees, licenses, certifications) is first collected into Deluge `List` and `Map` structures, then rendered as numbered sub-tables inside the description.

---

### Step 3 – Resolve Desk Contact

The submitter's email (`POC_Email`) is searched in Zoho Desk:

```
GET https://desk.zoho.com/api/v1/contacts/search?email=<email>
```

- If a contact is found → its ID is used.
- If not found → a fallback test contact (`test.user@example.com`) is searched.
- If neither is found → the function exits early with a log message.

> **Note:** Replace `test.user@example.com` with a real fallback contact email in your environment.

---

### Step 4 – Create Ticket

A `POST` request is made to `https://desk.zoho.com/api/v1/tickets` with:

| Field | Value |
|---|---|
| `subject` | POC Name from the record |
| `departmentId` | Configured department ID |
| `teamId` | Configured team ID |
| `status` | `Open` |
| `priority` | `High` |
| `channel` | `Email` |
| `contactId` | Resolved in Step 3 |
| `description` | Full HTML built in Step 2 |
| `cf.cf_lead_workflow_status` | `Pre Verification Call Needed` |
| `cf.cf_missing_min_ob_info` | `true` |

The returned ticket ID is used for all subsequent attachment uploads.

---

### Step 5 – Upload Main Record Attachments

The main record is fetched again via the Creator v2.1 REST API to get raw file paths:

```
GET https://www.zohoapis.com/creator/v2.1/data/your-portal/your-app/report/All_Main_Onboarding_Wizards/<id>
```

Each file path is normalised (replacing `/api/v2.1/` with `/creator/v2.1/data/`) and downloaded, then uploaded to the Desk ticket:

```
POST https://desk.zoho.com/api/v1/tickets/<ticketId>/attachments
```

Fields processed: Company Logo (single), W9 (multi), COI (multi), State Exemption Certificate (multi), Country File Upload (multi), Zip File Upload (multi).

---

### Step 6 – Upload Subform Attachments

For each of the three subforms, the function:

1. Reads the subform row IDs from the main record response.
2. Fetches each row's full data from its dedicated Creator report endpoint.
3. Extracts the file upload field.
4. Downloads and uploads each file to the Desk ticket.
5. Catches and logs any oversized or unreadable files without stopping the loop.

| Subform | Creator Report | File Field |
|---|---|---|
| Employee Details | `All_Employee_Details` | `File_upload` |
| Professional License Registration | `All_Employee_Professional_License_Report` | `Upload_a_copay_of_License` |
| Professional Certification | `Company_Professional_Certification_Report` | `Upload_a_copay_of_Certification` |

---

## Ticket Description Sections

The HTML description includes the following sections in order:

1. **Contact Information**
   - Credentialing & Compliance POC
   - General Business Questions POC
   - Lead Notifications POC

2. **Company Information**
   - Legal name, Tax ID, Website, DBA, additional legal names

3. **Location Information**
   - Ownership structure, roofing services flag
   - Physical address and mailing address (conditional)

4. **Employee Information**
   - Owner/employee counts, job-site headcount, policy access count, corporate ownership
   - Employee Details subform rows (name, email, phone, role, background check status)

5. **Service Information**
   - Services performed (multi-select)

6. **Service Territory**
   - County coverage flag; states served or zip/county preference

7. **Insurance Information**
   - COI upload status, coverage types held, WC exemption
   - Per-coverage tables: Auto Liability, General Liability, Professional Liability, Umbrella Liability, Bailees, Contractor Pollution Liability, Workers Compensation/Employer Liability

8. **Professional License/Registration**
   - Total license count
   - License subform rows (holder, number, state, type, expiry)

9. **Professional Certifications**
   - Total certification count
   - Certification subform rows (holder, number, state, type, expiry)

10. **Software & Technology**
    - Softwares used, Xactimate details, Symbility ID, MICA ID, contents documentation software

> A footer note reminds agents to check the ticket's **Attachments** tab for all uploaded files.

---

## File Fields Handled

### Main Record

| Field | Type |
|---|---|
| `Company_Logo` | Single file |
| `Upload_W9` | Multi-file array |
| `Upload_a_copy_of_COI` | Multi-file array |
| `Upload_a_copy_of_your_State_Exemption_Certificate` | Multi-file array |
| `Country_File_upload` | Multi-file array |
| `Zip_File_upload` | Multi-file array |

### Employee Details Subform

| Field | Type |
|---|---|
| `File_upload` | Multi-file array |

### Professional License Registration Subform

| Field | Type |
|---|---|
| `Upload_a_copay_of_License` | Multi-file array |

### Professional Certification Subform

| Field | Type |
|---|---|
| `Upload_a_copay_of_Certification` | Multi-file array |

---

## Configuration Reference

Replace these values before deploying:

| Placeholder | Location in Code | Replace With |
|---|---|---|
| `YOUR_ORG_ID` | Creator API URLs | Your Zoho org ID |
| `your-portal` | Creator API URLs | Your Creator portal name |
| `your-app` | Creator API URLs | Your Creator app link name |
| `testingDeptId` | Step 4 ticket map | Your Zoho Desk Department ID |
| `testingTeamId` | Step 4 ticket map | Your Zoho Desk Team ID |
| `test.user@example.com` | Step 3 fallback contact search | A valid fallback contact email in your Desk |
| `zohocreator` | All `invokeurl` connection params | Your Creator OAuth connection name |
| `zohodesk` | All `invokeurl` connection params | Your Desk OAuth connection name |
| `All_Main_Onboarding_Wizards` | Report names in API URLs | Your actual Creator report link name |
| `All_Employee_Details` | Subform report URL | Your employee subform report link name |
| `All_Employee_Professional_License_Report` | Subform report URL | Your license subform report link name |
| `Company_Professional_Certification_Report` | Subform report URL | Your certification subform report link name |

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Record not found | Logs `❌ RECORD NOT FOUND` and returns early |
| Contact not found in Desk | Falls back to test contact; logs warning |
| No fallback contact found | Logs `❌ NO CONTACT FOUND` and returns early |
| File > 5 MB or download error | Caught by `try/catch`; file added to `skippedFiles` list; loop continues |
| Any unhandled exception | Outer `try/catch` logs the error and exits gracefully |

All results (uploaded and skipped files) are tracked in `uploadResults` and `skippedFiles` lists and printed at the end of each upload phase.

---

## Limitations

- **5 MB file limit** — Zoho Creator's Deluge `invokeurl` cannot reliably download files larger than ~5 MB. Oversized files are skipped and logged.
- **API call count** — Each subform row requires one additional API call to fetch its full data. For records with many subform rows, this can be significant. Monitor against Zoho API rate limits.
- **Single ticket per record** — The function does not check for duplicate tickets. Add a duplicate-check step if re-submissions are possible.
- **`ticketId` scope** — `ticketId` is declared inside the `for each record` loop. The attachment code outside the loop relies on it being in scope from the last iteration. If the loop processes zero records, attachment upload will fail silently.

---

## License

MIT — free to use, modify, and distribute with attribution.

---

*Built with Zoho Deluge | Zoho Creator v2.1 API | Zoho Desk API v1*
