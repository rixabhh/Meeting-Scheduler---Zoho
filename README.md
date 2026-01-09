# Lead Conversion & Meeting Booking

**A fully automated engine for Zoho CRM that routes leads, checks real-time Google Calendar availability, books meetings, and creates deals instantly.**

## 📖 Overview

The **Zero-Touch Meeting Scheduler** eliminates the manual bottleneck between an SDR qualifying a lead and an Account Executive (AE) taking the meeting. By leveraging Zoho CRM Deluge and the Google Calendar API, this system performs the following actions in seconds:

1.  **Segments** the lead into the correct Sales Team (Ship, SME 2.0, or Enterprise 2.0).
2.  **Selects** an available AE using a robust **Round-Robin** algorithm.
3.  **Checks** the AE's real-time availability via **Google Calendar** (FreeBusy check).
4.  **Books** the meeting directly on the calendar if the slot is free.
5.  **Converts** the Lead to a Contact/Account and **Creates a Deal** in the correct pipeline.

---

## 📂 Repository Structure

### `/Code` (Deluge Scripts)
The core logic resides here. These scripts must be installed as **Custom Functions** in Zoho CRM.

* **`automation_inbound_lead_conversion.dg`**: The main orchestrator. Triggers on Workflow Rules, calls utilities, and executes the booking/conversion.
* **`utils_get_lead_bucket.dg`**: Routing logic. Classifies leads into `LST-Ship`, `LST-Fulfil`, or `FST` based on Segment and Order Volume.
* **`utils_get_rr_candidate.dg`**: Fetches the next AE from the appropriate Zoho Group using a round-robin pointer.
* **`utils_check_google_availability.dg`**: Connects to Google Calendar API to verify if the selected AE is free at the requested time.
* **`utils_update_rr_index.dg`**: Updates the Global Variable pointer after a successful assignment to ensure fair distribution.

### `/Widget` (SDR Console)
A custom widget for SDRs to visualize routing and trigger automation manually if needed.

* **`widget_ready_to_upload.zip`**: The pre-packaged widget ready for Zoho Developer Console.
* **`index.html`**: The source code for the widget interface (Tailwind CSS + Zoho SDK).
* **`WIDGET_INSTALLATION.md`**: Step-by-step guide to installing the widget.

### `/Docs` (Manuals)
* **`MASTER_IMPLEMENTATION_GUIDE.md`**: The technical "Bible" for the project. Contains all logic, setup steps, and code blocks.
* **`setup_guide.md`**: Infrastructure instructions (GCP Console, API Scopes, Zoho Connections).
* **`admin_guide
