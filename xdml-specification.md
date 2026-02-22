# XDML Specification

**xAssets Definition Markup Language**

Version 1.0

---

## Introduction

XDML is a plain-text data format designed to be readable by both humans and AI language models. It represents structured records as indented key-value pairs, with no brackets, no quoting rules, and no escape sequences in normal use.

A typical XDML document looks like this:

```
Location
    LocationName: London Office
    ParentLocationID: 4 : Head Office
END Location
```

That's a complete, parseable record. Three lines of content, immediately understandable to anyone who reads it.

### Why another format?

The world already has CSV, XML, JSON, YAML, and TOML. Each solves a real problem, but each introduces encoding complexity that makes the format harder for humans to read and harder for AI to produce reliably:

- **CSV** requires quoting rules for fields containing commas, quotes within quotes, and has no standard for nested data.
- **XML** is verbose, requires matched closing tags, and needs CDATA sections or entity encoding for special characters.
- **JSON** requires escaping of quotes, backslashes, and newlines. Multi-line strings are painful. A single mismatched bracket breaks the entire document.
- **YAML** looks clean but has a 90-page specification, implicit type coercion (the Norway problem: `NO` becomes `false`), and indent-sensitivity that causes subtle bugs.

XDML takes a different approach: keep the common case trivially simple, and provide a single mechanism (Block encoding) for the rare case where a value contains newlines.

### Designed for AI

XDML has specific properties that make it effective as an AI interchange format:

**Self-documenting lookup values.** Foreign key fields can include both the ID and a human-readable description:

```
    SubjectID: 1 : Assets
    LocationID: 47 : London Office
    MainMenuCategoryID: 855 : Actions
    StatusID: 1 : In Service
```

The `ID : Description` syntax is optional — the parser accepts any of these forms:

```
    LocationID: 47                    // ID only (compact)
    LocationID: 47 : London Office    // ID with description (self-documenting)
    LocationID: London Office         // Description only (reverse-looked up on save)
```

When an AI reads a XDML document, it understands what `47` means without needing a database lookup. When an AI writes XDML, it can use whichever form it has available. The description-only form means an AI can write `LocationID: London Office` and the system will resolve it to the correct ID automatically.

**No escaping.** Field values are literal text. There are no escape sequences for quotes, backslashes, angle brackets, or any other character. What you see is what gets stored.

**Line-oriented.** Each field is one line. An AI can produce XDML one line at a time without tracking bracket depth, indentation level, or quoting state. A malformed line fails that line — it doesn't corrupt the rest of the document.

**Tolerant indentation.** Indentation is for readability, not parsing. The parser doesn't care whether you use 2 spaces, 4 spaces, or tabs — only the structure markers (`END`, child block names) determine nesting.

---

## Document Structure

Every XDML document follows the same pattern:

```
ElementType
    FieldName: Value
    FieldName: Value
END ElementType
```

The first line declares the record type. Subsequent lines are field-value pairs. The document ends with `END ElementType`.

### Element types

XDML supports friendly "nice names" that map to database tables:

| Nice Name | Database Table | Child Table |
|-----------|---------------|-------------|
| Form | AssetView | AssetViewField |
| Query | SavedSelect | SavedSelectField |
| Script | DBTransform | — |
| Client Script | XCSScript | — |
| Menu | MainMenu | — |
| Menu Category | MainMenuCategory | — |
| Menu Group | MainMenuGroup | — |

Any database table name can also be used directly:

```
Location
    LocationName: Sydney Office
END Location

CostCentre
    CostCentreName: Engineering
    AccountingSystemCode: ENG-01
END CostCentre

Department
    DepartmentName: Research & Development
END Department
```

---

## Field Values

### Simple values

Most field values are plain text on a single line:

```
    AssetViewName: Edit Asset
    FieldWidth: 300
    Enabled: 1
    DatePurchased: 2026-01-15
```

No quoting is needed, regardless of what the value contains. Colons, spaces, special characters — all are part of the value:

```
    Tooltip: Show a list of Assets with the same Custodian as the selected Assets
    Formatting: color:#2167c0 mediumicon:fa-person
    CommandOverride: target=dialog&mode=edit
```

### Lookup values

Foreign key fields support three notations:

```
    SubjectID: 1                      // ID only
    SubjectID: 1 : Assets             // ID : Description (self-documenting)
    SubjectID: Assets                 // Description only (auto-resolved)
```

The `ID : Description` form is the canonical serialization format. On output, the system writes the ID and looks up the description so the document is self-explanatory. On input, all three forms are accepted — the description after the colon is informational and ignored if an ID is present.

The description-only form triggers a reverse lookup: the parser searches the referenced table's description field for a matching record and substitutes the ID. This fails with an error if the description is ambiguous or not found.

### Empty values

Omit the field entirely, or include it with an empty value:

```
    SearchFor2:
```

Both are equivalent — the field is not set.

### Boolean values

Use `True`, `False`, `0`, or `1` as appropriate to the underlying database field:

```
    Enabled: 1
    SourceUnicode: False
    SourceFirstRowHeaders: 0
```

### Dates

ISO 8601 format:

```
    DatePurchased: 2026-01-15
    LastRun: 2024-11-05T18:57:36Z
```

---

## Child Records

Some element types have child records. These are declared inline using `Child Type` / `END Child Type` markers:

```
Form
    AssetViewName: Asset Summary
    SubjectID: 1 : Assets

    Form Field
        FieldName: AssetDesc
        DisplayName: Description
        FieldWidth: 300
        FieldPosition: 1
        HTMLSource: span
    END Form Field

    Form Field
        FieldName: Location.LocationName
        DisplayName: Location
        FieldWidth: 200
        FieldPosition: 2
        HTMLSource: span
    END Form Field

    Form Field
        FieldName: DatePurchased
        DisplayName: Date Purchased
        FieldWidth: 120
        FieldPosition: 3
        HTMLSource: date
    END Form Field

END Form
```

The child type name is always `{ElementType} Field` — so `Form` has `Form Field`, `Query` has `Query Field`.

When saved, the parent record is created first, then each child record is created with the parent's primary key injected automatically.

---

## Multi-Line Values: Block Encoding

Most values are single-line, but some fields contain multi-line text — for example, a script's source code. XDML handles this with **Block encoding**:

```
    SourceQuery: Block 7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c
Location
    LocationName: Auto-Created Office
END Location

Set count = Sql "SELECT COUNT(*) FROM Asset"
    End Block 7f3a9b2c-1d4e-5f6a-8b9c-0d1e2f3a4b5c
```

The pattern is:

```
    FieldName: Block <delimiter>
<content — any number of lines, stored verbatim>
    End Block <delimiter>
```

The delimiter can be any string — GUIDs are conventional because they guarantee uniqueness, but shorter strings work too:

```
    SourceQuery: Block myblock1
// script code here
    End Block myblock1
```

**Why this works:** The delimiter is chosen at write time. Since no content will accidentally contain the exact `End Block <delimiter>` sequence on a line by itself, there is no escaping problem. The content is stored exactly as written, including newlines, indentation, and any special characters.

**When serializing**, any field value containing newlines is automatically wrapped in Block/End Block with a generated GUID. When parsing, content between matching delimiters is stored verbatim.

This is the only encoding mechanism in XDML. There are no escape sequences, no quoting rules, and no special characters. The common case (single-line values) needs no encoding at all. The uncommon case (multi-line values) uses a self-delimiting block that cannot collide with content.

---

## Creating, Updating, and Deleting

### Creating a new record

Omit the primary key field. The system generates one automatically:

```
Location
    LocationName: Tokyo Office
END Location
```

### Updating an existing record

Include the primary key field. Only the fields you specify are changed — all other fields retain their current database values:

```
Location
    LocationID: 47
    LocationName: London Headquarters
END Location
```

### Deleting a record

Add the reserved field `Delete: True`:

```
Form
    AssetViewID: 1234
    AssetViewName: Old Form
    Form Field
        FieldName: ObsoleteField
        Delete: True
    END Form Field
END Form
```

This deletes the matching child record while leaving the parent and other children untouched.

---

## Advanced Value Types

Some fields use structured value blocks for complex data.

### QueryVariant blocks

Filter conditions for queries are stored as structured XML, represented in XDML as nested Xml/XmlRecord blocks within a QueryVariant:

```
Query
    SavedSelectName: Active London Assets
    SubjectID: 1 : Assets
    WhereClause:     QueryVariant
    Xml T
        XmlRecord T
            ConditionName: A
            F: Location.LocationName
            Comparator: =
            SearchFor: London Office
            SearchFor2:
            Scheme: A AND B
        End XmlRecord
        XmlRecord T
            ConditionName: B
            F: Status.InService
            Comparator: <>
            SearchFor: 0
            SearchFor2:
            Scheme:
        End XmlRecord
    End Xml
    End QueryVariant

    Query Field
        FieldName: AssetDesc
        DisplayName: Description
        FieldWidth: 240
        FieldPosition: 1
        HTMLSource: span
    END Query Field

END Query
```

The Xml/XmlRecord structure maps to XML elements. `Xml T` creates a root element named `T`. Each `XmlRecord T` creates a child element named `T`. Fields within the XmlRecord become child elements of that record.

### QueryString blocks

Key-value parameter sets, used for field configuration:

```
    Form Field
        FieldName: Notes
        HTMLSource: textarea
        QueryString:         QueryString
            rows: 5
            maxlength: 2000
        End QueryString
    END Form Field
```

---

## Complete Examples

### A query with filter conditions and display fields

```
Query
    SavedSelectName: Contract Renewal Report
    Type: SYSTEM
    SecurityCode: PUBLIC
    SubjectID: 1 : Assets
    OrderBy: DatePurchased
    WhereClause:     QueryVariant
    Xml T
        XmlRecord T
            ConditionName: A
            F: Category.CategoryName
            Comparator: LIKE
            SearchFor: %License%
            SearchFor2:
            Scheme: A
        End XmlRecord
    End Xml
    End QueryVariant

    Query Field
        FieldName: AssetDesc
        DisplayName: Description
        FieldWidth: 300
        FieldPosition: 1
        HTMLSource: assetlink
    END Query Field

    Query Field
        FieldName: Category.CategoryName
        DisplayName: Category
        FieldWidth: 200
        FieldPosition: 2
        HTMLSource: span
    END Query Field

    Query Field
        FieldName: DatePurchased
        DisplayName: Date Purchased
        FieldWidth: 120
        FieldPosition: 3
        HTMLSource: date
    END Query Field

    Query Field
        FieldName: Custodian.CustodianName
        DisplayName: Custodian
        FieldWidth: 200
        FieldPosition: 4
        HTMLSource: span
    END Query Field

END Query
```

### A form with various field types

```
Form
    AssetViewName: Quick Asset Editor
    TableFlags: SYSTEM
    SubjectID: 1 : Assets
    ViewWidth: 800
    ViewHeight: 500

    Form Field
        FieldName: AssetDesc
        DisplayName: Description
        FieldWidth: 400
        FieldPosition: 1
        HTMLSource: input
    END Form Field

    Form Field
        FieldName: Location.LocationName
        DisplayName: Location
        FieldWidth: 250
        FieldPosition: 2
        HTMLSource: dotlookup
    END Form Field

    Form Field
        FieldName: DatePurchased
        DisplayName: Date Purchased
        FieldWidth: 120
        FieldPosition: 3
        HTMLSource: date
    END Form Field

    Form Field
        FieldName: Notes
        DisplayName: Notes
        FieldWidth: 99999
        FieldPosition: 4
        HTMLSource: textarea
        QueryString:         QueryString
            rows: 4
        End QueryString
    END Form Field

END Form
```

### A menu item

```
Menu
    MenuSet: Asset Management
    Name: View All Assets by Location
    MainMenuCategoryID: 855 : Actions
    CommandOverride:     QueryString
        savedselectid: 11439
        queryvariant: Base Query
        target: popover
        selectboxes: true
    End QueryString
    SortCode: 100
    Tooltip: Display all assets grouped by their physical location
    DisplayContext: AssetRecords
    Enabled: 1
    Formatting: color:#2167c0 mediumicon:fa-location-dot
END Menu
```

### A script with embedded code

```
Script
    DBTransformDesc: Deploy London Office Configuration
    DBTransformClass: IMPEXP
    SourceType: XAMSCRIPT
    TargetType: NONE
    DBTransformType: 1
    SourceQuery: Block 9a8b7c6d-5e4f-3a2b-1c0d-ef9876543210

// Create the location hierarchy
Location
    LocationName: United Kingdom
END Location

Set ukID = Sql "SELECT LocationID FROM Location WHERE LocationName = 'United Kingdom'"

Location
    LocationName: London Office
    ParentLocationID: %ukID%
END Location

// Create supporting reference data
Department
    DepartmentName: London Operations
END Department

CostCentre
    CostCentreName: LON-OPS
    AccountingSystemCode: LON-001
END CostCentre

    End Block 9a8b7c6d-5e4f-3a2b-1c0d-ef9876543210
END Script
```

---

## Grammar Summary

```
document        = element-type NEWLINE
                  field-list
                  "END " element-type NEWLINE

element-type    = nice-name | table-name

field-list      = { field-line | child-block | blank-line | comment }

field-line      = INDENT field-name ":" [ " " field-value ] NEWLINE

field-value     = simple-value
                | lookup-value
                | block-value
                | queryvariant-value
                | querystring-value
                | xml-value

simple-value    = <any text to end of line>

lookup-value    = INTEGER " : " <description text>
                | <description text>

block-value     = "Block " delimiter NEWLINE
                  <any lines>
                  INDENT "End Block " delimiter NEWLINE

child-block     = INDENT child-type NEWLINE
                  field-list
                  INDENT "END " child-type NEWLINE

child-type      = element-type " Field"

comment         = "//" <any text> NEWLINE

blank-line      = NEWLINE
```

INDENT is optional whitespace (spaces or tabs). The parser is case-insensitive for keywords (`END`, `End`, `end` all work) and element type names.

---

## Design Principles

1. **The common case should be trivial.** Most records are simple key-value pairs. No brackets, no quoting, no escaping.

2. **Self-documenting by default.** Lookup values carry their human-readable description alongside the ID. A reader never needs to cross-reference another table to understand what a value means.

3. **One encoding mechanism.** Block/End Block with a unique delimiter handles all multi-line content. There is no second escaping system to learn.

4. **Partial updates are natural.** Include only the fields you want to change. Omitted fields are untouched. This makes XDML documents concise — you don't need to echo back an entire record to change one field.

5. **AI-native.** The format can be produced by language models line-by-line. There are no bracket-matching requirements, no escaping rules to remember, and no implicit type coercion to get wrong. An AI that can write `FieldName: Value` can write XDML.
