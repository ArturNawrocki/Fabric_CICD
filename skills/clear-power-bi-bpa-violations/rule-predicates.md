# BPA rule predicates

Tabular Editor BPA expressions, keyed by rule ID. Use them to confirm that a finding is real or stale and that a fix will clear it. Rules not listed here have no predicate in this skill.

## SET_ISAVAILABLEINMDX_TO_TRUE_ON_NECESSARY_COLUMNS (columns)
```
IsAvailableInMDX = false
and (UsedInSortBy.Any() or UsedInHierarchies.Any() or UsedInVariations.Any() or SortByColumn != null)
```

## PERCENTAGE_FORMATTING
```
FormatString.Contains("%") and (FormatString <> "#,0.0%;-#,0.0%;#,0.0%" and FormatString <> "0%;-0%;0%")
```

## DATECOLUMN_FORMATSTRING
```
Name.IndexOf("Date", "OrdinalIgnoreCase") >= 0
and DataType = "DateTime"
and not (FormatString = "Short Date" or FormatString = "yyyy-MM-dd" or FormatString = "yyyy/MM/dd"
         or FormatString = "dd/MM/yyyy" or FormatString = "dd-MM-yyyy" or FormatString = "dd.MM.yyyy")
```

## APPLY_FORMAT_STRING_MEASURES (measures)
```
IsVisible
and string.IsNullOrWhiteSpace(FormatString)
and (DataType = "DateTime" or DataType = "Double" or DataType = "Decimal")
and string.IsNullOrWhiteSpace(FormatStringExpression)
```

## PROVIDE_FORMAT_STRING_FOR_MEASURES (measures)
```
not IsHidden
and not Table.IsHidden
and DataType <> DataType.String
and string.IsNullOrWhiteSpace(FormatString)
and string.IsNullOrWhiteSpace(FormatStringExpression)
and not HasAnnotation("PBI_FormatHint")
```

## APPLY_FORMAT_STRING_COLUMNS (columns)
```
IsVisible
and string.IsNullOrWhitespace(FormatString)
and (DataType = "Int64" or DataType = "DateTime" or DataType = "Double" or DataType = "Decimal")
```

## RELATIONSHIP_COLUMNS_SHOULD_BE_OF_INTEGER_DATA_TYPE (columns)
```
UsedInRelationships.Any() and DataType != DataType.Int64
```

## INTEGER_FORMATTING (measures)
```
(not FormatString.Contains("$")
 and not FormatString.Contains("%")
 and not (FormatString = "#,0" or FormatString = "#,0.0" or FormatString = "0.00" or FormatString = "#,0.00"
          or FormatString = "0" or FormatString = "#,##0.00" or FormatString = "0.0" or FormatString = "#0.00"
          or FormatString = "#0.0" or FormatString = "G"))
and string.IsNullOrEmpty(FormatStringExpression)
```

## UNNECESSARY_COLUMNS (columns)
```
(IsHidden or Table.IsHidden)
and ReferencedBy.Count = 0
and (not UsedInRelationships.Any())
and (not UsedInSortBy.Any())
and (not UsedInHierarchies.Any())
and (not Table.RowLevelSecurity.Any(it <> null and it.IndexOf("[" + current.Name + "]", "OrdinalIgnoreCase") >= 0))
and (not Model.Roles.Any(RowLevelSecurity.Any(it <> null and (
      it.IndexOf(current.Table.Name + "[" + current.Name + "]", "OrdinalIgnoreCase") >= 0
   or it.IndexOf("'" + current.Table.Name + "'[" + current.Name + "]", "OrdinalIgnoreCase") >= 0))))
and not (ObjectLevelSecurity.Any(it.ToString() == "None"))
and not (Table.ObjectLevelSecurity.Any(it.ToString() == "None"))
and not Table.HasAnnotation("MEASURESTABLE") and not HasAnnotation("ALLOW_UNUSED")
```

## OBJECTS_WITH_NO_DESCRIPTION
```
string.IsNullOrWhitespace(Description) and IsHidden == false
```

## NO_CAMELCASE_MEASURES_TABLES (measures, tables)
```
IsVisible
and RegEx.IsMatch(Name, "[A-Z]([A-Z0-9]*[a-z][a-z0-9]*[A-Z]|[a-z0-9]*[A-Z][A-Z0-9]*[a-z])[A-Za-z0-9]*")
and not Name.Contains(" ")
and (Model.Cultures.Count = 0 or TranslatedNames.Any(it = "" or it = outerIt.Name))
and (not HasAnnotation("ALLOW_CAMEL") and not HasAnnotation("MEASURESTABLE"))
```

## PARTITION_NAME_SHOULD_MATCH_TABLE_NAME_FOR_SINGLE_PARTITION_TABLES
```
Table.Partitions.Count = 1 and Table.Partitions[0].Name <> Table.Name
```

## SNOWFLAKE_SCHEMA_ARCHITECTURE (tables)
```
UsedInRelationships.Any(current.Name == FromTable.Name)
and UsedInRelationships.Any(current.Name == ToTable.Name)
```

## DATE/CALENDAR_TABLES_SHOULD_BE_MARKED_AS_A_DATE_TABLE (tables)
```
(Name.ToUpper().Contains("DATE") or Name.ToUpper().Contains("CALENDAR"))
and (DataCategory <> "Time" or Columns.Any(IsKey == true && DataType == "DateTime") == false)
```

## AVOID_FLOATING_POINT_DATA_TYPES (columns)
```
DataType = "Double" and not HasAnnotation("ALLOW_FLOAT")
```

## MINIMIZE_POWER_QUERY_TRANSFORMATIONS (partitions)
```
SourceType.ToString() = "M"
and (Query.Contains("Table.Combine(") or Query.Contains("Table.Join(") or Query.Contains("Table.NestedJoin(")
  or Query.Contains("Table.AddColumn(") or Query.Contains("Table.Group(") or Query.Contains("Table.Sort(")
  or Query.Contains("Table.Pivot(") or Query.Contains("Table.Unpivot(") or Query.Contains("Table.UnpivotOtherColumns(")
  or Query.Contains("Table.Distinct(") or Query.Contains("[Query=""SELECT") or Query.Contains("Value.NativeQuery")
  or Query.Contains("OleDb.Query") or Query.Contains("Odbc.Query"))
```

## REDUCE_USAGE_OF_CALCULATED_TABLES (calculated tables)
```
not Expression.Contains("NAMEOF(") and not Name.Contains("Measure")
```

## ISAVAILABLEINMDX_FALSE_NONATTRIBUTE_COLUMNS (columns)
```
IsAvailableInMDX
and (IsHidden or Table.IsHidden)
and not UsedInSortBy.Any()
and not UsedInHierarchies.Any()
and not UsedInVariations.Any()
and SortByColumn = null
```

## PERF_UNUSED_COLUMNS (columns)
```
not IsVisible
and ReferencedBy.Count = 0
and (not UsedInRelationships.Any())
and (not UsedInSortBy.Any())
and (not UsedInHierarchies.Any())
and (not UsedInVariations.Any())
and not Table.HasAnnotation("MEASURESTABLE") and not HasAnnotation("ALLOW_UNUSED")
```

## REMOVE_REDUNDANT_COLUMNS_IN_RELATED_TABLES (columns)
```
UsedInRelationships.Any() == false
and Model.AllColumns.Any(Name == current.Name and Table.Name != current.Table.Name
    and Table.UsedInRelationships.Any(FromTable.Name == current.Table.Name))
```

## AVOID_DUPLICATE_MEASURES (measures)
```
Model.AllMeasures.Any(
    Expression.Replace(" ","").Replace("\n","").Replace("\r","").Replace("\t","")
      = outerIt.Expression.Replace(" ","").Replace("\n","").Replace("\r","").Replace("\t","")
    and (FormatStringExpression = null ? "" : FormatStringExpression).Replace(" ","").Replace("\n","").Replace("\r","").Replace("\t","")
      = (outerIt.FormatStringExpression = null ? "" : outerIt.FormatStringExpression).Replace(" ","").Replace("\n","").Replace("\r","").Replace("\t","")
    and it <> outerIt
)
```

## ADD_DATA_CATEGORY_FOR_COLUMNS (columns)
```
string.IsNullOrWhiteSpace(DataCategory)
&& !Name.ToLower().Contains("capacity")
&& (
    ((Name.ToLower().Contains("country") || Name.ToLower().Contains("continent") || Name.ToLower().Contains("city"))
      && DataType == DataType.String)
    || ((Name.ToLower() == "latitude" || Name.ToLower() == "longitude")
      && (DataType == DataType.Decimal || DataType == DataType.Double))
)
```
