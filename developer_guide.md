# APH Purchasing Power App - Developer Implementation Guide

This guide provides instructions for setting up the necessary backend (SharePoint lists) and understanding the structure and logic of the APH Purchasing Power App based on the source code provided in the `src1` directory. Following these steps chronologically should allow a developer to reconstruct and deploy the application.

## 1. Prerequisites: SharePoint List Setup

Create the following SharePoint lists in the target SharePoint site. The **Internal Name** is crucial for Power Apps/Automate integration if different from the Display Name. Use the specified column types.

**Target SharePoint Site:** The URL for this site will be stored in the `ENV_TENANT_URL` environment variable.

**List 1: Purchase Requests**

*   **Display Name:** `APH Purchase Requests` (or similar)
*   **Internal Name:** `Purchase Requests` (Matches `DataSources.json`)
*   **Minimum Columns:**

    | Display Name        | Internal Name         | Type                     | Notes / Required | Indexed |
    | :------------------ | :-------------------- | :----------------------- | :--------------- | :------ |
    | ID                  | `ID`                  | Auto-generated           | Yes              | Yes     |
    | Title               | `Title`               | Single line of text      | Yes (Default SP) |         |
    | Description         | `Description`         | Multiple lines of text   | Yes              |         |
    | Vendor              | `Vendor`              | Single line of text      | Yes              |         |
    | ContractFlag        | `ContractFlag`        | Yes/No (Boolean)         | Yes              |         |
    | DSRIPFlag           | `DSRIPFlag`           | Yes/No (Boolean)         | Yes              |         |
    | ShipTo              | `ShipTo`              | Single line of text      | Yes              |         |
    | NeedBy              | `NeedBy`              | Date Only                | Yes              |         |
    | Phase               | `Phase`               | Choice                   | Yes              | Yes     |
    |                     |                       |                          | *Choices:* `Draft`, `Pending Mgr`, `Pending DSRIP`, `Pending Contract`, `Pending Budget`, `Pending Purchasing`, `Approved`, `Denied`, `Archived` |         |
    | TotalAmount         | `TotalAmount`         | Currency                 | Yes              |         |
    | CurrentApprover     | `CurrentApprover`     | Person or Group          | **Yes**          | Yes     |
    |                     |                       |                          | *Allow multiple selections: Yes* |         |
    | **Revision Comments** | `RevisionComments`    | **Multiple lines of text** | No               |         |
    |                     |                       |                          | *(Plain Text)*   |         |
    | Created By          | `Author`              | Person or Group          | Yes (Default SP) |         |
    | Modified            | `Modified`            | Date and Time            | Yes (Default SP) |         |
    | Requestor           | `Requestor`           | Person or Group          | No (App sets this) |         |
    |                     |                       |                          | *Allow multiple selections: No* |         |
    | *Attachments*       | *Attachments*         | *Built-in*               | Yes (Default SP) |         |

**List 2: Funding Lines**

*   **Display Name:** `APH Funding Lines` (or similar)
*   **Internal Name:** `Funding Lines` (Matches `DataSources.json`)
*   **Minimum Columns:**

    | Display Name | Internal Name | Type                | Notes / Required | Indexed |
    | :----------- | :------------ | :------------------ | :--------------- | :------ |
    | ID           | `ID`          | Auto-generated      | Yes              | Yes     |
    | RequestID    | `RequestID`   | Number              | Yes              | Yes     |
    |              |               |                     | *Lookup target: `Purchase Requests` list, `ID` column* |         |
    | CostCenter   | `CostCenter`  | Choice              | Yes              |         |
    |              |               |                     | *Define relevant choices* |         |
    | ObjectCode   | `ObjectCode`  | Choice              | Yes              |         |
    |              |               |                     | *Define relevant choices* |         |
    | Amount       | `Amount`      | Currency            | Yes              |         |

**List 3: Approvals**

*   **Display Name:** `APH Approvals` (or similar)
*   **Internal Name:** `Approvals` (Matches `DataSources.json`)
*   **Minimum Columns:**

    | Display Name | Internal Name | Type                | Notes / Required | Indexed |
    | :----------- | :------------ | :------------------ | :--------------- | :------ |
    | ID           | `ID`          | Auto-generated      | Yes              | Yes     |
    | RequestID    | `RequestID`   | Number              | Yes              | Yes     |
    |              |               |                     | *Lookup target: `Purchase Requests` list, `ID` column* |         |
    | Approver     | `Approver`    | Single line of text | Yes              | Yes     |
    |              |               |                     | *Stores User().Email* |         |
    | Decision     | `Decision`    | Choice              | Yes              |         |
    |              |               |                     | *Choices:* `Approved`, `Denied`, `Revision Requested` |         |
    | Timestamp    | `Timestamp`   | Date and Time       | Yes              |         |

**List 4: Purchasing Admins**

*   **Display Name:** `APH Purchasing Admins` (or similar)
*   **Internal Name:** `Purchasing Admins` (Matches `DataSources.json`)
*   **Minimum Columns:**

    | Display Name | Internal Name | Type            | Notes / Required | Indexed |
    | :----------- | :------------ | :-------------- | :--------------- | :------ |
    | ID           | `ID`          | Auto-generated  | Yes              |         |
    | User         | `User`        | Person or Group | Yes              | Yes     |
    |              |               |                 | *Allow multiple selections: No* |         |

**Important:**
*   After creating the lists, **note down their GUIDs**. These will be needed for Environment Variables.
*   Ensure appropriate **permissions** are set on these lists for app users and admins.
*   The `CurrentApprover` column in `Purchase Requests` is critical and **must be populated by the "Assign Approver & Notify" Power Automate flow**.

## 2. Prerequisites: Environment Variables & Connections

The app relies on Environment Variables defined within the Power Platform solution.

*   **Create Connection Reference:** Create a SharePoint connection reference within your solution. Note its logical name (used for `SP_CONNECTION_ID`).
*   **Define Environment Variables:** Create the following Environment Variables within your solution:
    *   `ENV_TENANT_URL`: (Type: Text) SharePoint Site URL.
    *   `ENV_LIST_REQUESTS_ID`: (Type: Text) GUID of the `Purchase Requests` list.
    *   `ENV_LIST_FUNDINGLINES_ID`: (Type: Text) GUID of the `Funding Lines` list.
    *   `ENV_LIST_APPROVALS_ID`: (Type: Text) GUID of the `Approvals` list.
    *   `ENV_LIST_ADMINS_ID`: (Type: Text) GUID of the `Purchasing Admins` list.
    *   `SP_CONNECTION_ID`: (Type: Text) Logical name of the SharePoint Connection Reference created above.
*   **Set Values:** Set the *Current Value* for each Environment Variable in the target environment before running the app. The `defaultValue` fields in `EnvironmentVariables.json` are placeholders in the source code.

## 3. Prerequisites: Power Automate Flows

Set up the two Power Automate flows as detailed in `.docs/power_automate_requirements.md`:

1.  **Assign Approver & Notify Flow:** Triggered on `Purchase Requests` item create/modify. Updates the `CurrentApprover` column based on the `Phase` and sends email notifications for approval or revision requests. **This is essential for the "Pending Approvals" screen and user notifications.**
2.  **Archive Requests Flow:** Triggered manually from the app's Admin screen. Handles archiving logic.

Ensure these flows are included in the same Power Platform solution as the app and use the SharePoint connection reference.

## 4. Building the App from Source (`src1`)

1.  **Enable YAML Source Files:** Ensure the Power Platform environment settings allow YAML source file format for Canvas Apps.
2.  **Use PAC CLI:** Use the Power Platform CLI (`pac`) to pack the source code from the `src1` directory into an `.msapp` file.
    ```bash
    pac canvas pack --sources src1 --msapp MyPurchasingApp.msapp
    ```
3.  **Import `.msapp`:** Import the generated `MyPurchasingApp.msapp` file into your Power Platform environment via the Maker Portal or `pac canvas create --msapp ...`.
4.  **Connect Data Sources:** Open the imported app in the Studio. Verify/establish connections to the SharePoint lists using the connection reference. The `DataSources.json` file defines the structure, but the connection needs to be active.
5.  **Add Flows:** Add the "Archive Requests Flow" to the app via the Power Automate pane in the Studio so it can be called from the Admin screen. The "Assign Approver & Notify" flow runs automatically in the background.
6.  **Publish:** Publish the app.

## 5. Component Details

These reusable components form the core UI elements.

**`cmpHeader`**
*   (No changes from previous description)

**`cmpHeaderMenu`**
*   (No changes from previous description)

**`cmpFundingLineEntry`**
*   (No changes from previous description)

**`cmpRequestForm`**
*   **Purpose:** Multi-step wizard for creating or editing purchase requests. Now includes display for revision comments.
*   **Inputs:** `Mode` ("New" or "Edit"), `Request` (Record - *Existing request data for Edit mode*)
*   **Outputs:** `OnCompleted` (Behavior - *Fired after successful save, passes the saved/created record*)
*   **Logic:**
    *   `OnVisible`: Initializes step (`varStep`), defines wizard steps (`colWizardSteps`), loads existing data (`varHeader`, `colFundingLines`) in Edit mode, or clears collections in New mode.
    *   **Progress Bar (`galProgressBar`):** Visual indicator based on `varStep` and `colWizardSteps`.
    *   **Steps (`grpStep1` - `grpStep5`):** Groups of controls, visibility controlled by `varStep`. Inputs capture Vendor, Description, Flags, Ship To, Need By Date.
    *   **Revision Comments Display (Step 1):** A label is added to Step 1, visible only in Edit mode if `Request.RevisionComments` is not blank, displaying the comments received from the approver.
    *   **Step 3 (Funding):** Uses `galFundingLines` (containing `cmpFundingLineEntry` components) bound to `colFundingLines`. "Add Row" button adds a blank record to `colFundingLines`. Displays live total (`lblFundingTotalValue`) using `Sum(colFundingLines, Amount)`.
    *   **Step 4 (Attachments):** Standard Attachments control.
    *   **Step 5 (Review):** Displays an HTML summary (`lblReviewSummary`) of entered data.
    *   **Navigation (`btnBack`, `btnNext`):** Increment/decrement `varStep`. `btnNext` has `DisplayMode` logic for basic validation per step.
    *   **Submit (`btnSubmit`):**
        *   Aggregates data into `_headerPayload`. **Crucially, it also clears the `RevisionComments` field when saving**, ensuring comments are only shown for the current revision cycle.
        *   Uses `IfError`/`Patch` to save the main request to `Purchase Requests`. Stores result in `varParentResult`.
        *   If parent save succeeds: Clears existing funding lines, saves new funding lines, provides notifications, and calls `OnCompleted`. Includes error handling via `IfError`.
*   **Data:** Reads/Writes `Purchase Requests`, `Funding Lines`. Reads `Choices` for funding lines (via child component). Uses local collections `colFundingLines`, `colWizardSteps`.

## 6. App `OnStart` Logic (`App.fx.yaml`)

*   (No changes from previous description)

## 7. Screen Details

**`scrMain`**
*   (No changes from previous description)

**`scrNewRequest`**
*   (No changes from previous description)

**`scrEditRequest`**
*   **Purpose:** Hosts the request form for editing existing *Draft* requests (including those sent back for revision).
*   **Logic:**
    *   `OnVisible`: Retrieves `requestId`. Looks up the request (`varSelectedRequest`).
    *   Embeds `cmpRequestForm` with `Mode="Edit"` and passes `varSelectedRequest`. The form itself will display `RevisionComments` if present.
    *   Navigates to `scrViewMyRequests` via `OnCompleted`.
*   **Data:** Reads `Purchase Requests`. Interacts further via `cmpRequestForm`.

**`scrViewMyRequests`**
*   (No changes from previous description, still navigates to Edit for Draft items)

**`scrViewPendingRequests`**
*   **Purpose:** Displays requests awaiting the current user's approval. Handles the updated "Deny" (Send Back for Revision) logic.
*   **Logic:**
    *   `txtSearchPendingRequests`: Filters the gallery.
    *   `galPending`:
        *   `Items`: Filters `Purchase Requests` where user is in `CurrentApprover`, applies search, sorts.
        *   Displays key fields.
        *   `btnApprove`: Determines next phase, patches request (`Phase`, clears `CurrentApprover`), logs to `Approvals` (`Decision: "Approved"`), notifies. Includes `IfError`.
        *   **`btnDeny`:**
            *   **Shows a Popup:** A new popup control (defined on this screen) is made visible. This popup contains a TextInput (`inpRevisionComments`) for the approver's feedback and Confirm/Cancel buttons.
            *   **Popup Confirm Button `OnSelect`:**
                *   Uses `IfError`/`Patch` to update the request: Sets `Phase: "Draft"`, sets `RevisionComments: inpRevisionComments.Text`, clears `CurrentApprover`.
                *   Uses `IfError`/`Patch` to log to `Approvals` (`Decision: "Revision Requested"`).
                *   Provides notifications.
                *   Hides the popup.
    *   `lblEmpty`: Displays context-aware message.
*   **Data:** Reads `Purchase Requests`, `Approvals`. Uses `colPhaseOrder`. **Critically depends on the `CurrentApprover` column being correctly populated by the external flow.**

**`scrViewRequestDetail`**
*   **Purpose:** Displays a read-only summary. Now includes Revision Comments if present.
*   **Logic:**
    *   `OnVisible`: Retrieves `requestId`. Looks up request (`varDetailRequest`) and funding lines (`colDetailFundingLines`).
    *   Displays various request fields including `RevisionComments` (conditionally visible if not blank).
    *   Uses a gallery (`galDetailFundingLines`) for funding lines.
    *   Includes a "Back" button.
*   **Data:** Reads `Purchase Requests`, `Funding Lines`.

**`scrAdmin`**
*   (No changes from previous description regarding logic, but relies on updated Archive flow)

## 8. Deployment & Final Steps

*   **Solution Packaging:** Ensure the Canvas App, Environment Variables, Connection Reference(s), and **updated** Power Automate flows are included.
*   **Flow Activation & Updates:** Ensure the updated flows are deployed and turned on.
*   **Environment Variable Values:** Set current values.
*   **Sharing:** Share app and ensure permissions.

This guide provides a detailed walkthrough for the updated requirements. Remember to implement the popup for denial comments on `scrViewPendingRequests` and the display logic for comments in `cmpRequestForm` and `scrViewRequestDetail`. Test the revision loop and notifications thoroughly.