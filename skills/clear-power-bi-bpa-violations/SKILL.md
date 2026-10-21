---
name: clear-power-bi-bpa-violations
description: 'Resolve Tabular Editor Best Practice Analyzer (BPA) violations in Power BI PBIP/TMDL semantic models using project-specific fix and ignore strategies. Use when clearing, fixing, triaging, or suppressing BPA findings or warnings, or when adding BestPracticeAnalyzer_IgnoreRules annotations.'
---

# Clear BPA violations

Resolve reported BPA findings without changing business intent. This is one authoritative
policy, written in SudoLang with plain-language actions and reasons.

## How to read and follow this skill

- `Constraints` are shared requirements; apply them to every finding.
- `fn` defines a reusable procedure. Follow its steps when called.
- `RULE_ID => { ... }` defines the strategy for that rule.
- In `match`, take the first matching branch; `default` handles all other cases.
- `Why:` explains a decision. It is not an additional action.
- `ask` means use the ask-user tool and wait for an answer before acting.
- `ruleId`, `object`, `table`, `column`, and `measure` refer to the current finding.

Exact BPA predicates are in [rule-predicates.md](./references/rule-predicates.md).
Read the matching section, when present, before classifying a finding or checking
whether a fix clears it. Not every rule has a stored predicate.

## Shared workflow and safeguards

The rule strategies below use these shared procedures. They are part of the same policy.

```sudolang
ClearBpaViolations {
  State {
    model                 // the explicitly identified local PBIP/TMDL model
    findings = []         // only BPA-reported: { ruleId, object, objectType }
    results  = []         // { object, ruleId, status, action, reason, evidence, validation }
  }

  Status = fixed | ignored | textHinted | alreadyValid | userDecided | userOnly | needsOwner | outOfScope
  // needsOwner = leave unchanged; report "needs model-owner decision".
  // userOnly = report the finding for the user to address; make no change.
  // outOfScope = rule is not covered here; make no change, including annotations.

  Constraints {
    // Scope and precedence
    Process only reported findings whose ruleId is listed in RuleStrategies.
      For any other ruleId, return outOfScope. Do not edit the object or add an annotation.
    A rule's strategy overrides the general rules below. When a rule does not say otherwise,
      prefer a safe fix over ignore(). Use ignore() only for intentional exceptions.
    If a case is ambiguous or unclear, return needsOwner.

    // Safety
    Before you edit an object, read its TMDL, dependencies (model and report), and existing annotations.
    Make minimal edits. Preserve business logic, unrelated properties, and unrelated formatting.
    Do not change DAX if blank behavior, filter context, or totals could change, unless the rule allows it.
    Add or change a relationship only with clear model evidence.
    Delete an object only after usage checks prove that it is unused and the user approves.
    Ask the user first before any of these: delete, refresh data, deploy, connect to Fabric,
      change Power Query/partition sources, or change security (roles, RLS, OLS).
    Report validation as passed only when the validation command completed successfully.

    // Tools
    Use powerbi-modeling-mcp or powerbi-authoring-local against State.model only.
      Batch metadata reads and writes. Export changes to this repository's TMDL.
    Prefer built-in search and model tools over scripts. A local script may only read in-scope
      model files and print to stdout: no network, no credentials, no file writes, no subprocesses.
    If the host asks for approval of an MCP call for the current finding (inspect, fix, annotate,
      export, validate) or of a read-only inspection script: choose "Allow in this Session".
      Never auto-approve delete, refresh, deploy, or unrelated operations.
      This does not bypass host controls and does not persist across sessions.
  }

  fn run() {
    1. Baseline: record the model, rule IDs and BPA rules version, and the finding list.
    2. Group findings by ruleId. For each group, batch the inspection, then apply its rule strategy below.
       For each ask, use one question per rule group and allow per-object overrides.
    3. Export the changed TMDL and validate it. Rerun the same BPA rules (same version) on the same model.
    4. report()
  }

  fn ignore(object, ruleIds) {
    Put the annotation on the flagged object. For partition rules, put it on the owning table.
    Use each ruleId exactly as written in this skill, including any / - ( ) characters.
    If the annotation exists, add the ruleIds to RuleIDs. Keep existing IDs and avoid duplicates.
    Otherwise, create it. TMDL form:
      annotation BestPracticeAnalyzer_IgnoreRules = {"RuleIDs":["RULE_A","RULE_B"]}
    return ignored
  }

  fn updateColumn(column, change) {
    Why: MCP column Update may replace the whole definition instead of patching it.
    Get the full column definition. Send every existing property unchanged, plus only `change`,
      unless patch semantics are confirmed. Never send a sparse definition.
    If the column belongs to a field parameter:
      keep the label column's groupByColumns reference to the hidden fields column;
      keep the fields column's ParameterMetadata (kind: 2).
      Then get both columns again and verify. Export, re-import, and verify again.
      Check the report visuals that reference them.
      If metadata is missing, restore it from a known-good definition and validate.
      Never claim that visuals are fixed on the strength of a description update alone.
  }

  fn triage(finding, options) {
    1. Show the finding and the evidence (dependencies, usage).
    2. ask "How should <ruleId> be handled?" with options.
    3. Apply the user's choice within the shared safeguards.
       If the choice is ignore, call ignore(object, [ruleId]) and return ignored.
       Otherwise return userDecided.
  }

  fn formatByMeaning(object, ruleId) {
    Baseline: record the live data types, visibility, formats, and PBI_FormatHint.
      Handle stale findings first and return alreadyValid for each.
      Do not infer live counts from TMDL when it has no inferred types.
    Classify by final return paths, dependencies, and consumers, not by name or Variant metadata.
      Text-or-BLANK() output is text, not mixed. One sample does not prove every branch.
    match (output) {
      text (label, URL, color, image URI, format-mask helper), measure only =>
        Keep the DAX. Merge {"isText":true} into PBI_FormatHint and keep the other keys.
        If the hint already has a conflicting flag, return needsOwner.
        return textHinted

      numeric =>
        Use the format of comparable approved objects (same units and precision).
        Use "#,0" only for whole-number display. Keep dynamic formats.

      date =>
        Use "Short Date" only when the time part does not matter. Otherwise use a comparable approved date format.

      blank-only | mixed | unclear precision =>
        Check the relevant selections, totals, no data, zero, and negative values.
        If it is still unclear, return needsOwner and leave the object unchanged.
    }
    Never put a numeric format on text or date output.
    DAX changes, splitting outputs, hiding, or deleting need explicit user approval.
    Any PBI_FormatHint clears PROVIDE_FORMAT_STRING_FOR_MEASURES only. It does not clear
      APPLY_FORMAT_STRING_MEASURES or column rules. Never add a hint just to suppress a finding.
    Validate the output types, blanks, and affected consumers. Do not force zero warnings.
  }

  fn report() {
    For each finding: object, ruleId, status, action, reason, evidence, validation result, and checks that could not run.
  }
}
```

## Rule strategies

Each block below belongs to `RuleStrategies`. Shared safeguards still apply.

### Relationships and keys

```sudolang
ENSURE_TABLES_HAVE_RELATIONSHIPS => {
  Why: Measure containers, selectors, calculation helpers, and parameter tables can be intentionally disconnected.
  match (table) {
    looks like fact | dimension | bridge | security | date | master data => needsOwner

    (name contains Measure, Metric, or KPI; or starts with SEL, CALC, or PAR)
      and content confirms that purpose
      => ignore(table, [ruleId])
      // Evidence: measures; NAMEOF or field-parameter metadata; disconnected selector/parameter usage;
      // or helper-only columns.

    default => needsOwner
  }
  Never create a relationship to clear this rule without clear model evidence.
}

INACTIVE_RELATIONSHIPS_THAT_ARE_NEVER_ACTIVATED => {
  match (relationship) {
    intentionally inactive => ignore(relationship, [ruleId]) and state the rationale in the report
    unused and not intended => propose deletion. Delete only after the user approves.
    default => needsOwner
  }
}

HIDE_FOREIGN_KEYS => {
  Why: Hide technical relationship keys from the report field list.
  updateColumn(column, isHidden: true)
}

MARK_PRIMARY_KEYS => {
  Verify that the column is the table's unique key.
  Then updateColumn(column, isKey: true).
}

RELATIONSHIP_COLUMNS_SHOULD_BE_OF_INTEGER_DATA_TYPE => {
  Why: Source keys may be non-integer by design; keeping them avoids extra surrogate-key columns.
  triage(finding, options: [ignore (default), convert to integer])
}
```

### DAX expressions

```sudolang
USE_THE_DIVIDE_FUNCTION_FOR_DIVISION => {
  Rewrite `a / b` as DIVIDE(a, b).
  If the original guarded against zero, use DIVIDE(a, b, alternate).
  Never ignore this rule.

  Why: DIVIDE returns BLANK where `/` returned infinity, NaN, or an error.
  If downstream logic depends on that result (ISERROR, IFERROR, infinity checks), return needsOwner.
}

AVOID_DUPLICATE_MEASURES => {
  triage(finding, options: [ignore (default), consolidate])
}

EXPRESSION_RELIANT_OBJECTS_MUST_HAVE_AN_EXPRESSION => userOnly
RELATIONSHIP_COLUMNS_SAME_DATA_TYPE => userOnly
```

### Formatting and data categories

```sudolang
NUMERIC_COLUMN_SUMMARIZE_BY => {
  Why: Defaults to Do Not Summarize to avoid unintended aggregations.
  updateColumn(column, summarizeBy: none)
}

PERCENTAGE_FORMATTING => {
  formatString = "#,0.0%;-#,0.0%;#,0.0%"
}

DATECOLUMN_FORMATSTRING => {
  formatString = "Short Date"  // Preferred default
}

APPLY_FORMAT_STRING_MEASURES,
PROVIDE_FORMAT_STRING_FOR_MEASURES,
APPLY_FORMAT_STRING_COLUMNS => {
  Why: Formatting must follow actual output type, units, and precision, not just a rule match.
  formatByMeaning(object, ruleId)
  // Column rules also match DateTime. Never give a date column "#,0".
}

INTEGER_FORMATTING => {
  Why: This predicate has no numeric-type restriction, so a match is only a candidate.
  Change only Int64, Double, and Decimal measures.
  Keep date/time and other nonnumeric formats, currency (any symbol, code, or locale),
    percentage, and dynamic formats.
  Keep intentional decimals. Use a passing format with the needed precision (for example "#,0.00").
  Use "#,0" only when whole-number display fits.
  If an intentional, valid format still triggers the rule:
    ignore(measure, [ruleId]) and record the reason.
  If the intended format is unclear, return needsOwner.
}

ADD_DATA_CATEGORY_FOR_COLUMNS => {
  match (column) {
    content or use clearly matches Country, Continent, City, Latitude, or Longitude
      => updateColumn(column, dataCategory: <matching category>)
    default => userOnly
  }
}

FORMAT_FLAG_COLUMNS_AS_YES/NO_VALUE_STRINGS => {
  triage(finding, options: [ignore, manual review])
}
```

### Naming, descriptions, and layout

```sudolang
OBJECTS_SHOULD_NOT_START_OR_END_WITH_A_SPACE,
TRIM_OBJECT_NAMES => {
  Why: Leading or trailing spaces can be deliberate for visuals.
  Always ask about each object and show the dependency analysis.
  If no dependency would break, suggest trimming the name.
  If the user keeps the name:
    ignore(object, [the reported IDs of these two rules]).
    Record the rationale in the object's description.
}

NO_CAMELCASE_MEASURES_TABLES => {
  Why: Power BI inherits PascalCase from the Data Lake.
    Renaming adds ETL steps and puts refresh resilience at risk.
  ignore(object, [ruleId])
}

PARTITION_NAME_SHOULD_MATCH_TABLE_NAME_FOR_SINGLE_PARTITION_TABLES => {
  Why: Renaming a partition offers little benefit here compared with its risk.
  ignore(table, [ruleId])
}

OBJECTS_WITH_NO_DESCRIPTION => {
  Write a concise description of the data source and business logic.
  Base it on model evidence (M source, DAX).
  Use updateColumn() for columns.
}

LAYOUT_ADD_TO_PERSPECTIVES => {
  triage(finding, options: [
    add to selected perspective(s),
    add to all perspectives,
    add to none and ignore the rule
  ])
}

LAYOUT_COLUMNS_HIERARCHIES_DF => {
  Propose display folders based on industry-standard column grouping.
  Keep the number of folders low. Consider visual and object usage and data types.
  ask the user, then apply the choice.
  Never ignore this rule.
}
```

### Model design, precision, and MDX

```sudolang
SNOWFLAKE_SCHEMA_ARCHITECTURE => {
  Why: A snowflake can be an intentional design choice; do not redesign it just to clear BPA.
  Find out why it is used and state the rationale in the report.
  ignore(table, [ruleId])
}

DATE/CALENDAR_TABLES_SHOULD_BE_MARKED_AS_A_DATE_TABLE => {
  Why: Marking supports time intelligence, but a Date or Calendar name alone is not proof.
  Review each table independently. Prefer one marked date table.
  match (table) {
    genuine date dimension
      and date column is unique, nonblank, and contiguous, with a consistent time part
      and (for classic time intelligence) covers full years
      and dependencies were checked (including report use of auto date hierarchies)
      => mark it as a date table

    confirmed intentional exception (helper matched by name, or a design that does not need marking)
      => ignore(table, [ruleId]) and record the rationale

    default => needsOwner
  }
}

AVOID_FLOATING_POINT_DATA_TYPES => {
  Why: These similar names have different precision and range:
    Double = "Decimal number": floating point, about 15 significant digits, possible rounding errors.
    Decimal = "Fixed decimal number"/Currency: 4 decimal places, range ±922,337,203,685,477.5807.
  Always ask the user, using these facts and the column's value range.
  match (answer) {
    keep Double (needs its range or more than 4 decimal places, and user accepts the tradeoffs)
      => ignore(column, [ruleId])

    convert (range checked, 4 decimal places and rounding accepted,
      explicit approval for any Power Query change)
      => convert the type, verify the column type, and rerun BPA.
         Do not add an ignore annotation.

    default => needsOwner
  }
}

SET_ISAVAILABLEINMDX_TO_TRUE_ON_NECESSARY_COLUMNS => {
  Why: Sort, hierarchy, and variation dependencies require MDX availability.
  updateColumn(column, isAvailableInMdx: true)
}

ISAVAILABLEINMDX_FALSE_NONATTRIBUTE_COLUMNS => {
  Why: Hidden status and no local dependencies do not rule out external MDX usage.
  match (column) {
    evidence or user confirms that no Analyze in Excel/MDX consumer needs it
      => updateColumn(column, isAvailableInMdx: false)
    confirmed MDX dependency => ignore(column, [ruleId])
    default => needsOwner
  }
}
```

### Unused objects and transformations

```sudolang
UNNECESSARY_COLUMNS => {
  Why: No local references alone does not prove that removal is safe.
  ignore(column, [ruleId])
  You may suggest removal; deletion still requires usage checks and user approval.
}

PERF_UNUSED_COLUMNS,
REMOVE_REDUNDANT_COLUMNS_IN_RELATED_TABLES => {
  triage(finding, options: [manual review, AI dependency check, ignore])
}

MINIMIZE_POWER_QUERY_TRANSFORMATIONS => {
  ignore(table, [ruleId])
  You may suggest improvements. Change Power Query only after explicit user approval.
}

REDUCE_USAGE_OF_CALCULATED_TABLES => {
  Why: Measure containers, field parameters, selectors, and helpers can be intentional.
  match (table) {
    measure container | field parameter | selector or helper => ignore(table, [ruleId])
    default => Suggest changes only and return needsOwner.
      Delete only after the user approves.
  }
}

REDUCE_USAGE_OF_CALCULATED_COLUMNS_THAT_USE_THE_RELATED_FUNCTION => {
  triage(finding, options: [manual review (Measure Killer), AI dependency check with suggestions, ignore])
  Also ask whether to create DAX query files in the model's DAXQueries folder
    that simulate the affected tables.
}
```

### Security-related decisions

```sudolang
REMOVE_ROLES_WITH_NO_MEMBERS => {
  ask the user to confirm that the roles use dynamic RLS.
  If confirmed => ignore(role, [ruleId]).
  Otherwise => userOnly.
}

MANY_RELATIONSHIPS_ON_TABLES_USED_FOR_DYNAMIC_ROW_LEVEL_SECURITY,
CHECK_IF_BI-DIRECTIONAL_AND_MANY-TO-MANY_RELATIONSHIPS_ARE_VALID,
CHECK_IF_DYNAMIC_ROW_LEVEL_SECURITY_(RLS)_IS_NECESSARY => {
  triage(finding, options: [ignore, manual review])
}
```
