# Power Automate Flow Requirements for APH Purchasing App

This document outlines the necessary Power Automate flows required for the proper functioning of the refactored APH Purchasing Power App (`src1`).

## 1. Assign Approver & Notify Flow

This flow is crucial for routing requests to the correct personnel based on the request's current phase and notifying the requestor of status changes.

*   **Name Suggestion:** `APH Purchasing - Assign Approver & Notify on Phase Change`
*   **Trigger:** Automated Cloud Flow - "When an item is created or modified" (SharePoint)
    *   **Site Address:** Select the SharePoint site hosting the APH Purchasing lists (use Environment Variable `ENV_TENANT_URL`).
    *   **List Name:** `Purchase Requests` (use Environment Variable `ENV_LIST_REQUESTS_ID`).
*   **Trigger Condition (Recommended):** To prevent unnecessary runs, add a trigger condition to only run when the `Phase` column changes OR when `RevisionComments` is modified (to catch the send-back scenario). Example:
    ```
    @or(not(equals(triggerOutputs()?['body/Phase'], triggerBody()?['Phase'])), not(equals(triggerOutputs()?['body/RevisionComments'], triggerBody()?['RevisionComments'])))
    ```
    *(Note: Verify exact syntax based on trigger outputs)*
*   **Core Logic:**
    1.  **Get Item:** Get the details of the item that triggered the flow (`triggerOutputs()?['body']`). Store the `Requestor Email` and `RevisionComments` in variables for easier use.
    2.  **Initialize Variable:** Create a variable (e.g., `varApprovers` of type Array) to hold the email addresses of the required approver(s). Initialize as empty `[]`.
    3.  **Switch on Phase:** Use a Switch control based on the `Phase` value from the trigger item.
        *   **Case "Pending Mgr":**
            *   Get the requestor's manager (e.g., using Office 365 Users connector - "Get manager (V2)").
            *   Append the manager's email to `varApprovers`.
        *   **Case "Pending DSRIP":**
            *   Identify the DSRIP approver(s). Append email(s) to `varApprovers`.
        *   **Case "Pending Contract":**
            *   Identify the Contract approver(s). Append email(s) to `varApprovers`.
        *   **Case "Pending Budget":**
            *   Identify the Budget approver(s). Append email(s) to `varApprovers`.
        *   **Case "Pending Purchasing":**
            *   Identify the Purchasing approver(s). Append email(s) to `varApprovers`.
        *   **Case "Approved":**
            *   Set `varApprovers` to empty `[]`.
            *   **Send Email Notification:** Add "Send an email (V2)".
                *   To: Requestor Email variable.
                *   Subject: `Purchase Request Approved: ID {Request ID}`
                *   Body: Inform the user their request (ID, Title) has been fully approved. Include a link.
        *   **Case "Draft":**
            *   Set `varApprovers` to empty `[]`.
            *   **Check for Revision Comments:** Add a Condition: `RevisionComments` variable is not empty/null.
            *   **If Yes (Sent Back for Revision):**
                *   **Send Email Notification:** Add "Send an email (V2)".
                    *   To: Requestor Email variable.
                    *   Subject: `Purchase Request Revision Needed: ID {Request ID}`
                    *   Body: Inform the user their request requires revision. Include the `RevisionComments` from the variable. Explain it's back in their "My Requests" list.
            *   **If No:** (Normal draft state, do nothing extra).
        *   **Case "Denied":** (This case might only be hit if manually set or if original deny logic is kept elsewhere)
             *   Set `varApprovers` to empty `[]`.
             *   *(Optional)* Send a "Denied" notification email if this state is used.
        *   **Default Case:** (Handles Archived, potentially Denied if not explicitly handled)
            *   Set `varApprovers` to empty `[]`.
    4.  **Update Item:** Update the triggering SharePoint item (`Purchase Requests` list).
        *   **ID:** ID from the trigger.
        *   **CurrentApprover Claims:** Set the `CurrentApprover` Person column using the emails collected in `varApprovers`. Format correctly for multi-select Person column (array of `{"Claims": "i:0#.f|membership|user@domain.com"}`). If `varApprovers` is empty, set the column to empty/null. **Important:** Only update this field if the phase *requires* an approver; don't clear it unnecessarily if the trigger was just for comments on a Draft item. Add a condition before this step to check if `varApprovers` needs setting.
*   **Error Handling:** Implement basic error handling (e.g., Try/Catch blocks, checking if users/approvers were found).
*   **Concurrency Control:** Set Concurrency Control on the trigger to 1 (Single instance) to prevent race conditions if multiple updates happen quickly.

## 2. Archive Requests Flow

This flow is triggered manually from the Admin screen in the Power App to handle old requests.

*   **Name Suggestion:** `APH Purchasing - Archive Completed Requests`
*   **Trigger:** Instant Cloud Flow - "Manually trigger a flow" (Power Apps V2)
*   **Core Logic:**
    1.  **Define Archive Criteria:** Cutoff date (e.g., `addDays(utcNow(), -180)`), statuses (`Approved`, `Denied`).
    2.  **Get Items:** SharePoint "Get items" action filtered by status and `Modified` date `<` cutoff date.
    3.  **Apply to Each:** Loop through returned items.
        *   **(Option A) Update Status:** Update `Phase` to "Archived". Clear `CurrentApprover`.
        *   **(Option B) Move Item:** Copy to "Archived Purchase Requests" list, then delete original.
    4.  **Respond to Power App (Recommended):** Use "Respond to a PowerApp or flow". Add outputs like `ArchivedCount` (initialize a counter variable before the loop and increment inside). Send success/failure status.
*   **Error Handling:** Implement error handling.

---

**Note:** These flows should use connection references configured within the Power Platform Solution. Test email content and formatting thoroughly.