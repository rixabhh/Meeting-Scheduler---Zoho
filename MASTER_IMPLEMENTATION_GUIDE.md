# Zero-Touch Conversion: Master Implementation Guide

This is your **single source of truth**. It contains the Setup instructions, the Logic explanations, and the **Complete Code** for all functions.

---

## Part 1: Architecture & Visibility
You asked for "Full Functionality and Visibility".
- **Functionality**: We use **Zoho CRM Custom Functions** (Deluge) because they are fast, free (included in Enterprise), and handle complex logic like "Round Robin" best.
- **Visibility**: To monitor this, you will use:
  1.  **Lead Fields**: `Booking Status` and `Booking Failure Reason` give you instant per-lead status.
  2.  **Settings > Developer Space > Functions > Usage**: Shows execution logs, success/failure counts, and debug print statements.

---

## Part 2: Setup Checklist
1.  **GCal API**: Enable `Google Calendar API` in GCP Console. Create Standard OAuth Credentials.
2.  **Zoho Connections**:
    -   `google_calendar_conn`: Scopes `calendar.events`, `calendar.readonly`.
    -   `zohocrm_conn`: Scopes `ZohoCRM.modules.ALL`, `ZohoCRM.users.READ`, `ZohoCRM.settings.groups.READ`.
3.  **Zoho Groups**: Create `LST_Ship_Team`, `LST_Fulfil_Team`, `FST_Team`.
4.  **Org Variables**: Create `rr_index_lst_ship`, `rr_index_lst_fulfil`, `rr_index_fst` (int, default 0).
5.  **Lead Fields**: Ensure `Segment`, `Order_Value`, `Preferred_Meeting_Time` exist.

---

## Part 3: The Code (Copy-Paste Ready)

### Function 1: utils_get_lead_bucket
*Determines which team gets the lead.*

```deluge
/*
    Return: Map (bucket_details)
*/
lead_details = zoho.crm.getRecordById("Leads", lead_id);
service_type = lead_details.get("Segment"); 
order_value = ifnull(lead_details.get("Order_Value"), 0.0).toDecimal();

bucket = "";
pipeline = "";
group_name = "";
rr_variable_key = "";

if(service_type.containsIgnoreCase("Ship") && !service_type.containsIgnoreCase("Fulfill")) {
    bucket = "LST-Ship";
    pipeline = "Ship";
    group_name = "LST_Ship_Team";
    rr_variable_key = "rr_index_lst_ship";
} else {
    if(order_value < 3000) {
        bucket = "LST-Fulfil";
        pipeline = "SME 2.0";
        group_name = "LST_Fulfil_Team";
        rr_variable_key = "rr_index_lst_fulfil";
    } else {
        bucket = "FST";
        pipeline = "Enterprise 2.0";
        group_name = "FST_Team";
        rr_variable_key = "rr_index_fst";
    }
}

if(group_name == "") {
    return {"error": "No Logic Matched"};
}

return {"bucket": bucket, "pipeline": pipeline, "group_name": group_name, "rr_variable_key": rr_variable_key};
```

### Function 2: utils_get_rr_candidate
*Finds the next user in line WITHOUT updating the counter (Peek)*

```deluge
/*
    Args: group_name, variable_name
*/
groups = invokeurl[url: "https://www.zohoapis.com/crm/v2/settings/groups", type: GET, connection: "zohocrm_conn"].get("groups");
target_group_id = null;
for each g in groups {
    if(g.get("name") == group_name) { target_group_id = g.get("id"); }
}

if(target_group_id == null) { return {"error": "Group Not Found"}; }

group_data = invokeurl[url: "https://www.zohoapis.com/crm/v2/settings/groups/" + target_group_id, type: GET, connection: "zohocrm_conn"];
users_list = group_data.get("groups").get(0).get("sources");
active_users = List();
for each u in users_list {
    if(u.get("type") == "user") { active_users.add({"id": u.get("source").get("id")}); }
}

if(active_users.size() == 0) { return {"error": "No Users"}; }

current_idx = ifnull(zoho.crm.getGlobalVariable(variable_name), 0).toNumber();
winner = active_users.get(current_idx % active_users.size());
winner_details = invokeurl[url: "https://www.zohoapis.com/crm/v2/users/" + winner.get("id"), type: GET, connection: "zohocrm_conn"];
return {"id": winner.get("id"), "email": winner_details.get("users").get(0).get("email")};
```

### Function 3: utils_check_google_availability
*Checks GCal FreeBusy*

```deluge
/*
    Args: email, start_time (DateTime)
*/
start_str = start_time.toString("yyyy-MM-dd'T'HH:mm:ssZ");
end_str = start_time.addMinutes(30).toString("yyyy-MM-dd'T'HH:mm:ssZ");

param_map = {"timeMin": start_str, "timeMax": end_str, "items": [{"id": email}]};
response = invokeurl[
    url: "https://www.googleapis.com/calendar/v3/freeBusy",
    type: POST,
    parameters: param_map.toString(),
    headers: {"Content-Type": "application/json"},
    connection: "google_calendar_conn"
];

busy_list = response.get("calendars").get(email).get("busy");
return if(busy_list.size() > 0, false, true);
```

### Function 4: utils_update_rr_index
*Commit: Move the line forward*

```deluge
/*
    Args: variable_name
*/
curr = ifnull(zoho.crm.getGlobalVariable(variable_name), 0).toNumber();
new_val = if(curr > 50000, 0, curr + 1);
param = {"variables": [{"api_name": variable_name, "value": new_val.toString()}]};
invokeurl[url: "https://www.zohoapis.com/crm/v2/orgVariables", type: PUT, parameters: param.toString(), connection: "zohocrm_conn"];
return "Success";
```

### Function 5: Main Automation (Inbound_Lead_Conversion)
*Orchestrator*

```deluge
lead_id_long = lead_id.toLong();
rec = zoho.crm.getRecordById("Leads", lead_id_long);

// 1. Idempotency & Validation
if(rec.get("Booking_Status") == "Success") { return "Ignored"; }
if(rec.get("Preferred_Meeting_Time") == null) {
    zoho.crm.updateRecord("Leads", lead_id_long, {"Booking_Failure_Reason": "Missing Time", "Booking_Status": "Failed"});
    return "Error: Missing Time";
}

// 2. Route
route = this.utils_get_lead_bucket(lead_id_long);
if(route.containKey("error")) {
    zoho.crm.updateRecord("Leads", lead_id_long, {"Booking_Failure_Reason": "No Logic", "Booking_Status": "Failed"});
    return "Error: Routing";
}
zoho.crm.updateRecord("Leads", lead_id_long, {"Assigned_Bucket": route.get("bucket")});

// 3. Assign
candidate = this.utils_get_rr_candidate(route.get("group_name"), route.get("rr_variable_key"));
if(candidate.containKey("error")) { return "Error: No Reps"; }

// 4. Availability
is_free = this.utils_check_google_availability(candidate.get("email"), rec.get("Preferred_Meeting_Time").toTime("yyyy-MM-dd'T'HH:mm:ss"));
if(!is_free) {
    zoho.crm.updateRecord("Leads", lead_id_long, {"Booking_Failure_Reason": "Rep Busy", "Booking_Status": "Failed"});
    // Send Alert Email Code Here...
    return "Failed: Busy";
}

// 5. Book
event_body = {
    "summary": rec.get("Company") + " - Intro Call",
    "start": {"dateTime": rec.get("Preferred_Meeting_Time")},
    "end": {"dateTime": rec.get("Preferred_Meeting_Time").toTime("yyyy-MM-dd'T'HH:mm:ss").addMinutes(30).toString("yyyy-MM-dd'T'HH:mm:ss")},
    "attendees": [{"email": candidate.get("email")}, {"email": rec.get("Email")}]
};
cal_resp = invokeurl[url: "https://www.googleapis.com/calendar/v3/calendars/primary/events", type: POST, parameters: event_body.toString(), connection: "google_calendar_conn"];

// 6. Convert
convert_map = {
    "overwrite": true,
    "notify_lead_owner": true,
    "notify_new_entity_owner": true,
    "assign_to": candidate.get("id"),
    "Deals": {
        "Deal_Name": rec.get("Company") + " - Deal",
        "Pipeline": route.get("pipeline"),
        "Stage": "Qualification",
        "Closing_Date": zoho.currentdate.addMonth(1)
    }
};
zoho.crm.convertLead(lead_id_long, convert_map);

// 7. Update Index
this.utils_update_rr_index(route.get("rr_variable_key"));

return "Success";
```
