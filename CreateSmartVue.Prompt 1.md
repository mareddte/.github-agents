Use the attached SmartVue_Input.xlsx workbook as metadata source.

Follow ALL instructions from CreateSmartVue.agent.md.

Read:

1. SmartVueInfo
2. SearchFields
3. ResultFields
4. ViewDefinition
5. Menu
6. DataSourceDefinition

=====================================================
DATA SOURCE STRATEGY (MANDATORY)
=====================================================

Determine behavior from SmartVueInfo sheet.

Columns:

DataSourceMode
ExistingViewName
ExistingSPName
CreateView
CreateSP

Rules:

CASE 1
If ExistingViewName contains a value:

- Use that ViewName in TRK_LAYOUT SELECTVIEW.
- Use that ViewName in FROMQUERY.
- DO NOT generate CREATE VIEW.
- DO NOT modify the existing View.

Example:

ExistingViewName = MDV_AgentDetail

Result:
Use MDV_AgentDetail only.

-----------------------------------------------------

CASE 2

If ExistingSPName contains a value:

- Use ExistingSPName in SELECTSP.
- DO NOT generate CREATE PROCEDURE.
- DO NOT modify existing procedure.

Example:

ExistingSPName = USP_Agent_Search

Result:
Use USP_Agent_Search only.

-----------------------------------------------------

CASE 3

If CreateView = Yes

- Generate a new MDV View.
- Read join conditions from DataSourceDefinition sheet.
- View name comes from ViewDefinition sheet.
- Generate CREATE VIEW statement.

-----------------------------------------------------

CASE 4

If CreateSP = Yes

- Generate a new Search Stored Procedure.
- Read filters and joins from DataSourceDefinition sheet.
- Procedure name:

USP_<SubCategoryName>_Search

- Generate full CREATE PROCEDURE statement.

-----------------------------------------------------

CASE 5

If

ExistingViewName populated
AND
CreateView = No

Then:
NEVER create or alter a View.

-----------------------------------------------------

CASE 6

If

ExistingSPName populated
AND
CreateSP = No

Then:
NEVER create or alter a Procedure.

=====================================================
SMARTVUE GENERATION
=====================================================

Generate:

- TRK_CATEGORY
- TRK_SUBCATEGORY
- Delete/Recreate Logic
- TRK_SUBCATEGORYAPPLICATION
- SEC_COMPONENT
- SEC_ACCESSLEVELRIGHT
- Result Layout
- Result TableFields
- Width Properties
- Search Layout
- Search TableFields
- MDV View (only when CreateView=Yes)
- Search Procedure (only when CreateSP=Yes)
- TRK_SUBMENU

=====================================================
VIEW GENERATION RULES
=====================================================

Only if CreateView=Yes.

Use:

- ViewDefinition sheet
- DataSourceDefinition sheet

to generate:

CREATE VIEW dbo.<ViewName>

using supplied JOINs and WHERE conditions.

=====================================================
STORED PROCEDURE RULES
=====================================================

Only if CreateSP=Yes.

Generate:

CREATE PROCEDURE USP_<SubCategoryName>_Search

using:

- SearchFields metadata
- DataSourceDefinition joins
- DataSourceDefinition conditions

=====================================================
OUTPUT
=====================================================

Return only executable SQL.

No explanation.

No placeholders.

No samples.

Output file name:

<SubCategoryName>.txt