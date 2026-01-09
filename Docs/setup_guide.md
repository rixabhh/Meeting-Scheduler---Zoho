# Environment & Infrastructure Setup Guide (V2)

This guide details the steps required to configure Google Cloud Platform and Zoho CRM for the "Zero-Touch Inbound Lead Conversion" system.

## 1. Google Cloud Platform (GCP) Configuration

### 1.1 Create a New Project
1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (e.g., `zoho-lead-conversion`).

### 1.2 Enable Google Calendar API
1. Navigate to **APIs & Services > Library**.
2. Search for `Google Calendar API`.
3. Click **Enable**.

### 1.3 Configure OAuth Consent Screen
1. Navigate to **APIs & Services > OAuth consent screen**.
2. Select **Internal** (if only used within your org) or **External**.
3. Fill in the required application details (App Name, Support Email).
4. Save and Continue.

### 1.4 Create OAuth 2.0 Credentials
1. Navigate to **APIs & Services > Credentials**.
2. Click **Create Credentials > OAuth client ID**.
3. Application type: **Web application**.
4. Name: `Zoho CRM Connection`.
5. **Authorized Redirect URIs**: You will get this from Zoho CRM in the next step.
   - *Note down the `Client ID` and `Client Secret` for now.*

---

## 2. Zoho CRM Connections

### 2.1 Google Calendar Connection (`google_calendar_conn`)
1. In Zoho CRM, go to **Settings > Developer Space > Connections**.
2. Click **Create Connection**.
3. Select **Google Calendar**.
4. Choose **Default Service** or Custom (if using your own Client ID/Secret from Step 1.4 - Recommended for prod).
5. Scopes:
   - `https://www.googleapis.com/auth/calendar.events`
   - `https://www.googleapis.com/auth/calendar.readonly`
6. Name the connection **`google_calendar_conn`**.
7. Authorize the connection.

### 2.2 Zoho CRM Connection (`zohocrm_conn`)
This connection allows the script to access CRM modules and users.
1. Go to **Settings > Developer Space > Connections**.
2. Click **Create Connection**.
3. Select **Zoho CRM**.
4. Scopes:
   - `ZohoCRM.settings.groups.READ`
   - `ZohoCRM.users.READ`
   - `ZohoCRM.modules.ALL`
5. Name the connection **`zohocrm_conn`**.
6. Authorize.

---

## 3. Zoho CRM Configuration

### 3.1 User Groups (Availability Pools)
1. Go to **Settings > Users & Control > Groups**.
2. Create the following groups and add the relevant Users:
   - **`LST_Ship_Team`**
   - **`LST_Fulfil_Team`**
   - **`FST_Team`**

### 3.2 Organization Variables (Round Robin Trackers)
1. Go to **Settings > Developer Space > Global Variables**.
2. Create the following variables (Type: **Integer**, Default: **0**):
   - `rr_index_lst_ship`
   - `rr_index_lst_fulfil`
   - `rr_index_fst`

### 3.3 Fields (Critical Updates)
Please ensure the following fields exist in the **Leads** module.

| Field Name | Type | Options / details |
| :--- | :--- | :--- |
| **Preferred Meeting Time** | DateTime | |
| **Segment** | Picklist | Ship, Warehousing, etc. |
| **Order Value** | Currency | |
| **Booking Status** | Picklist | Pending, Success, Failed |
| **Booking Failure Reason** | Single Line | |
| **Assigned Bucket** | Picklist | LST-Ship, LST-Fulfil, FST |
| **Assigned AE** | Lookup (User) | |
