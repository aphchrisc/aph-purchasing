# Power Automate Flow Requirements for APH Purchasing App

This document outlines the necessary Power Automate flows required for the proper functioning of the refactored APH Purchasing Power App (`src1`).

## 1. Assign Approver Flow

This flow is crucial for routing requests to the correct personnel based on the request's current phase.

*   **Name Suggestion:** `APH Purchasing - Assign Approver on Phase Change`
*   **Trigger:** Automated Cloud Flow - "When an item is created or modified" (SharePoint)
    *   **Site Address:** Select the SharePoint site hosting the APH Purchasing lists (use Environment Variable `ENV_TENANT_URL`).
    *   **List Name:** `Purchase Requests` (use Environment Variable `ENV_LIST_REQUESTS_ID`).
*   **Trigger Condition (Optional but Recommended):** To prevent unnecessary runs, add a trigger condition to only run when the `Phase` column changes. Example using triggerOutputs:
    ```
    @not(equals(triggerOutputs()?['body/Phase'], triggerBody()?['Phase']))
    ```
    *(Note: Verify exact syntax based on trigger outputs)*
*   **Core Logic:**
    1.  **Get Item:** Get the details of the item that triggered the flow.
    2.  **Initialize Variable:** Create a variable (e.g., `varApprovers` of type Array) to hold the email addresses of the required approver(s).
    3.  **Switch on Phase:** Use a Switch control based on the `Phase` value from the trigger item.
        *   **Case "Pending Mgr":**
            *   Get the requestor's manager (e.g., using Office 365 Users connector - "Get manager (V2)").
            *   Append the manager's email to `varApprovers`.
        *   **Case "Pending DSRIP":**
            *   Identify the DSRIP approver(s) (e.g., from a configuration list, hardcoded group email, or specific user).
            *   Append their email(s) to `varApprovers`.
        *   **Case "Pending Contract":**
            *   Identify the Contract approver(s).
            *   Append their email(s) to `varApprovers`.
        *   **Case "Pending Budget":**
            *   Identify the Budget approver(s).
            *   Append their email(s) to `varApprovers`.
        *   **Case "Pending Purchasing":**
            *   Identify the Purchasing approver(s).
            *   Append their email(s) to `varApprovers`.
        *   **Case "Approved", "Denied", "Draft":** (Default Case)
            *   Do nothing or clear approvers (handled by the app on action, but flow should handle direct phase changes if possible). Set `varApprovers` to an empty array `[]`.
    4.  **Update Item:** Update the triggering SharePoint item (`Purchase Requests` list).
        *   **ID:** ID from the trigger.
        *   **CurrentApprover Claims:** Set the `CurrentApprover` Person column using the emails collected in `varApprovers`. You might need to format this correctly for a multi-select Person column (often involves creating an array of objects like `{"Claims": "i:0#.f|membership|user@domain.com"}`). Use a loop or `join` expression as needed. If `varApprovers` is empty, set the column to empty/null.
*   **Error Handling:** Implement basic error handling (e.g., Try/Catch blocks, checking if users/approvers were found).
*   **Notifications (Optional):** Add an action to notify the assigned approver(s) via email or Teams.

## 2. Archive Requests Flow

This flow is triggered manually from the Admin screen in the Power App to handle old requests.

*   **Name Suggestion:** `APH Purchasing - Archive Completed Requests`
*   **Trigger:** Instant Cloud Flow - "Manually trigger a flow" (Power Apps V2)
    *   *No inputs are strictly required from the app based on the current implementation, but you could add inputs like "Archive Older Than (Days)" if needed.*
*   **Core Logic:**
    1.  **Define Archive Criteria:**
        *   Determine the cutoff date (e.g., `addDays(utcNow(), -180)` for requests older than 180 days).
        *   Identify statuses to archive (e.g., "Approved", "Denied").
    2.  **Get Items:** Use the SharePoint "Get items" action.
        *   **Site Address:** Select the SharePoint site.
        *   **List Name:** `Purchase Requests`.
        *   **Filter Query:** Construct an OData filter query to get items matching the criteria. Example:
            ```
            (Phase eq 'Approved' or Phase eq 'Denied') and Modified lt '@{formatDateTime(variables('cutoffDate'), 'yyyy-MM-ddTHH:mm:ssZ')}'
            ```
            *(Adjust `cutoffDate` variable name and date format as needed)*
    3.  **Apply to Each:** Loop through the items returned by "Get items".
        *   **Inside Loop:**
            *   **(Option A) Update Status:** Update the `Phase` of the item to "Archived" (add this as a valid choice in the SharePoint column).
            *   **(Option B) Move Item:** Copy the item to a separate "Archived Purchase Requests" SharePoint list and then delete the original item. This is cleaner but more complex.
            *   *Consider logging which items were archived.*
    4.  **Respond to Power App (Optional):** If you want to provide feedback beyond the app's basic notification, use the "Respond to a PowerApp or flow" action. Add outputs like "ArchivedCount".
*   **Error Handling:** Implement error handling within the loop and for the "Get items" action.

---

**Note:** These are baseline requirements. The specific implementation details (how approvers are identified, exact archive logic) may need refinement based on APH's specific organizational structure and policies. Ensure the flows use connection references configured within the Power Platform Solution containing the app and flows for proper ALM.