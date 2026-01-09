# Zero-Touch System: Admin Guide

**Owner:** Umang Seth  
**Purpose:** Manage Groups, Users, and Availability Configuration.

## 1. Managing Availability Pools (Groups)
The Round Robin system relies on Zoho CRM **Groups** to know who is eligible for leads.

### Adding a New Rep
1. Go to **Settings > Users & Control > Groups**.
2. Select the relevant group (e.g., `LST_Ship_Team`).
3. Click **Edit**.
4. Add the new User to the group.
5. **Important:** Ensure the user has their Google Calendar connected/authorized if individual auth is used (or ensure the service account checks their variability).

### Removing a Rep
1. Go to **Settings > Users & Control > Groups**.
2. Edit the group and remove the User.
3. *Note:* The Round Robin logic automatically adjusts to the new group size. You do not need to reset the index variable.

## 2. Troubleshooting "Rep Busy" Failures
If the system sends an alert "Lead Conversion Failed - Rep Busy":
1. Check the **Notes** on the Lead record. It will say who was attempted (Winner User).
2. Manually check that User's Google Calendar for that time slot.
3. **Resolution:**
   - Manually book the meeting for another time or assign to another rep.
   - Convert the Lead manually.

## 3. Resetting Round Robin
If the distribution seems "stuck" (e.g., always going to one person, though unlikely):
1. Go to **Settings > Developer Space > Global Variables**.
2. Search for `rr_index_...`.
3. Reset the value to `0` (or any number).
