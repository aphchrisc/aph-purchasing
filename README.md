# APH Purchasing Power App - User Guide

Welcome to the Austin Public Health (APH) Purchasing Power App! This application streamlines the process for requesting and approving purchases.

## Purpose

This app provides a modern, digital workflow for:
*   Submitting new purchase requests.
*   Tracking the status of your submitted requests.
*   Reviewing and approving requests assigned to you.
*   Managing the purchasing process (for Admins).

## User Roles

*   **Requestor:** Any APH staff member who needs to submit a purchase request.
*   **Approver:** Staff members (Managers, DSRIP, Contract, Budget, Purchasing teams) responsible for reviewing and approving requests at different stages.
*   **Admin:** Purchasing administrators who oversee the process, manage lists, and perform administrative tasks.

## Getting Started

Access the application through the link provided by APH IT or the Purchasing department. Sign in using your standard APH credentials.

## Main User Flow: Submitting a Request

1.  **Home Screen:** Click the "New Request" button.
2.  **New Request Form (Multi-Step Wizard):**
    *   **Step 1: Header Info:** Enter the Vendor name and a clear Description of the purchase. Indicate if a Contract is required or if it's DSRIP-related using the checkboxes. Click "Next".
    *   **Step 2: Delivery:** Provide the "Ship To" address/location and the "Need-By Date". Click "Next".
    *   **Step 3: Funding:** Click "Add Row" to add one or more funding lines. Select the appropriate Cost Center and Object Code from the dropdowns and enter the Amount for each line. The total request amount updates automatically. Ensure all lines are complete. Click "Next".
    *   **Step 4: Attachments:** Upload any supporting documents (quotes, justifications, etc.) using the attachments control. Click "Next".
    *   **Step 5: Review:** Carefully review the summary of all entered information. If correct, click "Submit". If changes are needed, use the "Back" button.
3.  **Confirmation:** Upon successful submission, you'll receive a confirmation message, and the request will enter the "Draft" phase (or proceed directly to "Pending Mgr" if configured). You will be navigated to the "My Requests" screen.

## Screen Guide

*   **Home (`scrMain`):** The main landing page. Provides buttons to navigate to key areas:
    *   `New Request`: Starts the request submission wizard.
    *   `My Requests`: Shows requests you submitted or created.
    *   `Pending Approvals`: Shows requests waiting for *your* approval action.
    *   `Admin Panel`: (Visible only to Admins) Access administrative functions.
*   **New Request (`scrNewRequest`):** Hosts the multi-step form (`cmpRequestForm`) for creating a new request.
*   **My Requests (`scrViewMyRequests`):**
    *   Displays a searchable list of requests you created or are the listed requestor for.
    *   Shows Request ID, Title, Total Amount, and current Phase (with visual indicators).
    *   You can search by ID or Title.
    *   Clicking a request in "Draft" phase navigates to the Edit screen.
    *   Clicking a request in any other phase navigates to the read-only Detail View screen.
*   **Pending Approvals (`scrViewPendingRequests`):**
    *   Displays a searchable list of requests currently assigned to *you* for approval.
    *   Shows Request ID, Title, Requestor, Amount, and Age (days pending).
    *   You can search by ID, Title, or Requestor.
    *   Provides "Approve" and "Deny" buttons for each request. Clicking these updates the request status and logs your decision.
*   **Edit Request (`scrEditRequest`):** Allows modification of requests that are still in the "Draft" phase. Uses the same multi-step form as "New Request".
*   **View Request Detail (`scrViewRequestDetail`):** Provides a read-only view of all details for a submitted request, including header information and funding lines. Accessed from the "My Requests" screen for non-Draft items.
*   **Admin Panel (`scrAdmin`):** (Admins Only)
    *   Provides direct links to open the underlying SharePoint lists (`Purchase Requests`, `Funding Lines`, `Approvals`).
    *   Includes an action to trigger the "Archive Script" (a background process to handle old, completed requests).

## Approval Process Overview

1.  After submission, a request typically enters the "Pending Mgr" phase.
2.  An automated process assigns the request to the appropriate approver(s) based on the current phase (e.g., Requestor's Manager, DSRIP team, Contract team, etc.).
3.  Approvers will see assigned requests on their "Pending Approvals" screen.
4.  Approvers review the request details (they may need to navigate to the Detail View) and click "Approve" or "Deny".
5.  **Approve:** The request moves to the next phase in the workflow (e.g., Pending DSRIP -> Pending Contract -> Pending Budget -> Pending Purchasing -> Approved). The system assigns it to the next approver(s).
6.  **Deny:** The request moves to the "Denied" phase and the process stops.
7.  Requestors can track the progress of their requests on the "My Requests" screen by observing the "Phase" column.

## Admin Functions

Administrators use the "Admin Panel" screen to:
*   Directly access and manage the data in the SharePoint lists if needed.
*   Initiate the process for archiving old, completed requests via the "Run Archive Script" button.

## Support

If you encounter issues or have questions about using the APH Purchasing Power App, please contact [Placeholder - Insert APH Support Contact/Process Here].