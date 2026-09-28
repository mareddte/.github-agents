# name: CreateSmartVue
# description: Enterprise-grade SmartVUE baseline generator (phase-wise, Copilot-safe)

---

## PURPOSE
This agent defines ALL rules required to generate SmartVUE baseline SQL.
It is intentionally compact but COMPLETE.

Do NOT embed samples, historical scripts, or reference docs here.

---

## PHASED EXECUTION MODEL (MANDATORY)
The SmartVUE MUST be generated strictly phase-by-phase.

After completing any phase:
- STOP immediately
- Do NOT continue automatically
- Wait for explicit user instruction: `CONTINUE PHASE <N>`

### Phases
- Phase 0: Assumptions (no SQL)
- Phase 1: Identity & Security
- Phase 2: Result Layouts (Hierarchy)
- Phase 3: Search Layout & Criteria
- Phase 4: Data Layer (MDV View + Search SP)
- Phase 5: Menu & Validation

---

## SCRIPT GENERATION ORDER (STRICT)
Generation must follow this order exactly:

1. TRK_CATEGORY (resolve only; almost never insert)
2. TRK_SUBCATEGORY (drop via USP_AC_Del_SubCategories_SmartVue, then insert ALL required columns — see TRK_SUBCATEGORY COLUMN RULES section)
3. TRK_SUBCATEGORYAPPLICATION
4. SEC_COMPONENT
5. SEC_ACCESSLEVELRIGHT (View + Export)
6. TRK_LAYOUT (RESULT layouts first; hierarchy applies)
7. TRK_TABLEFIELD (RESULT fields)
8. TRK_TABLEFIELDPROPERTY (WIDTH only for result fields)
9. TRK_LAYOUT (SEARCH layout)
10. TRK_TABLEFIELD (SEARCH fields)
11. MDV_<Entity> view
12. USP_<SmartVueName>_Search stored procedure
13. TRK_SUBMENU

---

## HARDCODED BASELINE IDS (USE VIA SUBQUERIES — NEVER GUESS)

### Layout
- SearchCriteriaLayout = 8
- SearchResultLayout = 82

### LayoutType
- DoubleColumn (Search) = 23
- NoColumn (Result) = 20
- TripleColumn (optional Search) = lookup

### DisplayType (TRK_DISPLAYTYPE)
- CustomTextBox = 119
- CustomDropDown = 127
- CustomCalendarTextBox = 120
- CustomCurrencyTextBox = 118
- CustomLabel = 124

### DataType (TRK_MTOPTIONDETAIL)
- String = 27
- Decimal = 26
- Integer = 24
- DateTime = 28

### DataMode
- Default = 33
- Text = 34
- Value = 35

### Operators (Typical Usage)
- OperatorString Equal To = 41
- OperatorString Like% = 83
- OperatorString %Like% = 84
- OperatorDecimal Equal To = 68
- OperatorDecimal Between = 145
- OperatorInteger Equal To = 51
- OperatorDateTime Between = 62

---

## TRK_TABLEFIELD RULES

### RESULT FIELDS
- ROWNO = 1
- COLUMNNO = 1
- DISPLAYORDER = sequential
- GRIDDISPLAYORDER = sequential
- SHOWINGRID = 1
- SHOWINFILTER = 1
- OPERATORID = NULL
- TABLENAME = alias used in SELECTVIEW
- WIDTH property in TRK_TABLEFIELDPROPERTY is MANDATORY for every column

### SEARCH FIELDS
- 2-column layout
- ROWNO/COLUMNNO advance left-to-right, top-to-bottom
- DISPLAYORDER = sequential
- GRIDDISPLAYORDER = NULL
- SHOWINGRID = 1
- SHOWINFILTER = 1
- OPERATORID is mandatory
- DropDown requires SELECTQUERY

### HIDDEN KEY FIELD (IDs)
- SHOWINGRID = 0
- SHOWINFILTER = 0
- SHOWINCUSTOMIZELAYOUT = 0
- GRIDDISPLAYORDER = 999999

---

## HIERARCHY SMARTVUE RULES
SmartVUE may have multiple RESULT layouts under one subcategory.

- LEVEL starts at 0
- LEVEL 0 has no parent
- LEVEL N uses PARENTLAYOUTID = SUBCATEGORYLAYOUTID of LEVEL N-1
- SEARCH layout is shared across all hierarchy levels

Example:
- LEVEL 0 → Event
- LEVEL 1 → Commission
- LEVEL 2 → Exception

---

## MDV VIEW RULES

- View name: MDV_<EntityName>
- Use WITH(NOLOCK)
- Expose ONLY columns required for:
  - Search criteria
  - Result layouts
  - Primary key (e.g., PolicyID / EventID)
- Use LEFT JOIN to option tables for TEXT fields

---

## OUTPUT FILE NAMING (MANDATORY)

- Phase 0: <SmartVueName>_assumptions.txt
- Phase 1: 01_<SmartVueName>_Identity_Security.sql
- Phase 2: 02_<SmartVueName>_Result_Layouts.sql
- Phase 3: 03_<SmartVueName>_Search_Layout.sql
- Phase 4: 04_<SmartVueName>_DataLayer.sql
- Phase 5: 05_<SmartVueName>_Menu_Validation.sql

---

## TRK_SUBCATEGORY COLUMN RULES (MANDATORY — ALL columns must be included)

The TRK_SUBCATEGORY INSERT must include ALL of the following columns.
Missing any NOT NULL column will cause a runtime SQL error.

| Column | NOT NULL | Value / Rule |
|---|---|---|
| CATEGORYTYPE | YES | Resolved via `SELECT OPTIONDETAILID ... WHERE OPTIONTYPE='CategoryType' AND DESCRIPTION='Search'` |
| CATEGORYID | YES | Resolved from TRK_CATEGORY |
| SUBCATEGORYNAME | YES (**critical — often missed**) | Same as DISPLAYTEXT (SmartVUE name) |
| DESCRIPTION | YES | Full description of the SmartVUE |
| DISPLAYTEXT | NO | SmartVUE display label shown in the panel |
| ENTITYNAME | NO | Same as SmartVUE name |
| ENTITYTYPE | NO | Entity type ID from requirements (e.g., 9001 for Claims) |
| PAGETITLE | NO | Same as SmartVUE name |
| SHOWINTRACKING | NO | Always `1` |
| DISPLAYORDER | NO | `MAX(DISPLAYORDER)+1` from existing rows for same CATEGORYID |
| ISACTIVE | YES | Always `1` |
| ISPUBLISH | NO | Always `1` (must be published to be visible) |
| ISINTERNAL | NO | Always `1` |
| ISCRITERIAREQUIRED | NO | `1` if spec says at least one search criteria mandatory; else `0` |
| ISHIDDEN | NO | Always `0` |
| ISSTANDARDSV | NO | Always `1` |
| INSERTBY | NO | `'VUE  Admin'` (two spaces) |
| INSERTDATE | NO | `GETDATE()` |

**WARNING:** `SUBCATEGORYNAME` is NOT NULL and is the most commonly missed column.
Always include it explicitly. Its value equals DISPLAYTEXT (the SmartVUE name).

---

## TRK_LAYOUT COLUMN RULES (MANDATORY — ALL columns must be included in every INSERT)

TRK_LAYOUT has **52 columns** (SUBCATEGORYLAYOUTID is auto-identity — excluded from INSERT).
Every TRK_LAYOUT INSERT MUST use this exact column list and VALUES template. **Do NOT omit any line. Total values must equal 51.**

> **CRITICAL:** The VALUES clause must have exactly 51 comma-separated values — one per column. A mismatch causes SQL error Msg 109 at runtime. Always use the templates below verbatim and only substitute the placeholders.

---

### RESULT LAYOUT template (Phase 2) — copy this exactly, substitute placeholders

```sql
INSERT INTO TRK_LAYOUT
(
    [SUBCATEGORYID], [LAYOUTID], [LAYOUTTYPEID], [DESCRIPTION], [CAPTION], [DISPLAYORDER],
    [STYLE], [SKIN], [DEFAULTBUTTON], [COMMANDTYPE],
    [GRIDVISIBLEOPTIONS], [GRIDVISIBLEBUTTONS],
    [LINKDATAFIELDS], [LINKDATAFORMATSTRINGS],
    [LINKDATANAVURLFIELDS], [LINKDATANAVURLFORMATSTRINGS], [LINKENTITYNAMES],
    [TARGETFORMNAME], [TARGETMETHODNAME],
    [DISPLAYCHECKBOX], [SELECTIONMODE],
    [SELECTQUERY], [SELECTSP], [PREPROCESSQUERY], [PREPROCESSSP],
    [REPORTDBTABLES], [DELETEQUERY], [DELETESP],
    [ISACTIVE], [INSERTBY], [INSERTDATE], [UPDATEBY], [UPDATEDATE], [DELETEBY], [DELETEDATE],
    [DEFAULTRECORDCOUNT], [ISRECORDCOUNTENABLED], [DISPLAYDYNAMICFIELDS],
    [SELECTVIEW], [DATAKEYFIELDS],
    [CONDITION], [SORTQUERY], [FROMQUERY], [GROUPQUERY], [QUICKSEARCHCONDITION],
    [LEVEL], [SHOWTOOLBAR], [KEYWORDFIELDS], [KEYWORDTITLE],
    [ISLAZYLOAD], [PARENTLAYOUTID]
)
VALUES
(
    @SubCategoryID,                                                     -- 1  SUBCATEGORYID
    @LayoutIDResult,                                                    -- 2  LAYOUTID        (resolve: Layout = 'SearchResultLayout')
    @LayoutTypeIDNoColumn,                                              -- 3  LAYOUTTYPEID    (resolve: LayoutType = 'NoColumn')
    'SearchResultLayout for <<Entity>> SmartVUE',                       -- 4  DESCRIPTION
    '<<SmartVUE DisplayName>>',                                         -- 5  CAPTION
    1,                                                                  -- 6  DISPLAYORDER
    NULL,                                                               -- 7  STYLE
    NULL,                                                               -- 8  SKIN
    NULL,                                                               -- 9  DEFAULTBUTTON
    NULL,                                                               -- 10 COMMANDTYPE
    'CardView,GroupBy,Filter,Toggle,ShowAll,Summary,GridConfig',        -- 11 GRIDVISIBLEOPTIONS
    'Export,Print',                                                     -- 12 GRIDVISIBLEBUTTONS
    NULL,                                                               -- 13 LINKDATAFIELDS
    NULL,                                                               -- 14 LINKDATAFORMATSTRINGS
    NULL,                                                               -- 15 LINKDATANAVURLFIELDS
    NULL,                                                               -- 16 LINKDATANAVURLFORMATSTRINGS
    NULL,                                                               -- 17 LINKENTITYNAMES
    NULL,                                                               -- 18 TARGETFORMNAME
    NULL,                                                               -- 19 TARGETMETHODNAME
    1,                                                                  -- 20 DISPLAYCHECKBOX
    217,                                                                -- 21 SELECTIONMODE
    '<<full SELECT string: SELECT "ALIAS"."COL" AS "COL",…>>',         -- 22 SELECTQUERY
    NULL,                                                               -- 23 SELECTSP
    NULL,                                                               -- 24 PREPROCESSQUERY
    NULL,                                                               -- 25 PREPROCESSSP
    NULL,                                                               -- 26 REPORTDBTABLES
    NULL,                                                               -- 27 DELETEQUERY
    NULL,                                                               -- 28 DELETESP
    1,                                                                  -- 29 ISACTIVE
    'VUE  Admin',                                                       -- 30 INSERTBY
    GETDATE(),                                                          -- 31 INSERTDATE
    NULL,                                                               -- 32 UPDATEBY
    NULL,                                                               -- 33 UPDATEDATE
    NULL,                                                               -- 34 DELETEBY
    NULL,                                                               -- 35 DELETEDATE
    200,                                                                -- 36 DEFAULTRECORDCOUNT
    1,                                                                  -- 37 ISRECORDCOUNTENABLED
    NULL,                                                               -- 38 DISPLAYDYNAMICFIELDS
    'MDV_<<Entity>>',                                                   -- 39 SELECTVIEW
    '<<PKColumn>>',                                                     -- 40 DATAKEYFIELDS    e.g. 'CLAIMID'
    NULL,                                                               -- 41 CONDITION
    NULL,                                                               -- 42 SORTQUERY
    'FROM MDV_<<Entity>> <<ALIAS>>',                                    -- 43 FROMQUERY        e.g. 'FROM MDV_CLAIMS MDVC'
    NULL,                                                               -- 44 GROUPQUERY
    NULL,                                                               -- 45 QUICKSEARCHCONDITION
    '0',                                                                -- 46 LEVEL            '0' for first grid
    '1',                                                                -- 47 SHOWTOOLBAR
    NULL,                                                               -- 48 KEYWORDFIELDS
    NULL,                                                               -- 49 KEYWORDTITLE
    0,                                                                  -- 50 ISLAZYLOAD
    NULL                                                                -- 51 PARENTLAYOUTID   NULL for level 0
)
```

---

### SEARCH LAYOUT template (Phase 3) — copy this exactly, substitute placeholders

```sql
INSERT INTO TRK_LAYOUT
(
    [SUBCATEGORYID], [LAYOUTID], [LAYOUTTYPEID], [DESCRIPTION], [CAPTION], [DISPLAYORDER],
    [STYLE], [SKIN], [DEFAULTBUTTON], [COMMANDTYPE],
    [GRIDVISIBLEOPTIONS], [GRIDVISIBLEBUTTONS],
    [LINKDATAFIELDS], [LINKDATAFORMATSTRINGS],
    [LINKDATANAVURLFIELDS], [LINKDATANAVURLFORMATSTRINGS], [LINKENTITYNAMES],
    [TARGETFORMNAME], [TARGETMETHODNAME],
    [DISPLAYCHECKBOX], [SELECTIONMODE],
    [SELECTQUERY], [SELECTSP], [PREPROCESSQUERY], [PREPROCESSSP],
    [REPORTDBTABLES], [DELETEQUERY], [DELETESP],
    [ISACTIVE], [INSERTBY], [INSERTDATE], [UPDATEBY], [UPDATEDATE], [DELETEBY], [DELETEDATE],
    [DEFAULTRECORDCOUNT], [ISRECORDCOUNTENABLED], [DISPLAYDYNAMICFIELDS],
    [SELECTVIEW], [DATAKEYFIELDS],
    [CONDITION], [SORTQUERY], [FROMQUERY], [GROUPQUERY], [QUICKSEARCHCONDITION],
    [LEVEL], [SHOWTOOLBAR], [KEYWORDFIELDS], [KEYWORDTITLE],
    [ISLAZYLOAD], [PARENTLAYOUTID]
)
VALUES
(
    @SubCategoryID,                                                     -- 1  SUBCATEGORYID
    @LayoutIDSearch,                                                    -- 2  LAYOUTID        (resolve: Layout = 'SearchCriteriaLayout')
    @LayoutTypeIDDoubleCol,                                             -- 3  LAYOUTTYPEID    (resolve: LayoutType = 'DoubleColumn')
    'SearchCriteriaLayout for <<Entity>> SmartVUE',                     -- 4  DESCRIPTION
    '<<SmartVUE DisplayName>> Search',                                  -- 5  CAPTION
    1,                                                                  -- 6  DISPLAYORDER
    NULL,                                                               -- 7  STYLE
    NULL,                                                               -- 8  SKIN
    NULL,                                                               -- 9  DEFAULTBUTTON
    NULL,                                                               -- 10 COMMANDTYPE
    NULL,                                                               -- 11 GRIDVISIBLEOPTIONS
    'Export,Print',                                                     -- 12 GRIDVISIBLEBUTTONS
    NULL,                                                               -- 13 LINKDATAFIELDS
    NULL,                                                               -- 14 LINKDATAFORMATSTRINGS
    NULL,                                                               -- 15 LINKDATANAVURLFIELDS
    NULL,                                                               -- 16 LINKDATANAVURLFORMATSTRINGS
    NULL,                                                               -- 17 LINKENTITYNAMES
    NULL,                                                               -- 18 TARGETFORMNAME
    NULL,                                                               -- 19 TARGETMETHODNAME
    1,                                                                  -- 20 DISPLAYCHECKBOX
    215,                                                                -- 21 SELECTIONMODE
    NULL,                                                               -- 22 SELECTQUERY     NULL — SP handles query
    'USP_<<Entity>>_Search',                                            -- 23 SELECTSP
    NULL,                                                               -- 24 PREPROCESSQUERY
    NULL,                                                               -- 25 PREPROCESSSP
    NULL,                                                               -- 26 REPORTDBTABLES
    NULL,                                                               -- 27 DELETEQUERY
    NULL,                                                               -- 28 DELETESP
    1,                                                                  -- 29 ISACTIVE
    'VUE  Admin',                                                       -- 30 INSERTBY
    GETDATE(),                                                          -- 31 INSERTDATE
    NULL,                                                               -- 32 UPDATEBY
    NULL,                                                               -- 33 UPDATEDATE
    NULL,                                                               -- 34 DELETEBY
    NULL,                                                               -- 35 DELETEDATE
    200,                                                                -- 36 DEFAULTRECORDCOUNT
    1,                                                                  -- 37 ISRECORDCOUNTENABLED
    NULL,                                                               -- 38 DISPLAYDYNAMICFIELDS
    NULL,                                                               -- 39 SELECTVIEW      NULL — search uses SP
    NULL,                                                               -- 40 DATAKEYFIELDS
    NULL,                                                               -- 41 CONDITION
    NULL,                                                               -- 42 SORTQUERY
    NULL,                                                               -- 43 FROMQUERY       NULL — search uses SP
    NULL,                                                               -- 44 GROUPQUERY
    NULL,                                                               -- 45 QUICKSEARCHCONDITION
    NULL,                                                               -- 46 LEVEL
    '1',                                                                -- 47 SHOWTOOLBAR
    NULL,                                                               -- 48 KEYWORDFIELDS
    NULL,                                                               -- 49 KEYWORDTITLE
    0,                                                                  -- 50 ISLAZYLOAD
    NULL                                                                -- 51 PARENTLAYOUTID
)
```

> **WARNING:** Never generate a TRK_LAYOUT INSERT with only a subset of columns. All 51 values (numbered 1–51 in the templates above) must be present. Missing even one value causes SQL error Msg 109.

---

## SEC_COMPONENT & SEC_ACCESSLEVELRIGHT RULES (from SecurityPrivilges.agent.md — Block 1: SmartVUE Component)

For SmartVUE Phase 1, generate security following this EXACT pattern:

```sql
DECLARE @COMPONENTID INT
SELECT @COMPONENTID = MAX(ISNULL(COMPONENTID, 0))+1 FROM SEC_COMPONENT

IF NOT EXISTS (SELECT 1 FROM SEC_COMPONENT WHERE SANAME = '<<SmartVueName>>' AND SACODE = @SubCategoryID)
BEGIN
    INSERT INTO SEC_COMPONENT
        (COMPONENTID, SACODE, SUBCATEGORYID, SANAME, SADESC, SATYPE, PARENTCODE, ISACTIVE, APPLICATIONID, DISPLAYTEXT)
    VALUES
    (
        @COMPONENTID,
        @SubCategoryID,                   -- SACODE = integer SubCategoryID
        @SubCategoryID,                   -- SUBCATEGORYID = same value
        '<<SmartVueName>>',
        '<<SmartVue Description>>',
        'Tracking',                        -- SATYPE always 'Tracking' for SmartVUE
        '<<CategoryName>>',                -- PARENTCODE = category name string
        1,
        (SELECT OPTIONDETAILID FROM TRK_MTOPTIONDETAIL
         WHERE DESCRIPTION = 'VUE'
           AND OPTIONID = (SELECT OPTIONID FROM TRK_MTOPTION WHERE OPTIONTYPE = 'ApplicationName')),
        '<<SmartVue DisplayText>>'
    )
END

DECLARE @RIGHTID INT
SELECT @COMPONENTID = COMPONENTID FROM SEC_COMPONENT WHERE SACODE = CONVERT(VARCHAR, @SubCategoryID)

-- Export
SELECT @RIGHTID = RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'Export'
IF NOT EXISTS (SELECT 1 FROM SEC_ACCESSLEVELRIGHT WHERE COMPONENTID = @COMPONENTID AND RIGHTID = @RIGHTID)
BEGIN
    INSERT INTO SEC_ACCESSLEVELRIGHT (ACCESSLEVELRIGHTID, COMPONENTID, RIGHTID)
    VALUES ((SELECT MAX(ACCESSLEVELRIGHTID)+1 FROM SEC_ACCESSLEVELRIGHT), @COMPONENTID, @RIGHTID)
END

-- View
SELECT @RIGHTID = RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'View'
IF NOT EXISTS (SELECT 1 FROM SEC_ACCESSLEVELRIGHT WHERE COMPONENTID = @COMPONENTID AND RIGHTID = @RIGHTID)
BEGIN
    INSERT INTO SEC_ACCESSLEVELRIGHT (ACCESSLEVELRIGHTID, COMPONENTID, RIGHTID)
    VALUES ((SELECT MAX(ACCESSLEVELRIGHTID)+1 FROM SEC_ACCESSLEVELRIGHT), @COMPONENTID, @RIGHTID)
END
```

### Security Rules — NEVER violate these:
- `COMPONENTID` = `MAX(ISNULL(COMPONENTID,0))+1` — NEVER use SCOPE_IDENTITY()
- `ACCESSLEVELRIGHTID` = `MAX(ACCESSLEVELRIGHTID)+1` — always explicit, never SCOPE_IDENTITY()
- `APPLICATIONID` = subquery on TRK_MTOPTIONDETAIL WHERE DESCRIPTION='VUE' — never hardcode
- `SUBCATEGORYID` column MUST be included in SEC_COMPONENT INSERT
- `RIGHTID` resolved via `SEC_RIGHT WHERE SRNAME = 'View'` / `'Export'` — never hardcode RIGHTID
- Use `IF NOT EXISTS` guard on SACODE for SEC_COMPONENT
- Use `IF NOT EXISTS` guard on COMPONENTID+RIGHTID for SEC_ACCESSLEVELRIGHT
- SmartVUE component gets **View + Export** rights only (Block 1 from SecurityPrivilges.agent.md)
- Do NOT generate Block 2 (BLU_) or Block 3 (ACT_) in CreateSmartVue phase — those are separate

---

## GLOBAL CONSTRAINTS

- INSERTBY = 'VUE  Admin' (two spaces — always)
- INSERTDATE = GETDATE()
- Never reprint baseline ID tables in output
- Never regenerate earlier phases
- Keep each phase within safe Copilot response size

### @SubCategoryID Resolution Rule (Phases 2, 3, 5)
**NEVER use `SET @SubCategoryID = 0` with a manual placeholder.** Phases 2, 3, and 5 must resolve `@SubCategoryID` automatically via subquery at the top of each script:

```sql
DECLARE @SubCategoryID INT

SELECT @SubCategoryID = S.SUBCATEGORYID
FROM TRK_SUBCATEGORY S
INNER JOIN TRK_CATEGORY C ON C.CATEGORYID = S.CATEGORYID
WHERE UPPER(S.SUBCATEGORYNAME) = UPPER('<<SmartVueName>>')
  AND UPPER(C.CATEGORYNAME)    = UPPER('<<CategoryName>>')

IF @SubCategoryID IS NULL
BEGIN
    RAISERROR('ERROR: TRK_SUBCATEGORY ''<<SmartVueName>>'' under ''<<CategoryName>>'' not found. Ensure Phase 1 completed successfully.', 16, 1)
    RETURN
END

PRINT 'Resolved SubCategoryID = ' + CAST(@SubCategoryID AS VARCHAR)
```



### Script Generation Order (execute in this exact sequence)
1. `TRK_CATEGORY` — resolve existing CategoryID (almost never insert new)
2. `TRK_SUBCATEGORY` — drop/recreate via `USP_AC_Del_SubCategories_SmartVue` if exists, then INSERT with ALL columns per TRK_SUBCATEGORY COLUMN RULES (SUBCATEGORYNAME is NOT NULL and must be set)
3. `TRK_SUBCATEGORYAPPLICATION` — link to VUEPP application
4. `SEC_COMPONENT` + `SEC_ACCESSLEVELRIGHT` — follow SecurityPrivilges.agent.md Block 1 pattern exactly (see SEC_COMPONENT & SEC_ACCESSLEVELRIGHT RULES section below)
5. `TRK_LAYOUT` (Result first) — ALL 51 columns required (see TRK_LAYOUT COLUMN RULES section); SELECTVIEW='MDV_<<Entity>>', SELECTIONMODE=217, DATAKEYFIELDS='<<PK>>', LEVEL='0'
6. `TRK_TABLEFIELD` (Result columns) — one INSERT per result column + WIDTH property
7. `TRK_LAYOUT` (Search) — ALL 51 columns required (see TRK_LAYOUT COLUMN RULES section); SELECTSP='USP_<<Entity>>_Search', SELECTIONMODE=215
8. `TRK_TABLEFIELD` (Search fields) — ROWNO/COLUMNNO for 2-column layout (row,col)
9. `TRK_SUBMENU` — link to menu, RIBBONTARGET='Ribbon/Ribbon/<<Tab>>/<<Tab>>/<<Tab>>Search'
10. Create MDV view with only fields needed for search SP + key ID (e.g., PolicyID). Use ERD to determine joins needed to get text fields for search criteria. Use database_erd.agent.md for reference on table structures and relationships.

### All Hardcoded IDs (use SELECT subqueries with these descriptions — DO NOT guess)
| What | Lookup | Value |
|------|--------|-------|
| LAYOUTID search | Layout / SearchCriteriaLayout | **8** |
| LAYOUTID result | Layout / SearchResultLayout | **82** |
| LAYOUTTYPEID DoubleColumn (search) | LayoutType / DoubleColumn | **23** |
| LAYOUTTYPEID TripleColumn (search) | LayoutType / TripleColumn | use subquery |
| LAYOUTTYPEID NoColumn (result) | LayoutType / NoColumn | **20** |
| DISPLAYTYPEID CustomTextBox | TRK_DISPLAYTYPE | **119** |
| DISPLAYTYPEID CustomDropDown | TRK_DISPLAYTYPE | **127** |
| DISPLAYTYPEID CustomCalendarTextBox | TRK_DISPLAYTYPE | **120** |
| DISPLAYTYPEID CustomCurrencyTextBox | TRK_DISPLAYTYPE | **118** |
| DISPLAYTYPEID CustomLabel | TRK_DISPLAYTYPE | **124** |
| DATATYPEID String | DataType / String | **27** |
| DATATYPEID Decimal | DataType / Decimal | **26** |
| DATATYPEID Integer | DataType / Integer | **24** |
| DATATYPEID DateTime | DataType / DateTime | **28** |
| DATAMODEID Default | DataMode / Default | **33** |
| DATAMODEID Text | DataMode / Text | **34** |
| DATAMODEID Value | DataMode / Value | **35** |
| OperatorString Like% | OperatorString / Like% | **83** |
| OperatorString %Like% | OperatorString / %Like% | **84** |
| OperatorString Equal To | OperatorString / Equal To | **41** |
| OperatorDecimal Equal To | OperatorDecimal / Equal To | **68** |
| OperatorDecimal Between | OperatorDecimal / Between | **145** |
| OperatorInteger Equal To | OperatorInteger / Equal To | **51** |
| CategoryType Search | CategoryType / Search | use subquery |

### TRK_TABLEFIELD Key Rules
- **Search fields**: ROWNO=row, COLUMNNO=col (2-col layout), DISPLAYORDER=sequential, GRIDDISPLAYORDER=NULL, SHOWINGRID=1, SHOWINFILTER=1, OPERATORID=use operator above
- **Result fields**: ROWNO=1, COLUMNNO=1, DISPLAYORDER=sequential, GRIDDISPLAYORDER=sequential, SHOWINGRID=1, SHOWINFILTER=1, OPERATORID=NULL
- **Hidden key field** (e.g., ENTITYID): SHOWINGRID=0, SHOWINFILTER=0, SHOWINCUSTOMIZELAYOUT=0, GRIDDISPLAYORDER=999999
- Always add `TRK_TABLEFIELDPROPERTY` Width (px) for every result column
- TABLENAME = alias used in FROM query (e.g., 'MDVF' if FROM MDV_FUNDER "MDVF")
- INSERTBY = 'VUE  Admin' (two spaces)

### MDV_<<Entity>> View Pattern (in HO database)
```sql
CREATE VIEW dbo.MDV_<<ENTITY>> AS
SELECT T."COL1" AS "COL1", ..., MTO.ITEM AS "<<FIELD>>TEXT"
FROM dbo.<<ENTITY>> T WITH(NOLOCK)
LEFT JOIN dbo.MTOPTION MTO WITH(NOLOCK) ON MTO.OPTIONID = T.<<FIELD>>OPTIONID
-- Only expose columns needed for SmartVUE + key ID
```

### Output file name: `<<EntityName>>_SmartVue_Baseline.sql`

---

You need to create required baseline for new SmartVUE as per the instructions mentioned in the document below in a separate .sql file.

### Smart Vue creation document

Steps to create a new SmartVUE.

Understanding VUE baseline tables.

List of VUE baseline tables.

TRK_CATEGORY

TRK_SUBCATEGORY

TRK_SUBCATEGORYAPPLICATION

TRK_LAYOUT

TRK_TABLEFIELD

TRK_TABLEFIELDPROPERTY

List of security tables.

SEC_COMPONENT

SEC_ACCESSLEVELRIGHT

TRK_SUBMENU

Detail about each table.

TRK_CATEGORY

This table is used to store the category in which the SmartVUE will be created.

Ex: “Appointment Renewal Process” SmartVUE is created under “Agent” category.

Usually we will be adding new SmartVUE in existing Category’s only.

TRK_SUBCATEGORY

This table is used to store SmartVUE details.

Ex: “Appointment Renewal Process” (Subcategory) is name of the SmartVUE.

The description of other field’s in this table.

CATEGORYTYPE: default value should be ‘Search’ for SmartVUE.

CATEGORYID: This will be parent ID from TRK_CATEGORY table. The categoryid of category in which SmartVUE will be created.

DESCRIPTION: Description of SmartVUE

DISPLAYTEXT: SmartVUE display name is the SmartVUE panel.

SHOWINTRACKING: default should be 1.

DISPLAYORDER: The order in which SmartVUE is displayed in the category.

Ex: In “Agent” category if we add a new SmartVUE and the display order should be set as next continuation number or wherever we need to display the SmartVUE.

ISACTIVE: this is used to control the SmartVUE is active or not. If active then only it will be displayed in the SmartVUE panel.

ISINTERNAL: default value is 1.

TRK_SUBCATEGORYAPPLICATION

This table stores the type of SmartVUE, ignore this table.

TRK_LAYOUT

The Search & Result criteria of SmartVUE will be stored in this table.

There will be two entries into this table, one for search criteria and another for result criteria.

The description of other field’s in this table.

SUBCATEGORYID: the SmartVUE subcategoryid should be inserted here.

LAYOUTID: comes from baseline table TRK_MTOPTIONDETAIL and this will distinguish the search and result criteria.

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

LAYOUTID for search criteria:

SELECT  OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('Layout') AND UPPER(MTO.DESCRIPTION)  = UPPER('SearchCriteriaLayout')

LAYOUTID for result criteria:

SELECT  OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('Layout') AND UPPER(MTO.DESCRIPTION)  = UPPER('SearchResultLayout')

LAYOUTTYPEID: comes from baseline table TRK_MTOPTIONDETAIL and this will define the layout for search and result criteria.

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

LAYOUTTYPEID for search criteria:

SELECT OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID where UPPER(MT.OPTIONTYPE) = UPPER('LayoutType') and UPPER(MTO.DESCRIPTION)  = UPPER('DoubleColumn')

LAYOUTTYPEID for result criteria:

SELECT OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('LayoutType') AND UPPER(MTO.DESCRIPTION)  = UPPER('NoColumn')

DESCRIPTION: description of the search and result criteria ex: “SearchCriteriaLayout for Appointment Renewal Process” SmartVUE.

CAPTION: SmartVUE caption.

DISPLAYORDER: default is 1 for both search & result criteria.

GRIDVISIBLEBUTTONS:optional buttons which will be displayed to right hand top corner of SmartVUE. Ex: Export,Print

DISPLAYCHECKBOX: default is 1.

SELECTIONMODE: default is 217.

SELECTSP:

ISACTIVE

DEFAULTRECORDCOUNT

ISRECORDCOUNTENABLED

TRK_TABLEFIELD

The fields list for Search & Result Criteria will be define in this table, including the data type, control type etc.,

We need to decide on the Search & Result fields for the new SV to proceed to create baseline.

The description of field’s in this table.

SUBCATEGORYLAYOUTID: Subcategorylayoutid from TRK_LAYOUT table, it will be either search/result record.

TABLENAME: alias table name which comes from the select query from the SmartVUE search SP which will be described in later sections of this document.

FIELDNAME: fieldname of the search/result criteria.

ELEMENTNAME: elementname of the search/result criteria.

DISPLAYTEXT: display text of the field name, it could be either search /result criteria.

DISPLAYTYPEID: This ID comes from TRK_DISPLAYTYPE baseline table. If the search/result criteria has date control then we have to insert ID using below query.

SELECT DISPLAYTYPEID FROM TRK_DISPLAYTYPE WHERE UPPER(Name) = UPPER('CustomCalendarTextBox')

Only “Custom*” named Names from TRK_DISPLAYTYPE should be used here.

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

ISREQUIRED: value should be 1 if the field should be mandatory or else 0.

DATATYPEID: Based on the displaytypeid selected above the datatypeid should be inserted here. If DISPLAYTYPEID = date control then DATATYPEID will be “DateTime”.

Use below query for all the data types available in TRK_MTOPTIONDETAIL table.

SELECT * FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('DataType')

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

DATAMODEID: Based on the display type selected this value should be selected. If display type is like calender or drop down then data mode should be “Value” else if it is text field then “Text” should be selected.

Use below query to see all the data modes available.

Select * from TRK_MTOPTIONDETAIL MTO inner join TRK_MTOPTION MT on MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('DataMode')

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

ROWNO & COLUMNNO: the search criteria has layout of 2 columns, each column contains field name & control and multiple rows. So to appear the field in sequence we have to give Rowno=1, Columnno=1 for 1st field and then Rowno=1, Columnno=2 for 2nd  field and so on how many fields are there.

For result criteria these can be Rowno=1, Columnno=1 for all the fields.

DISPLAYORDER: this can be always 1 for search fields. But for result fields this should be set based on the preference for each column to be shown in which order in sequence.

OPERATORID: select appropriate operator for each search field, this operator will be sent to search procedure for the particular field when search is clicked.

Use below query to see all the data modes available.

SELECT * FROM TRK_MTOPTIONDETAIL MTO INNER JOIN TRK_MTOPTION MT ON MT.OPTIONID = MTO.OPTIONID WHERE UPPER(MT.OPTIONTYPE) = UPPER('OperatorDateTime')

Please see “Search & Result fields, Baseline tables and values” section to see all possible values.

ISLABELASSOCIATED:	 this should be 1 by default.

ISACTIVE: this should be 1 by default to show the field in SmartVUE. This field can be used to switch some fields from stop showing in SmartVUE if not needed.

SELECTQUERY: This is only for search criteria fields. Select query should be specified when display type is selected as “CustomDropdown”. This is required to display values in the drop down.

SHOWINFILTER: Default value is 1 if the field should be available in SmartVUE filter.

INSERTBY: name of the user who is performing this action.

INSERTDATE: SYSDATE

MERGE column should be generated with braces as SQL error getting error ex. [MERGE] 

TRK_TABLEFIELDPROPERTY

This table will store the properties for fields defined in TRK_TABLEFIELD, if not defined it will take default property.

Property could be any one of below list

DISABLED

Height

MaskInput

MaxLength

MaxValue

COMMANDJSFUNCTION

Format

PasswordChar

IgnoreFromWhere

WIDTH

align

IGNOREFROMWHERE

ReadOnly

width

InputMask

ValidationErrorMessage

Width

AllowDuplicateRecords

MULTISELECT

RegularExpression

TARGETURL

AssignMultipleRecords

Enabled

IsEncrypted

REPEATDIRECTION

The description of field’s in this table.

TABLEFIELDID: tablefieldid is field ID from TRK_TABLEFIELD for the field to which we need to set the property.

PROPERTYNAME: Name of the property to be assigned to the field. Sample script provided in “04_SV_ResultColumns_Width.sql” shows how to set WIDTH property for some fields.

PROPERTYVALUE: value to be set for the property defined here.

INSERTBY: name of the user who is performing this action.

INSERTDATE: SYSDATE

SEC_COMPONENT

The security related component for each SmartVUE will be stored in this table.

The description of field’s in this table.

SACODE: SubcategoryID of the SmartVUE.

SANAME: Display name of the SmartVUE.

SADESC: Description of the SmartVUE.

SATYPE: default should be 'Tracking' for SmartVUE.

PARENTCODE: parent code (category) of the new SmartVUE.

ISACTIVE: default to 1.

APPLICATIONID: default to 252.

DISPLAYTEXT: Display name of the SmartVUE.

SEC_ACCESSLEVELRIGHT

The access level rights for component created in SEC_COMPONENT will be defined in this table.

The description of field’s in this table.

COMPONENTID: the Componentid from the sec_component table which we created for the new SmartVUE.

RIGHTID: default to 5.

Scripts

Below are the list of scripts which can be used to create a new SmartVUE.

The instructions are provided in each script file.

“01_SV_Names_Baseline.sql”

This script will have insert statements which will insert data into below tables.

TRK_SUBCATEGORY

TRK_LAYOUT

TRK_SUBCATEGORYAPPLICATION

SEC_COMPONENT

SEC_ACCESSLEVELRIGHT

“02_SV_Search-Result_Baseline.sql”

This script will have insert statements which will insert data into TRK_TABLEFIELD.

Basically this script will insert the search & result fields into TRK_TABLEFIELD baseline table.

“03_SV_Search_Procedure.sql”

This script has sample SmartVUE search procedure which should be used to create a new procedure for new SmartVUE.

“04_SV_ResultColumns_Width.sql”

This script will insert properties for the fields we created for SmartVUE search and result.

The different kinds of properties allowed are mentioned in the same document when describing trk_tablefield.

Search & Result fields, Baseline tables and values.

| Field Name | Baseline Table | Description | Value |

| --- | --- | --- | --- |

| LAYOUTID (TRK_LAYOUT) for search criteria | TRK_MTOPTIONDETAIL |  | 8 |

| LAYOUTID (TRK_LAYOUT) for result criteria | TRK_MTOPTIONDETAIL |  | 82 |

|  |  |  |  |

| LAYOUTTYPEID (TRK_LAYOUT) for search criteria | TRK_MTOPTIONDETAIL |  | 23 |

| LAYOUTTYPEID (TRK_LAYOUT) for result criteria | TRK_MTOPTIONDETAIL |  | 20 |

|  |  |  |  |


DISPLAYTYPEID	NAME	DESCRIPTION	DISPLAYTEXT	NAMESPACE	ASSEMBLY
1	UltraButton	NULL	UltraButton	CSSI.Windows.Controls	CSSI.Windows.Controls
3	CustomMultiSelectDropDown	NULL	Multi Select Drop Down	NULL	NULL
6	UltraCalendarInfo	NULL	UltraCalendarInfo	CSSI.Windows.Controls	CSSI.Windows.Controls
7	UltraCalendarLook	NULL	UltraCalendarLook	CSSI.Windows.Controls	CSSI.Windows.Controls
10	UltraCheckedListBox	NULL	UltraCheckedListBox	CSSI.Windows.Controls	CSSI.Windows.Controls
13	UltraComboEditor	NULL	UltraComboEditor	CSSI.Windows.Controls	CSSI.Windows.Controls
14	UltraContextMenu	NULL	UltraContextMenu	CSSI.Windows.Controls	CSSI.Windows.Controls
15	UltraDataSource	NULL	UltraDataSource	CSSI.Windows.Controls	CSSI.Windows.Controls
16	UltraEditButtons	NULL	UltraEditButtons	CSSI.Windows.Controls	CSSI.Windows.Controls
17	UltraExpandableGroupBox	NULL	UltraExpandableGroupBox	CSSI.Windows.Controls	CSSI.Windows.Controls
18	UltraExplorerBar	NULL	UltraExplorerBar	CSSI.Windows.Controls	CSSI.Windows.Controls
19	UltraGrid	NULL	UltraGrid	CSSI.Windows.Controls	CSSI.Windows.Controls`
20	UltraGridEditButtons	NULL	UltraGridEditButtons	CSSI.Windows.Controls	CSSI.Windows.Controls
21	UltraGridOptions	NULL	UltraGridOptions	CSSI.Windows.Controls	CSSI.Windows.Controls
22	UltraGroupBox	NULL	UltraGroupBox	CSSI.Windows.Controls	CSSI.Windows.Controls
24	UltraLabelCharacterCounter	NULL	UltraLabelCharacterCounter	CSSI.Windows.Controls	CSSI.Windows.Controls
25	UltraLinkLabel	NULL	UltraLinkLabel	CSSI.Windows.Controls	CSSI.Windows.Controls
26	UltraListBox	NULL	UltraListBox	CSSI.Windows.Controls	CSSI.Windows.Controls
27	UltraMainMenu	NULL	UltraMainMenu	CSSI.Windows.Controls	CSSI.Windows.Controls
29	UltraMenuItem	NULL	UltraMenuItem	CSSI.Windows.Controls	CSSI.Windows.Controls
31	UltraOptionSet	NULL	UltraOptionSet	CSSI.Windows.Controls	CSSI.Windows.Controls
32	UltraPanel	NULL	UltraPanel	CSSI.Windows.Controls	CSSI.Windows.Controls
33	UltraPictureBox	NULL	UltraPictureBox	CSSI.Windows.Controls	CSSI.Windows.Controls
34	UltraProgressBar	NULL	UltraProgressBar	CSSI.Windows.Controls	CSSI.Windows.Controls
35	UltraRadioButton	NULL	UltraRadioButton	CSSI.Windows.Controls	CSSI.Windows.Controls
36	UltraRichTextBox	NULL	UltraRichTextBox	CSSI.Windows.Controls	CSSI.Windows.Controls
37	UltraSpellChecker	NULL	UltraSpellChecker	CSSI.Windows.Controls	CSSI.Windows.Controls
38	UltraSplitter	NULL	UltraSplitter	CSSI.Windows.Controls	CSSI.Windows.Controls
39	UltraTabControl	NULL	UltraTabControl	CSSI.Windows.Controls	CSSI.Windows.Controls
40	UltraTableLayoutPanel	NULL	UltraTableLayoutPanel	CSSI.Windows.Controls	CSSI.Windows.Controls
41	UltraTabPageControl	NULL	UltraTabPageControl	CSSI.Windows.Controls	CSSI.Windows.Controls
42	UltraTabSharedControlsPage	NULL	UltraTabSharedControlsPage	CSSI.Windows.Controls	CSSI.Windows.Controls
43	UltraTabStripControl	NULL	UltraTabStripControl	CSSI.Windows.Controls	CSSI.Windows.Controls
46	UltraToolbarsDockArea	NULL	UltraToolbarsDockArea	CSSI.Windows.Controls	CSSI.Windows.Controls
47	UltraToolbarsManager	NULL	UltraToolbarsManager	CSSI.Windows.Controls	CSSI.Windows.Controls
48	UltraTree	NULL	UltraTree	CSSI.Windows.Controls	CSSI.Windows.Controls
49	PopupMenuTool	NULL	PopupMenuTool	Infragistics.Win.UltraWinToolbars	Infragistics2.Win.UltraWinToolbars.v7.1
50	ButtonTool	NULL	ButtonTool	Infragistics.Win.UltraWinToolbars	Infragistics2.Win.UltraWinToolbars.v7.1
52	AccessDataSource	NULL	AccessDataSource	CSSI.Web.Controls	CSSI.Web.Controls
53	AdRotator	NULL	AdRotator	CSSI.Web.Controls	CSSI.Web.Controls
54	BulletedList	NULL	BulletedList	CSSI.Web.Controls	CSSI.Web.Controls
55	Button	NULL	Button	CSSI.Web.Controls	CSSI.Web.Controls
56	Calendar	NULL	Calendar	CSSI.Web.Controls	CSSI.Web.Controls
57	CheckBox	NULL	Check Box	NULL	NULL
58	CheckBoxList	NULL	Check Box List	CSSI.Web.Controls	CSSI.Web.Controls
59	ControlHelper	NULL	ControlHelper	CSSI.Web.Controls	CSSI.Web.Controls
60	DataList	NULL	DataList	CSSI.Web.Controls	CSSI.Web.Controls
61	DetailsView	NULL	DetailsView	CSSI.Web.Controls	CSSI.Web.Controls
62	DropDownList	NULL	Drop Down	NULL	NULL
63	FileUpLoad	NULL	FileUpLoad	CSSI.Web.Controls	CSSI.Web.Controls
64	FormView	NULL	FormView	CSSI.Web.Controls	CSSI.Web.Controls
65	GridView	NULL	GridView	CSSI.Web.Controls	CSSI.Web.Controls
66	Hidden	NULL	Hidden	NULL	NULL
67	HyperLink	NULL	Hyper Link	NULL	NULL
68	ImageButton	NULL	Image Button	NULL	NULL
69	ImageMap	NULL	ImageMap	CSSI.Web.Controls	CSSI.Web.Controls
70	Label	NULL	Microsoft Label	NULL	NULL
71	LinkButton	NULL	LinkButton	CSSI.Web.Controls	CSSI.Web.Controls
72	Literal	NULL	Literal	CSSI.Web.Controls	CSSI.Web.Controls
73	MultiView	NULL	MultiView	CSSI.Web.Controls	CSSI.Web.Controls
74	ObjectDataSource	NULL	ObjectDataSource	CSSI.Web.Controls	CSSI.Web.Controls
75	Panel	NULL	Panel	CSSI.Web.Controls	CSSI.Web.Controls
76	PlaceHolder	NULL	PlaceHolder	CSSI.Web.Controls	CSSI.Web.Controls
77	RadioButton	NULL	RadioButton	CSSI.Web.Controls	CSSI.Web.Controls
78	RadioButtonList	NULL	Radio Button List	CSSI.Web.Controls	CSSI.Web.Controls
79	Repeater	NULL	Repeater	CSSI.Web.Controls	CSSI.Web.Controls
80	SiteMapDataSource	NULL	SiteMapDataSource	CSSI.Web.Controls	CSSI.Web.Controls
81	SqlDataSource	NULL	SqlDataSource	CSSI.Web.Controls	CSSI.Web.Controls
82	Table	NULL	Table	CSSI.Web.Controls	CSSI.Web.Controls
83	TableCell	NULL	TableCell	CSSI.Web.Controls	CSSI.Web.Controls
84	TableFooterRow	NULL	TableFooterRow	CSSI.Web.Controls	CSSI.Web.Controls
85	TableHeaderCell	NULL	TableHeaderCell	CSSI.Web.Controls	CSSI.Web.Controls
86	TableHeaderRow	NULL	TableHeaderRow	CSSI.Web.Controls	CSSI.Web.Controls
87	TableRow	NULL	TableRow	CSSI.Web.Controls	CSSI.Web.Controls
88	TemplatedWizardStep	NULL	TemplatedWizardStep	CSSI.Web.Controls	CSSI.Web.Controls
89	TextBox	NULL	Microsoft Text Box	NULL	NULL
90	View	NULL	View	CSSI.Web.Controls	CSSI.Web.Controls
91	Wizard	NULL	Wizard	CSSI.Web.Controls	CSSI.Web.Controls
92	WizardStep	NULL	WizardStep	CSSI.Web.Controls	CSSI.Web.Controls
93	XmlDataSource	NULL	XmlDataSource	CSSI.Web.Controls	CSSI.Web.Controls
94	UltraWebGrid	NULL	UltraWebGrid	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
95	UltraWebGridToolBar	NULL	UltraWebGridToolBar	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
96	UltraWebTab	NULL	UltraWebTab	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
97	UltraWebTree	NULL	UltraWebTree	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
98	WebAsyncRefreshPanel	NULL	WebAsyncRefreshPanel	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
99	WebCalendar	NULL	WebCalendar	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
103	WebDateTimeEdit	NULL	WebDateTimeEdit	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
104	WebGroupBox	NULL	WebGroupBox	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
105	WebImageButton	NULL	WebImageButton	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
108	WebPanel	NULL	WebPanel	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
109	WebPercentEdit	NULL	WebPercentEdit	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
110	WebResizingExtender	NULL	WebResizingExtender	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
111	WebTextEdit	NULL	WebTextEdit	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
113	HtmlAnchor	HtmlAnchor	HtmlAnchor	System.Web.UI.HtmlControls	System.Web.UI.HtmlControls
114	FindAndUse	Find And Use Control	Find And Use	NULL	NULL
115	UltraAutoCompleteComboBox	UltraAutoCompleteComboBox	UltraAutoCompleteComboBox	CSSI.Windows.Controls	CSSI.Windows.Controls
116	UltraSearchBox	UltraSearchBox	UltraSearchBox	CSSI.Windows.Controls	CSSI.Windows.Controls
117	UltraAutoCompleteComboEditor	UltraAutoCompleteComboEditor	UltraAutoCompleteComboEditor	CSSI.Windows.Controls	CSSI.Windows.Controls
118	Autocomplete	Autocomplete	Autocomplete	CSSI.Windows.Controls	CSSI.Windows.Controls
119	DatePicker	DatePicker	Microsoft DatePicker	CSSI.Web.IG.Controls	CSSI.Web.IG.Controls
120	MultiUpload	MultiUpload	MultiUpload	CSSI.Windows.Controls	CSSI.Windows.Controls
121	MultiSelectDropDown	NULL	MultiSelectDropDown	CSSI.Web.Controls	CSSI.Web.Controls
122	CustomCurrencyTextBox	NULL	Currency Box	NULL	NULL
123	CustomTextBox	NULL	Text Box	NULL	NULL
124	CustomCalendarTextBox	NULL	Date Picker with Box	NULL	NULL
125	CustomMaskedTextBox	NULL	Masked Box	NULL	NULL
126	CustomNumericTextBox	NULL	Numeric Box	NULL	NULL
127	CustomCalendar	NULL	Date Picker	NULL	NULL
128	CustomLabel	NULL	Label	NULL	NULL
129	CustomCheckBox	NULL	Check Box	NULL	NULL
130	CustomDropDownTextBox	NULL	Combo Box With Text Box	NULL	NULL
131	CustomDropDown	NULL	Combo Box	NULL	NULL
132	CustomReadOnlyGridView	NULL	Grid	NULL	NULL
133	CustomGridView	NULL	Editable Grid	NULL	NULL
134	CascadeDropDown	NULL	Cascade Dropdown	NULL	NULL
135	FindAndUseDropDownList	NULL	Multi Select Find And Use	NULL	NULL
136	CustomIntegerTextBox	NULL	CustomIntegerTextBox	NULL	NULL
137	CustomPercentTextBox	NULL	CustomPercentTextBox	NULL	NULL
138	MultiSelectDropDownList	MultiSelectDropDownList	Multi Select Drop Down	NULL	NULL
139	TimePicker	TimePicker	TimePicker	NULL	NULL
140	MonthYearPicker	MonthYearPicker	MonthYearPicker	NULL	NULL
141	FindAndUseSmartSearch	FindAndUse with SmartSearch	FindAndUseSmartSearch	NULL	NULL
142	ToggleSlider	A slider toggle control to switch betwen 2 values.	ToggleSlider	NULL	NULL
146	Priority	Displays a radio button based priorty control in VUE 9	Priority	NULL	NULL
147	Image	NULL	Image	NULL	NULL
148	AutoCompleteMultiSelect	A multi select control which supports auto complete	AutoCompleteMultiSelect	NULL	NULL
149	ToggleTags	A list of tags which you can select. This can be used in place of a multiselect dropdown list	Toggle Tags	NULL	NULL
150	ButtonGroup	Collection of image buttons as group prepared by html elements in VUE 8.5	ButtonGroup	NULL	NULL
151	RangeSlider	Display Range Slider With Min And Max Values	RangeSlider	NULL	NULL
152	Switch	A slider control which switch two values and also with no value	Switch	NULL	NULL
153	Text	NULL	Text	NULL	NULL
154	AutoCompleteSingleMultiSelectLookup	A multi or single select lookup control which supports auto complete	AutoCompleteSingleMultiSelectLookup	NULL	NULL
155	Editor	The Editor allows users to create rich text content	Editor	NULL	NULL
156	Menu	The allows users to create a menu list where user can perform actons	MenuGroup	NULL	NULL
157	Custom	To render different control for each row of a column	Custom	NULL	NULL


| DATATYPEID (TRK_TABLEFIELD) | TRK_MTOPTIONDETAIL |  |  |

|  |  | BLOB | 294 |

|  |  | Currency | 30 |

|  |  | DateTime | 28 |

|  |  | Decimal | 26 |

|  |  | Double | 25 |

|  |  | Guid | 31 |

|  |  | Image | 269 |

|  |  | Integer | 24 |

|  |  | NoDataType | 32 |

|  |  | String | 27 |

|  |  |  |  |

| DATAMODEID | TRK_MTOPTIONDETAIL | Default | 33 |

|  |  | Text | 34 |

|  |  | Value | 35 |

|  |  |  |  |

| OPERATORID | TRK_MTOPTIONDETAIL |  |  |

| OperatorCurrency |  | Equal To | 75 |

|  |  | Greater Than Or Equal To | 80 |

|  |  | Is Not Null | 79 |

|  |  | Is Null | 78 |

|  |  | Less Than Or Equal To | 81 |

|  |  | Like | 76 |

|  |  | Not Equal To | 77 |

|  |  |  |  |

| OperatorCustomDateTime |  | Next X Days | 261 |

|  |  | Next X Months | 263 |

|  |  | Next X Weeks | 262 |

|  |  | Next X Years | 264 |

|  |  | Previous X Days | 257 |

|  |  | Previous X Months | 259 |

|  |  | Previous X Weeks | 258 |

|  |  | Previous X Years | 260 |

|  |  |  |  |

| OperatorDateTime |  | Between | 62 |

|  |  | Custom | 256 |

|  |  | Equal To | 58 |

|  |  | Greater Than | 61 |

|  |  | Greater Than Or Equal To | 85 |

|  |  | Is Not Null | 66 |

|  |  | Is Null | 65 |

|  |  | Less Than | 60 |

|  |  | Less Than Or Equal To | 86 |

|  |  | Not Between | 63 |

|  |  | Not Equal To | 64 |

|  |  |  |  |

| OperatorDecimal |  | Between | 145 |

|  |  | Containing | 144 |

|  |  | Equal To | 68 |

|  |  | Greater Than | 142 |

|  |  | Greater Than Or Equal To | 73 |

|  |  | Is Not Null | 72 |

|  |  | Is Null | 71 |

|  |  | Less Than | 143 |

|  |  | Less Than Or Equal To | 74 |

|  |  | Not Between | 146 |

|  |  | Not Containing | 297 |

|  |  | Not Equal To | 70 |

|  |  |  |  |

| OperatorInteger |  | Between | 138 |

|  |  | Containing | 137 |

|  |  | Equal To | 51 |

|  |  | Greater Than | 135 |

|  |  | Greater Than Or Equal To | 56 |

|  |  | Is Not Null | 55 |

|  |  | Is Null | 54 |

|  |  | Less Than | 136 |

|  |  | Less Than Or Equal To | 57 |

|  |  | Not Between | 141 |

|  |  | Not Containing | 296 |

|  |  | Not Equal To | 53 |

|  |  |  |  |

| OperatorString |  | %Like | 42 |

|  |  | %Like% | 84 |

|  |  | Containing | 50 |

|  |  | Equal To | 41 |

|  |  | Is Not Null | 49 |

|  |  | Is Null | 48 |

|  |  | Like% | 83 |

|  |  | Not Containing | 295 |

|  |  | Not Equal To | 47 |

|  |  |  |  |

|  |  |  |  |



"

### Hierarchy Smart View Guidelines

- For hierarchy Smart view, we can have mulitple result layouts( single TRK_SUBCATEGORY with Multiple TRK_LAYOUT records) which should differentiated with LEVEL, DESCRIPTION in TRK_LAYOUT table and the first result grid contains LEVEL =0 for send level it will be LEVEL =1 and so on. The search criteria will be same for all the levels and only result layout will be different based on the level. And for LEVEL-1 grid the PARENTLAYOUTID will be first layout SUBCATEGORYLAYOUTID and for LEVEL-2 grid the PARENTLAYOUTID will be LEVEL-1 layout SUBCATEGORYLAYOUTID and so on.  

And fields for each level will be defined in TRK_TABLEFIELD with respective SUBCATEGORYLAYOUTID. And based on the level we can show and hide the columns in gridview and also can set the drill down link in result grid to navigate to next level layout.

### Sample SmartVUE creation script

```sql 

 DECLARE  @ComponentID INT, @CategoryId INT, @SubCategoryId INT,@SubCategoryLayoutId INT,   
@TempSubCategoryId INT, @TableFieldId INT, @FilterId INT, @ToolbarId INT, @SubCategoryLayoutGroupId INT  
 IF NOT EXISTS ( SELECT 1 FROM TRK_CATEGORY WITH(NOLOCK) WHERE CATEGORYNAME ='BookofBusiness' ) 
 BEGIN 
INSERT INTO [TRK_CATEGORY]           
    ( [CATEGORYNAME], [DESCRIPTION], [DISPLAYTEXT], [ENTITYNAME], [ENTITYTYPE], [DISPLAYORDER],[ISACTIVE],  
    [INSERTBY], [INSERTDATE], [UPDATEBY], [UPDATEDATE], [DELETEBY], [DELETEDATE], [ISINTERNAL] )          
    SELECT 'BookofBusiness','Book of Business','Book of Business','Agent',(SELECT ENTITYTYPEID FROM ENTITYTYPE WITH(NOLOCK) WHERE NAME = 'Agent'),71,1,'VUE Admin',CONVERT(VARCHAR(50), GETDATE(), 109), NULL,NULL,NULL,NULL,0
 SET @CategoryId = SCOPE_IDENTITY()           
  END           
  ELSE           
  BEGIN 
 SELECT @CategoryId = CATEGORYID FROM TRK_CATEGORY WITH(NOLOCK) WHERE CATEGORYNAME='BookofBusiness'            
END 

IF EXISTS ( SELECT 1 FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYNAME = 'ListOfPolicies' AND CATEGORYID =  @CategoryId AND CATEGORYTYPE =(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search'))
 BEGIN 
DECLARE @pXMLString VARCHAR(MAX)
SELECT @TempSubCategoryId = SUBCATEGORYID FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYNAME ='ListOfPolicies'  
    AND CATEGORYID = @CategoryId AND CATEGORYTYPE =(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search')
SET @pXMLString = '<Root>        
      <SubCategory>        
       <SubcategoryID>'+CONVERT(VARCHAR,@TempSubCategoryId)+'</SubcategoryID>      
        <EntityState></EntityState>        
       <ChangedBy>VUE Admin</ChangedBy>    
      </SubCategory>      
     </Root>'
--<link>
 SELECT '||~||ListOfPolicies||~||' RETURN
--</link>
EXEC USP_AC_Del_SubCategories_SmartVue @pXMLString 
 END 
IF NOT EXISTS ( SELECT 1 FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYNAME='ListOfPolicies'   
    AND CATEGORYID =  @CategoryId AND CATEGORYTYPE =(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search'))        
     BEGIN 
INSERT INTO [TRK_SUBCATEGORY]           
    ( [CATEGORYID], [CATEGORYTYPE], [SUBCATEGORYNAME], [DESCRIPTION], [DISPLAYTEXT], [ENTITYNAME], [ENTITYTYPE], [PAGETITLE],  
    [SHOWINTRACKING], [DISPLAYORDER], [ISACTIVE], [INSERTBY], [INSERTDATE], [UPDATEBY], [UPDATEDATE], [DELETEBY], [DELETEDATE],  
    [ISPUBLISH], [DATASOURCEMODE] , [ISINTERNAL] ,[ISCRITERIAREQUIRED] )
SELECT @CategoryId,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search'),'ListOfPolicies','List of Policies','List of Policies',
'Agent',(SELECT ENTITYTYPEID FROM ENTITYTYPE WITH(NOLOCK) WHERE NAME = 'Agent'), NULL,1,775,1,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), NULL,NULL,NULL,NULL,1,'MDV',NULL,NULL 
 SET @SubCategoryId = SCOPE_IDENTITY()          
   
  END 
 ELSE 
 BEGIN 
  SELECT @SubCategoryId = SUBCATEGORYID FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYNAME='ListOfPolicies'                          
     AND CATEGORYID =  @CategoryId AND CATEGORYTYPE =(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search')
 UPDATE [TRK_SUBCATEGORY]   
 SET [CATEGORYID] = @CategoryID  
 ,[CATEGORYTYPE] = (SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'CategoryType' AND MTO.DESCRIPTION  = 'Search'),
[SUBCATEGORYNAME] = 'ListOfPolicies',
[DESCRIPTION] = 'List of Policies',
[DISPLAYTEXT] = 'List of Policies',
[ENTITYNAME] = 'Agent',
[ENTITYTYPE] = (SELECT ENTITYTYPEID FROM ENTITYTYPE WITH(NOLOCK) WHERE NAME = 'Agent'),
[PAGETITLE] = NULL,
[SHOWINTRACKING] = 1,
[DISPLAYORDER] = 775,
[ISACTIVE] = 1,
[INSERTBY] = 'VUE  Admin',
[INSERTDATE] = CONVERT(VARCHAR(50), GETDATE(), 109),
[UPDATEBY] = NULL,
[UPDATEDATE] = NULL,
[DELETEBY] = NULL,
[DELETEDATE] = NULL,
[ISPUBLISH] = 1,
[DATASOURCEMODE] = 'MDV',
[ISINTERNAL] = NULL,
[ISCRITERIAREQUIRED] = NULL
WHERE SUBCATEGORYID = @SubCategoryId AND CATEGORYID = @CategoryID END    

INSERT INTO [TRK_SUBCATEGORYAPPLICATION]          
    ( [SUBCATEGORYID], [APPLICATIONID], [INSERTBY], [INSERTDATE], [ISDATASECURITYENABLED], [DATASECURITYKEYFIELDS] )  
SELECT @SubCategoryId,(SELECT A.OPTIONDETAILID FROM TRK_MTOPTIONDETAIL A, TRK_MTOPTION B WHERE A.OPTIONID = B.OPTIONID AND B.OPTIONTYPE = 'ApplicationName'           
    AND A.DESCRIPTION = 'VUEPP'),'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 1,'AgentID'     

     
  INSERT INTO [AC_SUBCATEGORYVIEWRELATION]      
   ( [SUBCATEGORYID], [VIEWNAME], [SUBGROUPID] )   
SELECT @SubCategoryId,'MDV_ListOfPolicies_Aviva', (SELECT TOP 1 SUBGROUPID FROM AC_SUBGROUP WITH(NOLOCK) WHERE SUBGROUP = 'Agent Details')   
   

 IF NOT EXISTS ( SELECT 1 FROM SEC_COMPONENT WITH(NOLOCK) WHERE SACODE = CONVERT(varchar(max), @SubCategoryID))
BEGIN
      

INSERT INTO [SEC_COMPONENT]        
 ([COMPONENTID], [SACODE], [SUBCATEGORYID], [SANAME], [SADESC], [SATYPE], [PARENTCODE], [ISACTIVE], [APPLICATIONID], [DISPLAYTEXT] )  
 SELECT  (SELECT ISNULL(MAX(COMPONENTID)+1, 1) from SEC_COMPONENT WITH(NOLOCK)) , @SubCategoryID ,@SubCategoryID ,'ListOfPolicies','ListOfPolicies','Tracking','BookofBusiness','1',(SELECT A.OPTIONDETAILID FROM TRK_MTOPTIONDETAIL A, TRK_MTOPTION B WHERE A.OPTIONID = B.OPTIONID AND B.OPTIONTYPE = 'ApplicationName'         
    AND A.DESCRIPTION = 'VUEPP'),(SELECT DISPLAYTEXT FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYID = @SubCategoryId)   
 select @ComponentID = ISNULL(MAX(COMPONENTID), 1) from SEC_COMPONENT 
   

INSERT INTO [SEC_ACCESSLEVELRIGHT]        
( [ACCESSLEVELRIGHTID], [COMPONENTID], [RIGHTID] )  
 SELECT    
(SELECT ISNULL(MAX(ACCESSLEVELRIGHTID)+1, 1) FROM SEC_ACCESSLEVELRIGHT WITH(NOLOCK)),@ComponentID ,(SELECT TOP 1 RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'View')      

INSERT INTO [SEC_COMPONENT]        
 ([COMPONENTID], [SACODE], [SUBCATEGORYID], [SANAME], [SADESC], [SATYPE], [PARENTCODE], [ISACTIVE], [APPLICATIONID], [DISPLAYTEXT] )  
 SELECT  (SELECT ISNULL(MAX(COMPONENTID)+1, 1) from SEC_COMPONENT WITH(NOLOCK)) , @SubCategoryID ,@SubCategoryID ,'ListOfPolicies','ListOfPolicies','Tracking','AgentProfile','0',(SELECT A.OPTIONDETAILID FROM TRK_MTOPTIONDETAIL A, TRK_MTOPTION B WHERE A.OPTIONID = B.OPTIONID AND B.OPTIONTYPE = 'ApplicationName'         
    AND A.DESCRIPTION = 'VUEPP'),(SELECT DISPLAYTEXT FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYID = @SubCategoryId)   
 select @ComponentID = ISNULL(MAX(COMPONENTID), 1) from SEC_COMPONENT 
   

INSERT INTO [SEC_ACCESSLEVELRIGHT]        
( [ACCESSLEVELRIGHTID], [COMPONENTID], [RIGHTID] )  
 SELECT    
(SELECT ISNULL(MAX(ACCESSLEVELRIGHTID)+1, 1) FROM SEC_ACCESSLEVELRIGHT WITH(NOLOCK)),@ComponentID ,(SELECT TOP 1 RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'View')      

INSERT INTO [SEC_COMPONENT]        
 ([COMPONENTID], [SACODE], [SUBCATEGORYID], [SANAME], [SADESC], [SATYPE], [PARENTCODE], [ISACTIVE], [APPLICATIONID], [DISPLAYTEXT] )  
 SELECT  (SELECT ISNULL(MAX(COMPONENTID)+1, 1) from SEC_COMPONENT WITH(NOLOCK)) , @SubCategoryID ,@SubCategoryID ,'ListOfPolicies','ListOfPolicies','Tracking','Agent','1',(SELECT A.OPTIONDETAILID FROM TRK_MTOPTIONDETAIL A, TRK_MTOPTION B WHERE A.OPTIONID = B.OPTIONID AND B.OPTIONTYPE = 'ApplicationName'         
    AND A.DESCRIPTION = 'VUE'),(SELECT DISPLAYTEXT FROM TRK_SUBCATEGORY WITH(NOLOCK) WHERE SUBCATEGORYID = @SubCategoryId)   
 select @ComponentID = ISNULL(MAX(COMPONENTID), 1) from SEC_COMPONENT 
   

INSERT INTO [SEC_ACCESSLEVELRIGHT]        
( [ACCESSLEVELRIGHTID], [COMPONENTID], [RIGHTID] )  
 SELECT    
(SELECT ISNULL(MAX(ACCESSLEVELRIGHTID)+1, 1) FROM SEC_ACCESSLEVELRIGHT WITH(NOLOCK)),@ComponentID ,(SELECT TOP 1 RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'Export')      

INSERT INTO [SEC_ACCESSLEVELRIGHT]        
( [ACCESSLEVELRIGHTID], [COMPONENTID], [RIGHTID] )  
 SELECT    
(SELECT ISNULL(MAX(ACCESSLEVELRIGHTID)+1, 1) FROM SEC_ACCESSLEVELRIGHT WITH(NOLOCK)),@ComponentID ,(SELECT TOP 1 RIGHTID FROM SEC_RIGHT WHERE SRNAME = 'View')   END   
  
INSERT INTO [TRK_LAYOUT] ([SUBCATEGORYID],[LAYOUTID],[LAYOUTTYPEID],[DESCRIPTION],[CAPTION],[DISPLAYORDER],[STYLE],[SKIN],          
 [DEFAULTBUTTON],[COMMANDTYPE],[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS],[LINKDATAFIELDS],[LINKDATAFORMATSTRINGS],  
 [LINKDATANAVURLFIELDS],[LINKDATANAVURLFORMATSTRINGS],[LINKENTITYNAMES],[TARGETFORMNAME],[TARGETMETHODNAME],[DISPLAYCHECKBOX],  
 [SELECTIONMODE],[SELECTQUERY],[SELECTSP],[PREPROCESSQUERY],[PREPROCESSSP],[REPORTDBTABLES],[DELETEQUERY],[DELETESP],[ISACTIVE]  
 ,[INSERTBY],[INSERTDATE],[UPDATEBY],[UPDATEDATE],[DELETEBY],[DELETEDATE],[DEFAULTRECORDCOUNT],[ISRECORDCOUNTENABLED],   
 [DISPLAYDYNAMICFIELDS],[SELECTVIEW],[DATAKEYFIELDS],[CONDITION],[SORTQUERY],[FROMQUERY],[GROUPQUERY],[QUICKSEARCHCONDITION],  
 [LEVEL],[SHOWTOOLBAR],[KEYWORDFIELDS],[KEYWORDTITLE],[ISLAZYLOAD],[PARENTLAYOUTID]  
    )  
SELECT @SubCategoryId,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'Layout' AND MTO.DESCRIPTION  = 'SearchResultLayout'), (SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'LayoutType' AND MTO.DESCRIPTION  = 'NoColumn'), 'SearchResultLayout for ListOfPolicies', 'ListOfPolicies', 1, NULL, NULL, NULL, NULL, 'CardView,GroupBy,Filter,Toggle,ShowAll,Summary,GridConfig', 'Export,Print','POLICYOWNERNAME|POLICYNUMBER',NULL,'CUSTOMERID|POLICYID','javascript:OpenCustomerExtDetails({0});|javascript:OpenPolicyExtDetails({0});',NULL,NULL,NULL,NULL,215,'SELECT  "MDVLPA"."AGENTNAME" AS "AGENTNAME", "MDVLPA"."POLICYOWNERNAME" AS "POLICYOWNERNAME", "MDVLPA"."POLICYNUMBER" AS "POLICYNUMBER", "MDVLPA"."PolicyName" AS "PolicyName", "MDVLPA"."ApplicationReceivedDate" AS "ApplicationReceivedDate", "MDVLPA"."PolicyPremiumAmount" AS "PolicyPremiumAmount", "MDVLPA"."PremiumPaidToDate" AS "PremiumPaidToDate", "MDVLPA"."PaymentMethood" AS "PaymentMethood", "MDVLPA"."PaymentFrequency" AS "PaymentFrequency", "MDVLPA"."ISSUEDDATE" AS "ISSUEDDATE", "MDVLPA"."InceptionDate" AS "InceptionDate", "MDVLPA"."PolicyStatus" AS "PolicyStatus", "MDVLPA"."StatusEffectiveDate" AS "StatusEffectiveDate", "MDVLPA"."APE" AS "APE", "MDVLPA"."PolicyOwner" AS "PolicyOwner", "MDVLPA"."POLICYID" AS "POLICYID", "MDVLPA"."PAYMENTMETHODID" AS "PAYMENTMETHODID", "MDVLPA"."DISTRIBUTIONCODE" AS "DISTRIBUTIONCODE", "MDVLPA"."CUSTOMERID" AS "CUSTOMERID", "MDVLPA"."AGENTID" AS "AGENTID"',NULL,NULL,NULL,NULL,NULL,NULL,1,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 'VUE  Admin',NULL,NULL,NULL,50,1,NULL,'MDV_ListOfPolicies_Aviva',NULL,NULL,NULL,'FROM  "MDV_ListOfPolicies_Aviva" "MDVLPA"',NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL  
SET @SubCategoryLayoutId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','StatusEffectiveDate','StatusEffectiveDate','Status Effective Date',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,23,13,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PolicyOwner','PolicyOwner','Policy Owner',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,20,15,NULL,NULL,NULL,0,1,NULL,NULL,NULL,NULL,0,0,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ApplicationReceivedDate','ApplicationReceivedDate','Application Received Date',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,14,5,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PremiumPaidToDate','PremiumPaidToDate','Premium Paid To Date',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,13,7,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','InceptionDate','InceptionDate','Inception Date',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,12,11,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ISSUEDDATE','ISSUEDDATE','Issue Date',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,11,10,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PaymentMethood','PaymentMethood','Payment Methood',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,9,8,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','AGENTNAME','AGENTNAME','Servicing Agent Name',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,27,1,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,1,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()    
INSERT INTO [TRK_TABLEFIELDPROPERTY] ([TABLEFIELDID],[PROPERTYNAME],[PROPERTYVALUE],[INSERTBY],[INSERTDATE],[UPDATEBY],[UPDATEDATE],          
    [DELETEBY],[DELETEDATE],[VALUETYPE])    
SELECT @TableFieldId,'Width','150','Admin',CONVERT(VARCHAR(50), GETDATE(), 109), NULL,NULL,NULL,NULL,'px'   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','POLICYNUMBER','POLICYNUMBER','Policy Number',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,5,3,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','DISTRIBUTIONCODE','DISTRIBUTIONCODE','DISTRIBUTIONCODE',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,6,999999,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,0,1,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PolicyName','PolicyName','Policy Name',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,14,4,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PolicyStatus','PolicyStatus','Policy Status',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,20,12,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','POLICYOWNERNAME','POLICYOWNERNAME','Policy owner name and ID',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,25,2,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()    
INSERT INTO [TRK_TABLEFIELDPROPERTY] ([TABLEFIELDID],[PROPERTYNAME],[PROPERTYVALUE],[INSERTBY],[INSERTDATE],[UPDATEBY],[UPDATEDATE],          
    [DELETEBY],[DELETEDATE],[VALUETYPE])    
SELECT @TableFieldId,'Width','200','Admin',CONVERT(VARCHAR(50), GETDATE(), 109), NULL,NULL,NULL,NULL,'px'   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','Premiumstatus','Premiumstatus','Premium Status',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,3,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PolicyPremiumAmount','PolicyPremiumAmount','Policy Premium Amount',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Decimal'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,7,6,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','APE','APE','APE',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Decimal'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,22,14,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PAYMENTMETHODID','PAYMENTMETHODID','PAYMENTMETHODID',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,5,999999,NULL,NULL,NULL,0,1,NULL,NULL,NULL,NULL,0,0,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','AGENTID','AGENTID','AGENTID',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,23,999999,NULL,NULL,NULL,0,1,NULL,NULL,NULL,NULL,0,0,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','POLICYID','POLICYID','POLICYID',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,2,999999,NULL,NULL,NULL,0,1,NULL,NULL,NULL,NULL,0,1,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','PaymentFrequency','PaymentFrequency','Payment Frequency',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,9,9,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','CUSTOMERID','CUSTOMERID','CUSTOMERID',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,26,999999,NULL,NULL,NULL,1,1,NULL,NULL,NULL,NULL,0,0,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','DateOfIssuedMonth','DateOfIssuedMonth','Date of Issue Month',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,19,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ApplicationReceivedMonth','ApplicationReceivedMonth','Application Received Month',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,17,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','DateOfIssuedYear','DateOfIssuedYear','Date of Issue Year',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,18,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ApplicationReceivedYear','ApplicationReceivedYear','Application Received Year',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,16,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','StatusID','StatusID','Policy Status',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),NULL,NULL,2,999999,NULL,NULL,NULL,1,0,NULL,NULL,NULL,NULL,1,0,1,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,0,NULL,NULL,NULL,0,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
  
INSERT INTO [TRK_LAYOUT] ([SUBCATEGORYID],[LAYOUTID],[LAYOUTTYPEID],[DESCRIPTION],[CAPTION],[DISPLAYORDER],[STYLE],[SKIN],          
 [DEFAULTBUTTON],[COMMANDTYPE],[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS],[LINKDATAFIELDS],[LINKDATAFORMATSTRINGS],  
 [LINKDATANAVURLFIELDS],[LINKDATANAVURLFORMATSTRINGS],[LINKENTITYNAMES],[TARGETFORMNAME],[TARGETMETHODNAME],[DISPLAYCHECKBOX],  
 [SELECTIONMODE],[SELECTQUERY],[SELECTSP],[PREPROCESSQUERY],[PREPROCESSSP],[REPORTDBTABLES],[DELETEQUERY],[DELETESP],[ISACTIVE]  
 ,[INSERTBY],[INSERTDATE],[UPDATEBY],[UPDATEDATE],[DELETEBY],[DELETEDATE],[DEFAULTRECORDCOUNT],[ISRECORDCOUNTENABLED],   
 [DISPLAYDYNAMICFIELDS],[SELECTVIEW],[DATAKEYFIELDS],[CONDITION],[SORTQUERY],[FROMQUERY],[GROUPQUERY],[QUICKSEARCHCONDITION],  
 [LEVEL],[SHOWTOOLBAR],[KEYWORDFIELDS],[KEYWORDTITLE],[ISLAZYLOAD],[PARENTLAYOUTID]  
    )  
SELECT @SubCategoryId,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'Layout' AND MTO.DESCRIPTION  = 'SearchCriteriaLayout'), (SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'LayoutType' AND MTO.DESCRIPTION  = 'TripleColumn'), 'SearchCriteriaLayout for ListOfPolicies', 'ListOfPolicies', 1, NULL, NULL, NULL, NULL, NULL, NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,215,'SELECT  "MDVLPA"."AGENTNAME" AS "AGENTNAME", "MDVLPA"."POLICYOWNERNAME" AS "POLICYOWNERNAME", "MDVLPA"."POLICYNUMBER" AS "POLICYNUMBER", "MDVLPA"."PolicyName" AS "PolicyName", "MDVLPA"."ApplicationReceivedDate" AS "ApplicationReceivedDate", "MDVLPA"."PolicyPremiumAmount" AS "PolicyPremiumAmount", "MDVLPA"."PremiumPaidToDate" AS "PremiumPaidToDate", "MDVLPA"."PaymentMethood" AS "PaymentMethood", "MDVLPA"."PaymentFrequency" AS "PaymentFrequency", "MDVLPA"."ISSUEDDATE" AS "ISSUEDDATE", "MDVLPA"."InceptionDate" AS "InceptionDate", "MDVLPA"."PolicyStatus" AS "PolicyStatus", "MDVLPA"."StatusEffectiveDate" AS "StatusEffectiveDate", "MDVLPA"."APE" AS "APE", "MDVLPA"."PolicyOwner" AS "PolicyOwner", "MDVLPA"."POLICYID" AS "POLICYID", "MDVLPA"."PAYMENTMETHODID" AS "PAYMENTMETHODID", "MDVLPA"."DISTRIBUTIONCODE" AS "DISTRIBUTIONCODE", "MDVLPA"."CUSTOMERID" AS "CUSTOMERID", "MDVLPA"."AGENTID" AS "AGENTID"',NULL,NULL,NULL,NULL,NULL,NULL,1,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 'VUE  Admin',NULL,NULL,NULL,50,1,NULL,'MDV_ListOfPolicies_Aviva',NULL,NULL,NULL,'FROM  "MDV_ListOfPolicies_Aviva" "MDVLPA"',NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL  
SET @SubCategoryLayoutId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','AGENTNAME','AGENTNAME','Servicing Agent Name',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),3,1,0,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorString' AND MTO.DESCRIPTION  = '%Like%'),1,NULL,1,NULL,NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()    
INSERT INTO [TRK_TABLEFIELDPROPERTY] ([TABLEFIELDID],[PROPERTYNAME],[PROPERTYVALUE],[INSERTBY],[INSERTDATE],[UPDATEBY],[UPDATEDATE],          
    [DELETEBY],[DELETEDATE],[VALUETYPE])    
SELECT @TableFieldId,'Width','150','Admin',CONVERT(VARCHAR(50), GETDATE(), 109), NULL,NULL,NULL,NULL,'px'   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','Premiumstatus','Premiumstatus','Premium Status',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'CustomTextBox' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),2,1,4,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorString' AND MTO.DESCRIPTION  = '%Like%'),1,NULL,1,NULL,NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
'VUE  Admin',NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','DateOfIssuedMonth','DateOfIssuedMonth','Date of Issue Month',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),2,3,6,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorString' AND MTO.DESCRIPTION  = '%Like%'),1,NULL,1,'SELECT '''' AS [KEY],'''' AS [VALUE]  UNION SELECT 1 AS [KEY], ''January'' AS [VALUE]    UNION  SELECT 2 AS [KEY], ''February'' AS [VALUE]   UNION SELECT 3 AS [KEY], ''March'' AS [VALUE]    UNION  SELECT 4 AS [KEY], ''April'' AS [VALUE]   UNION SELECT 5 AS [KEY], ''May'' AS [VALUE]    UNION  SELECT 6 AS [KEY], ''June'' AS [VALUE]   UNION SELECT 7 AS [KEY], ''July'' AS [VALUE]    UNION  SELECT 8 AS [KEY], ''August'' AS [VALUE]   UNION SELECT 9 AS [KEY], ''September'' AS [VALUE]    UNION  SELECT 10 AS [KEY], ''October'' AS [VALUE]   UNION SELECT 11 AS [KEY], ''November'' AS [VALUE]    UNION  SELECT 12 AS [KEY], ''December'' AS [VALUE]',NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ApplicationReceivedMonth','ApplicationReceivedMonth','Application Received Month',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'String'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),1,3,3,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorString' AND MTO.DESCRIPTION  = '%Like%'),1,NULL,1,'SELECT '''' AS [KEY],'''' AS [VALUE]  UNION SELECT 1 AS [KEY], ''January'' AS [VALUE]    UNION  SELECT 2 AS [KEY], ''February'' AS [VALUE]   UNION SELECT 3 AS [KEY], ''March'' AS [VALUE]    UNION  SELECT 4 AS [KEY], ''April'' AS [VALUE]   UNION SELECT 5 AS [KEY], ''May'' AS [VALUE]    UNION  SELECT 6 AS [KEY], ''June'' AS [VALUE]   UNION SELECT 7 AS [KEY], ''July'' AS [VALUE]    UNION  SELECT 8 AS [KEY], ''August'' AS [VALUE]   UNION SELECT 9 AS [KEY], ''September'' AS [VALUE]    UNION  SELECT 10 AS [KEY], ''October'' AS [VALUE]   UNION SELECT 11 AS [KEY], ''November'' AS [VALUE]    UNION  SELECT 12 AS [KEY], ''December'' AS [VALUE]',NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','DateOfIssuedYear','DateOfIssuedYear','Date of Issue Year',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),2,2,5,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorInteger' AND MTO.DESCRIPTION  = 'Equal To'),1,NULL,1,'SELECT  DISTINCT
YEAR(ISSUEDDATE) AS [KEY], YEAR(ISSUEDDATE) AS [VALUE]         
    FROM  POLICY WHERE isnull(ISSUEDDATE,'''')<>'''' ORDER BY 1 DESC',NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','ApplicationReceivedYear','ApplicationReceivedYear','Application Received Year',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),1,2,2,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorInteger' AND MTO.DESCRIPTION  = 'Equal To'),1,NULL,1,'SELECT  DISTINCT
YEAR(APPDATE) AS [KEY], YEAR(APPDATE) AS [VALUE]         
    FROM  POLICY WHERE isnull(APPDATE,'''')<>'''' ORDER BY 1 DESC',NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()   
INSERT INTO [TRK_TABLEFIELD] ([SUBCATEGORYLAYOUTID],[LAYOUTGROUPID],[DESCRIPTION],[PARAMETERNAME]  
    ,[TABLENAME],[FIELDNAME], [ELEMENTNAME],[DISPLAYTEXT],[DISPLAYFORMAT],[DISPLAYTYPEID],[DATATYPEID],[DATAMODEID],[ROWNO]  
    ,[COLUMNNO],[DISPLAYORDER],[GRIDDISPLAYORDER],[BANDLEVEL], [OPERATORID],[ISLABELASSOCIATED],[ISSORTABLE],[ISACTIVE]  
    ,[SELECTQUERY],[STOREDPROCEDURENAME],[FORMULA],[DEFAULTVALUE] ,[SHOWINGRID],[SHOWINFILTER],          
     [SHOWINCUSTOMIZELAYOUT],[SHOWCALENDAR],[PARENTCONTROLID],[TARGETCONTROLID],[WRAPTEXT],[MERGE],[LISTID],[LISTNAME]  
     ,[ISREQUIRED],[VALIDATIONEXPRESSION], [FINDNUSESUBCATEGORYID],[FINDANDUSEDISPLAYFIELDS],[REPORTPARAMETERTYPE]  
     ,[GRIDVISIBLEOPTIONS],[GRIDVISIBLEBUTTONS] ,[INSERTBY],[INSERTDATE],[UPDATEBY], [UPDATEDATE],[DELETEBY],[DELETEDATE]  
     ,[LAYOUTTABID],[CUSTOMDISPLAYTEXT],[ISEDITABLE],[ISREMOVABLE], [SORTCOLORDER], [SORTDIRECTION], [ISGROUPCOL], [COLSUMMARYTYPE],          
  [FIELDEXPRESSION],[EXPRESSIONVALUE],[QUERYGROUPVALUE],[QUERYGROUPORDER],[QUERYSORTVALUE],[QUERYSORTORDER])   
SELECT @SubCategoryLayoutId,NULL,NULL,NULL,'MDVLPA','StatusID','StatusID','Policy Status',NULL,(SELECT TOP 1 DISPLAYTYPEID FROM TRK_DISPLAYTYPE WITH(NOLOCK) WHERE NAME = 'DropDownList' ),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataType' AND MTO.DESCRIPTION  = 'Integer'),(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'DataMode' AND MTO.DESCRIPTION  = 'Default'),1,1,1,NULL,NULL,(SELECT TOP 1 OPTIONDETAILID FROM TRK_MTOPTIONDETAIL MTO WITH(NOLOCK) INNER JOIN TRK_MTOPTION MT WITH(NOLOCK) ON MT.OPTIONID = MTO.OPTIONID     
 WHERE MT.OPTIONTYPE = 'OperatorInteger' AND MTO.DESCRIPTION  = 'Equal To'),1,NULL,1,'SELECT OPTIONID AS [KEY],ITEM AS [VALUE] FROM MTOPTION WHERE CATEGORY = ''POLICY STATUS'' AND PARENTID IS NULL AND ISACTIVE=1 AND ISNULL(DONOTSHOW,0)<>1 ORDER BY VALUE',NULL,NULL,NULL,NULL,1,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL, NULL,NULL,NULL,NULL,'VUE  Admin',CONVERT(VARCHAR(50), GETDATE(), 109), 
NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL   
SET @TableFieldId = SCOPE_IDENTITY()                       
              
    Select '"List of Policies" imported successfully under the Group "BookofBusiness".'         
')





EXEC( '


	IF NOT EXISTS(SELECT 1 FROM TRK_SUBMENU WHERE SACODE IN(SELECT CAST(SUBCATEGORYID AS VARCHAR(50)) FROM TRK_SUBCATEGORY SC INNER JOIN TRK_CATEGORY C ON SC.CATEGORYID = C.CATEGORYID WHERE CATEGORYNAME ='BookofBusiness' AND SUBCATEGORYNAME = 'ListOfPolicies' ))
	 BEGIN
	 DECLARE @SubmenuID int
DELETE SMLP FROM TRK_SUBMENU_LANDINGPAGE SMLP INNER JOIN TRK_SUBMENU SM ON SMLP.SUBMENUID=SM.ID
	WHERE TARGET = '/EntityGrid/Grid/BookofBusiness/ListOfPolicies' AND TEXT= 'List Of Policies' 
 DELETE FROM  TRK_SUBMENU  WHERE TARGET = '/EntityGrid/Grid/BookofBusiness/ListOfPolicies' AND TEXT= 'List Of Policies' 
INSERT INTO TRK_SUBMENU(
				MENUID
				, PARENTID
				, TEXT
				, TARGET
				, CSS
				, TOOLTIP
				, LEVEL
				, RIBBONGROUPID
				, SACODE
				, SORTORDER
				, ISACTIVE
				, DESCRIPTION
				, RIBBONTARGET
			)
SELECT (SELECT ID FROM TRK_MENU WHERE TEXT = 'Book of Business'),(SELECT ID FROM TRK_SUBMENU WHERE TEXT = 'Book of Business' AND ISACTIVE = 1 AND LEVEL = 0 AND MENUID IN (SELECT ID FROM TRK_MENU WHERE TEXT = 'Book of Business')),'List Of Policies','/EntityGrid/Grid/BookofBusiness/ListOfPolicies','pp-menuitem','ListOfPolicies',1,NULL,(SELECT SUBCATEGORYID FROM TRK_SUBCATEGORY SC INNER JOIN TRK_CATEGORY C ON SC.CATEGORYID = C.CATEGORYID WHERE CATEGORYNAME ='BookofBusiness' AND SUBCATEGORYNAME = 'ListOfPolicies' AND SC.ISACTIVE = 1),71,1,'ListOfPolicies','Ribbon/Ribbon/EmptyRibbon/EmptyRibbon/EmptyRibbonSearch'
SET @SubmenuID = SCOPE_IDENTITY()

	 END


  ```

#  Sample Script for TRK_SUBMENU
```sql

  IF NOT EXISTS (
			SELECT 1
			FROM TRK_SUBMENU
			WHERE ISNULL(TARGET, '') = ISNULL(('EntityGrid/Grid/Data Load Exceptions/ManageHolidayExceptions'), '')
			)
	BEGIN
		INSERT INTO TRK_SUBMENU (
			[MENUID]
			,[PARENTID]
			,[TEXT]
			,[TARGET]
			,[CSS]
			,[TOOLTIP]
			,[LEVEL]
			,[RIBBONGROUPID]
			,[SACODE]
			,[SORTORDER]
			,[ISACTIVE]
			,[DESCRIPTION]
			,[RIBBONTARGET]
			,[MENUIMG]
			,[COLUMNORDER]
			,[ISHIDDEN]
			)
		VALUES (
			(
				SELECT M.ID
				FROM DBO.TRK_MENU M
				WHERE TEXT = 'Tools'
				)
			,(
				SELECT ID
				FROM DBO.TRK_SUBMENU M
				WHERE ISNULL(PARENTID, '') = ''
					AND ISNULL(TARGET, '') = ''
					AND TEXT = 'Exceptions'
				)
			,'Manage Holiday'
			,'EntityGrid/Grid/Data Load Exceptions/ManageHolidayExceptions'
			,NULL
			,'Manage Holiday'
			,'1'
			,NULL
			,(
				(
					SELECT S.SUBCATEGORYID
					FROM DBO.TRK_SUBCATEGORY S
					INNER JOIN DBO.TRK_CATEGORY C ON C.CATEGORYID = S.CATEGORYID
					WHERE S.SUBCATEGORYNAME = 'ManageHolidayExceptions'
						AND C.CATEGORYNAME = 'Data Load Exceptions'
					)
				)
			,'194' -- Sequence number
			,'1'
			,NULL
			,'Ribbon/Ribbon/Dashboard/Dashboard/DashboardSearch'
			,NULL
			,NULL
			,NULL
			)
	END