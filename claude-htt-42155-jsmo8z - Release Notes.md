# claude/htt-42155-jsmo8z – Release Notes

## Requirements

Spex users were unable to close Spex Opportunities when the Product Mix included the Product Code 'Exhibitor.' Automation attempted to add that Product as an Opportunity Line Item, which failed because the Product only existed in the Standard Price Book, not the 'Spex Price Book - For Invoicing' set on the Opportunity. The error message shown to users did not reflect the actual cause. The requirement was to resolve the root cause so Spex Opportunities close successfully, with the correct Price Book and Opportunity Line Item set automatically, without automation that depends on hard-coded IDs or fragile references such as Record Type Name.

## Release Notes

The following changes were made to resolve the issue and remove fragile, hard-coded logic from the Spex Price Book automation:

- The 'Exhibitor' Product Code has been deactivated.
- Added a filter to the Product field on Product Mix (`Product_Mix__c.Product__c`) to require `Product2.IsActive = true`.
- Retired the Default Price Book flow by deploying an empty Flow Definition with no active version — it had hard-coded Price Book IDs and needed to be fully deactivated to safely update the Custom Metadata Type record it depended on.
- Removed the Price Book ID field (`Price_Book_ID__c`) from the Price Book Selector Custom Metadata Type — IDs should never be hard-coded.
- Added the Opportunity Record Type Developer Name field (`Opportunity_Record_Type_Developer_Name__c`) to the Price Book Selector CMDT as a safer match key than the Record Type Name/Label, which can be changed.
- Deactivated (Status = Obsolete) the `[OPPORTUNITY]-Spex-LF01-Updated` flow — it was unnecessarily setting the Price Book, had no decision logic, hard-coded IDs, referenced Record Type Name instead of Developer Name, and surfaced a custom error message unrelated to the actual error.
- Created the **Opportunity - Create - Set Price Book** flow (runs on Opportunity Create, before save). It matches the new Opportunity's Record Type Developer Name against the Price Book Selector CMDT and sets the Price Book accordingly. Currently only fires for Record Type Developer Name = `New_Spex_Opportunity`, but was built to scale to additional Record Types by adding more CMDT records.
- Created the **Opportunity - Update - Won Spex Opp** flow (runs on Opportunity Update, after save, only when the record newly meets criteria). When a `New_Spex_Opportunity` Opportunity is marked Closed Won, it looks up the Opportunity's Product Mix, finds the matching Price Book Entry (by Product, the Opportunity's Price Book, and Currency), and creates an Opportunity Line Item. If no Product Mix or no matching Price Book Entry is found, no Opportunity Line Item is created and no error is raised; a fault email is sent to tech@hansonwade.com only if the Line Item creation itself throws an error.

## Acceptance Criteria

To validate the fix, run through the following test cases:

**Test Case 1 — Happy path: Price Book auto-set and Line Item created**
Setup: Create a new Opportunity with Record Type = New Spex Opportunity (Developer Name `New_Spex_Opportunity`), as a user who does NOT have `ByPassLF__c` checked.
Steps:
1. Save the Opportunity and confirm `Pricebook2Id` is automatically set to "Spex Price Book - For Invoicing" immediately on creation.
2. Add a Product Mix record related to the Opportunity, selecting an Active Product. Confirm 'Exhibitor' does not appear as a selectable option.
3. Enter an Amount on the Opportunity.
4. Change Stage to Closed Won and Save.
Expected Result: Save completes with no errors. An Opportunity Line Item is created with `Product2Id` = the Product Mix's Product, `PricebookEntryId` matching that Product + the Opportunity's Price Book + Currency, `Quantity` = 1, and `UnitPrice` = the Opportunity Amount.

**Test Case 2 — No Product Mix: closes without error, no Line Item**
Setup: Same as Test Case 1, but do not add a Product Mix record.
Steps: Set Stage to Closed Won and Save.
Expected Result: Save completes with no errors. No Opportunity Line Item is created (the Product Mix lookup finds nothing and the flow exits quietly).

**Test Case 3 — Legacy 'Exhibitor' Product Mix: fixed after data load**
Setup: Use (or simulate) an open Opportunity from before this release that has a Product Mix record with Product Code 'Exhibitor'.
Steps:
1. Run the post-deployment data load to switch that Product Mix's Product to 'Exhibition Partner'.
2. Set the Opportunity Stage to Closed Won and Save.
Expected Result: Save completes with no errors, and an Opportunity Line Item is created correctly, same as Test Case 1.

**Test Case 4 — Legacy 'Exhibitor' Product Mix: data load skipped (known limitation)**
Setup: Same as Test Case 3, but skip the data load step.
Steps: Set Stage to Closed Won and Save.
Expected Result: Save completes with no errors, but no Opportunity Line Item is created — there is no Price Book Entry for the 'Exhibitor' product in the Spex Price Book.

**Test Case 5 — Bypass user is respected**
Setup: Run as a user with `ByPassLF__c = true`.
Steps:
1. Create a New Spex Opportunity.
2. Mark it Closed Won and Save.
Expected Result: Neither flow fires — the Price Book is not auto-set on create, and no Opportunity Line Item is created on Closed Won.

**Test Case 6 — Non-Spex Record Types are unaffected**
Setup: Create an Opportunity with a Record Type other than New Spex Opportunity.
Steps: Set Stage to Closed Won and Save.
Expected Result: Neither new flow fires — Price Book is not modified and no Opportunity Line Item is auto-created by this automation.

**Test Case 7 — Retired automation no longer runs**
Setup: In Setup, open Flows.
Steps:
1. Open the `[OPPORTUNITY]-Spex-LF01-Updated` flow and confirm Status = Obsolete.
2. Open the Default Price Book flow and confirm it has no active version (shows Inactive/no current version).
3. Close a Spex Opportunity and confirm the old custom error message no longer appears.
Expected Result: Both retired flows are confirmed inactive, and the previously reported error message is gone.

## Post Deployment Items

- Confirm the **Opportunity - Create - Set Price Book** and **Opportunity - Update - Won Spex Opp** flows are Active in the target org (deployed as Active).
- Confirm the Default Price Book flow has no active version in the target org (deployed via an empty Flow Definition) and that `[OPPORTUNITY]-Spex-LF01-Updated` shows Status = Obsolete.
- **Data load required:** for any open Opportunities with a Product Mix record whose Product Code is 'Exhibitor', switch the Product Mix to 'Exhibition Partner'. Without this data load, those Opportunities can still be closed without error, but no Opportunity Line Item will be created.
- **Repo cleanup:** `unpackaged/main/default/flows/Default_Price_Book.flow-meta.xml` is still tracked in source control even though the flow now has no active version in the org. Remove it from the repo (or add a destructive changes entry) so source control matches the org.
- Decide whether to remove the legacy, label-based `Opportunity_Record_Type__c` field (and its value and layout placement) on `Price_Book_Selector__mdt` — it is no longer referenced by any active automation now that `Opportunity_Record_Type_Developer_Name__c` is used instead.

## Component Manifest

### Github Branch

https://github.com/aaroncrear/HansonWadeDataSeparation/tree/claude/htt-42155-jsmo8z

| # | Component Type | Object | API Name | Label | Created/Updated/Deleted | Description |
|---|---|---|---|---|---|---|
| 1 | Custom Field | `Product_Mix__c` | `Product__c` | Product | Updated | Added lookup filter requiring `Product2.IsActive = true`, so inactive products (e.g. the retired 'Exhibitor' Product Code) can no longer be selected. |
| 2 | Flow | Opportunity | `Default_Price_Book` | Default Price Book | Deactivated | Deployed with no active Flow Definition version (see next row). Retired due to hard-coded Price Book ID; source file still present in repo — see Post Deployment Items for cleanup. |
| 3 | Flow Definition | Opportunity | `Default_Price_Book` | Default Price Book | Created | Empty Flow Definition (no `activeVersionNumber`) deployed to remove the active version of the Default Price Book flow. |
| 4 | Flow | Opportunity | `OPPORTUNITY_Spex_LF01_Updated` | [OPPORTUNITY]-Spex-LF01-Updated | Deactivated | Status changed from Active to Obsolete. Was unnecessarily setting the Price Book, had no decision logic, hard-coded IDs, referenced Record Type Name instead of Developer Name, and surfaced an unrelated custom error message. |
| 5 | Custom Metadata Type Field | `Price_Book_Selector__mdt` | `Price_Book_ID__c` | Price Book ID | Deleted | Removed — Price Book IDs should never be hard-coded. |
| 6 | Custom Metadata Type Field | `Price_Book_Selector__mdt` | `Opportunity_Record_Type_Developer_Name__c` | Opportunity Record Type Developer Name | Created | Safer match key than Record Type Name/Label; used by the Opportunity - Create - Set Price Book flow to find the correct CMDT record. |
| 7 | Layout | `Price_Book_Selector__mdt` | `Price_Book_Selector__mdt-Price Book Selector Layout` | Price Book Selector Layout | Updated | Replaced the Price Book ID field with the new Opportunity Record Type Developer Name field. |
| 8 | Custom Metadata | `Price_Book_Selector__mdt` | `Price_Book_Selector.Spex_Lookup` | Spex Lookup | Updated | Added value for `Opportunity_Record_Type_Developer_Name__c` = `New_Spex_Opportunity`; removed the `Price_Book_ID__c` value (field deleted). |
| 9 | Flow | Opportunity | `Opportunity_Create_Set_Price_Book` | Opportunity - Create - Set Price Book | Created | Runs on Opportunity Create (before save). Matches Record Type Developer Name against the Price Book Selector CMDT and sets the Price Book. Currently only fires for `New_Spex_Opportunity`; built to scale to more Record Types. |
| 10 | Flow | Opportunity | `Opportunity_Update_Won_Spex_Opp` | Opportunity - Update - Won Spex Opp | Created | Runs on Opportunity Update (after save, only when newly meeting criteria). When a `New_Spex_Opportunity` is marked Closed Won, creates an Opportunity Line Item from the related Product Mix and matching Price Book Entry. Sends a fault email to tech@hansonwade.com if Line Item creation errors. |
| 11 | Product (Data) | Product2 | N/A | Exhibitor | Deactivated | Product marked Inactive (`IsActive = false`) so it can no longer be selected on Product Mix. |
