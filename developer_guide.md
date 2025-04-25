# APH Purchasing Power App - Developer Implementation Guide

This guide provides instructions for setting up the necessary backend (SharePoint lists) and understanding the structure and logic of the APH Purchasing Power App based on the source code provided in the `src` directory. Following these steps chronologically should allow a developer to reconstruct and deploy the application.

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
    |              |               |                     | *Choices:* `Approved`, `Denied` |         |
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
*   The `CurrentApprover` column in `Purchase Requests` is critical and **must be populated by the "Assign Approver" Power Automate flow**.

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

1.  **Assign Approver Flow:** Triggered on `Purchase Requests` item create/modify. Updates the `CurrentApprover` column based on the `Phase`. **This is essential for the "Pending Approvals" screen to function.**
2.  **Archive Requests Flow:** Triggered manually from the app's Admin screen. Handles archiving logic.

Ensure these flows are included in the same Power Platform solution as the app and use the SharePoint connection reference.

## 4. Building the App from Source (`src`)

1.  **Enable YAML Source Files:** Ensure the Power Platform environment settings allow YAML source file format for Canvas Apps.
2.  **Use PAC CLI:** Use the Power Platform CLI (`pac`) to pack the source code from the `src` directory into an `.msapp` file.
    ```bash
    pac canvas pack --sources src --msapp MyPurchasingApp.msapp
    ```
3.  **Import `.msapp`:** Import the generated `MyPurchasingApp.msapp` file into your Power Platform environment via the Maker Portal or `pac canvas create --msapp ...`.
4.  **Connect Data Sources:** Open the imported app in the Studio. Verify/establish connections to the SharePoint lists using the connection reference. The `DataSources.json` file defines the structure, but the connection needs to be active.
5.  **Add Flows:** Add the "Archive Requests Flow" to the app via the Power Automate pane in the Studio so it can be called from the Admin screen. The "Assign Approver" flow runs automatically in the background and doesn't need to be explicitly added *to the app*.
6.  **Publish:** Publish the app.

## 5. Component Details

These reusable components form the core UI elements.

**`cmpHeader`**
*   **Purpose:** Displays the standard page header with logo, title, and user info.
*   **Inputs:** `PageTitle` (Text)
*   **Logic:** Uses `User()` function for user details. Displays input `PageTitle`.
*   **Data:** Reads `User()` info.

**`cmpHeaderMenu`**
*   **Purpose:** Provides the hamburger menu icon and flyout navigation panel.
*   **Inputs:** `IsOpen` (Boolean - *Currently unused, uses internal `varMenuOpen`*), `OnNavigate` (Behavior - *Action to perform on link selection*)
*   **Logic:** Uses internal variable `varMenuOpen` to toggle visibility of the panel (`ctlMenuPanel`) and link gallery (`galLinks`). The gallery items define navigation targets. Admin link visibility depends on `gblIsAdmin` (set in `App.OnStart`). Calls the `OnNavigate` behavior property when a link is selected.
*   **Data:** Reads `gblIsAdmin`.

**`cmpFundingLineEntry`**
*   **Purpose:** Represents a single row in the funding lines grid within the request form.
*   **Inputs:** `FundingLineRecord` (Record - *Passed in via gallery `ThisItem`*)
*   **Outputs:** `FundingLineRecord` (Record - *Contains the current values from the inputs within the component*)
*   **Logic:** Uses ComboBoxes for `CostCenter` and `ObjectCode` (bound to SharePoint list choices), TextInput for `Amount`. Remove icon (`icnRemove`) removes the corresponding item from the parent screen's `colFundingLines` collection. Sets `varChanged` (local context variable) on input changes (*Note: `varChanged` doesn't seem to be used elsewhere in the provided code*).
*   **Data:** Reads `Choices('Funding Lines'.CostCenter)`, `Choices('Funding Lines'.ObjectCode)`. Modifies `colFundingLines` (parent context).

**`cmpRequestForm`**
*   **Purpose:** Multi-step wizard for creating or editing purchase requests.
*   **Inputs:** `Mode` ("New" or "Edit"), `Request` (Record - *Existing request data for Edit mode*)
*   **Outputs:** `OnCompleted` (Behavior - *Fired after successful save, passes the saved/created record*)
*   **Logic:**
    *   `OnVisible`: Initializes step (`varStep`), defines wizard steps (`colWizardSteps`), loads existing data (`varHeader`, `colFundingLines`) in Edit mode, or clears collections in New mode.
    *   **Progress Bar (`galProgressBar`):** Visual indicator based on `varStep` and `colWizardSteps`.
    *   **Steps (`grpStep1` - `grpStep5`):** Groups of controls, visibility controlled by `varStep`. Inputs capture Vendor, Description, Flags, Ship To, Need By Date.
    *   **Step 3 (Funding):** Uses `galFundingLines` (containing `cmpFundingLineEntry` components) bound to `colFundingLines`. "Add Row" button adds a blank record to `colFundingLines`. Displays live total (`lblFundingTotalValue`) using `Sum(colFundingLines, Amount)`.
    *   **Step 4 (Attachments):** Standard Attachments control.
    *   **Step 5 (Review):** Displays an HTML summary (`lblReviewSummary`) of entered data.
    *   **Navigation (`btnBack`, `btnNext`):** Increment/decrement `varStep`. `btnNext` has `DisplayMode` logic for basic validation per step.
    *   **Submit (`btnSubmit`):**
        *   Aggregates data into `_headerPayload`.
        *   Uses `IfError`/`Patch` to save the main request to `Purchase Requests` (creates new or updates existing based on `Mode`). Stores result in `varParentResult`.
        *   If parent save succeeds:
            *   Uses `IfError`/`RemoveIf` to clear existing funding lines for the request.
            *   Uses `IfError`/`ForAll`/`Patch` to save items from `colFundingLines` to the `Funding Lines` list, linking via `RequestID`.
        *   Provides notifications (`Notify`) based on success/failure of patches.
        *   Calls the `OnCompleted` output behavior on full success.
*   **Data:** Reads/Writes `Purchase Requests`, `Funding Lines`. Reads `Choices` for funding lines (via child component). Uses local collections `colFundingLines`, `colWizardSteps`.

## 6. App `OnStart` Logic (`App.fx.yaml`)

*   **Purpose:** Initializes global settings and data once per user session.
*   **Logic:**
    1.  **Admin Check:** Looks up the current user's email in the `Purchasing Admins` list. Sets `gblIsAdmin` (Boolean) accordingly.
    2.  **Phase Ordering:** Creates `colPhaseOrder` collection, mapping phase names (Text) to a numerical order (Number). Used for sorting or conditional logic based on phase progression (e.g., in `scrViewMyRequests`, `scrViewPendingRequests`).
    3.  **Accessibility:** Initializes `gblLiveRegionText` (used for screen reader announcements - *implementation not shown in provided files but setup is standard*).
    4.  **Navigation:** Navigates to the initial screen (`scrMain`).
*   **Data:** Reads `Purchasing Admins`. Creates global variables/collections (`gblIsAdmin`, `colPhaseOrder`, `gblLiveRegionText`).

## 7. Screen Details

**`scrMain`**
*   **Purpose:** Landing/home screen providing navigation options.
*   **Logic:** Contains buttons that use `Navigate` to go to other screens (`scrNewRequest`, `scrViewMyRequests`, `scrViewPendingRequests`, `scrAdmin`). Admin button visibility depends on `gblIsAdmin`.
*   **Data:** Reads `gblIsAdmin`.

**`scrNewRequest`**
*   **Purpose:** Hosts the request form for creating new requests.
*   **Logic:** Embeds `cmpRequestForm` with `Mode="New"`. Navigates to `scrViewMyRequests` via the component's `OnCompleted` behavior.
*   **Data:** Interacts with data sources via `cmpRequestForm`.

**`scrEditRequest`**
*   **Purpose:** Hosts the request form for editing existing *Draft* requests.
*   **Logic:**
    *   `OnVisible`: Retrieves the `requestId` passed via navigation parameters (`Param("requestId")`). Looks up the corresponding record in `Purchase Requests` and stores it in `varSelectedRequest`.
    *   Embeds `cmpRequestForm` with `Mode="Edit"` and passes `varSelectedRequest` to its `Request` property.
    *   Navigates to `scrViewMyRequests` via the component's `OnCompleted` behavior.
*   **Data:** Reads `Purchase Requests`. Interacts further via `cmpRequestForm`.

**`scrViewMyRequests`**
*   **Purpose:** Displays a list of requests created by or requested by the current user.
*   **Logic:**
    *   `txtSearchMyRequests`: Filters the gallery based on input text (`varMyRequestsSearchTerm`).
    *   `galMyRequests`:
        *   `Items`: Filters `Purchase Requests` where `CreatedBy.Email` or `Requestor.Email` matches the current user, applies the search term (`varMyRequestsSearchTerm`), and sorts by ID descending.
        *   Displays key fields (ID, Title, Amount, Phase).
        *   Uses `conPhaseIndicator` (Container with Icon and Label) for visual phase status, colored based on phase.
        *   `OnSelect`: Navigates to `scrEditRequest` if the selected item's `Phase` is "Draft", otherwise navigates to `scrViewRequestDetail`.
    *   `lblEmpty`: Displays context-aware message when the gallery is loading or empty.
*   **Data:** Reads `Purchase Requests`. Uses `colPhaseOrder` (from `App.OnStart`) for phase indicator styling.

**`scrViewPendingRequests`**
*   **Purpose:** Displays requests awaiting the current user's approval. **Relies heavily on the "Assign Approver" Power Automate flow.**
*   **Logic:**
    *   `txtSearchPendingRequests`: Filters the gallery based on input text (`varPendingSearchTerm`).
    *   `galPending`:
        *   `Items`: Filters `Purchase Requests` where the current user's email is present in the `CurrentApprover` Person column (multi-select), applies the search term, and sorts by ID ascending.
        *   Displays key fields (ID, Title, Requestor, Amount, Age). Highlights age if > 7 days.
        *   `btnApprove`:
            *   Determines the next phase using `Switch` logic based on current phase and flags (`DSRIPFlag`, `ContractFlag`), referencing `colPhaseOrder`.
            *   Uses `IfError`/`Patch` to update the request's `Phase` and clear `CurrentApprover`.
            *   Uses `IfError`/`Patch` to log the approval action in the `Approvals` list.
            *   Provides notifications.
        *   `btnDeny`:
            *   Uses `IfError`/`Patch` to set the request's `Phase` to "Denied" and clear `CurrentApprover`.
            *   Uses `IfError`/`Patch` to log the denial action in the `Approvals` list.
            *   Provides notifications.
    *   `lblEmpty`: Displays context-aware message when the gallery is loading or empty.
*   **Data:** Reads `Purchase Requests`, `Approvals`. Uses `colPhaseOrder`. **Critically depends on the `CurrentApprover` column being correctly populated by the external flow.**

**`scrViewRequestDetail`**
*   **Purpose:** Displays a read-only summary of a selected request.
*   **Logic:**
    *   `OnVisible`: Retrieves `requestId` via `Param()`. Looks up the request (`varDetailRequest`) and its funding lines (`colDetailFundingLines`). Handles missing `requestId`.
    *   Displays various request fields using labels.
    *   Uses a gallery (`galDetailFundingLines`) to show funding lines read-only.
    *   Includes a "Back" button (`btnBackDetail`) using `Back()`.
    *   *Placeholder comments exist for potentially showing attachments.*
*   **Data:** Reads `Purchase Requests`, `Funding Lines`.

**`scrAdmin`**
*   **Purpose:** Provides administrative links and actions.
*   **Logic:**
    *   Visibility controlled by `gblIsAdmin`.
    *   `galAdminLinks`: Displays links. Uses `Launch()` to open SharePoint list views directly. For "Run Archive Script", it calls the `ArchiveRequestsFlow` Power Automate flow using `IfError` for feedback.
*   **Data:** Reads `gblIsAdmin`. Interacts with Power Automate (`ArchiveRequestsFlow`). Uses Environment Variables for constructing SharePoint URLs.

## 8. Deployment & Final Steps

*   **Solution Packaging:** Ensure the Canvas App, Environment Variables, Connection Reference(s), and Power Automate flows are all included in a single Power Platform Solution for deployment between environments (Dev, Test, Prod).
*   **Flow Activation:** Ensure flows are turned on in the target environment after deployment.
*   **Environment Variable Values:** Set the correct *Current Values* for all Environment Variables in each target environment.
*   **Sharing:** Share the Canvas App with appropriate end-users and admins. Ensure users have the necessary SharePoint permissions.

This guide provides a detailed walkthrough. Remember to test thoroughly, especially the approval routing logic dependent on the Power Automate flow and the error handling scenarios.
