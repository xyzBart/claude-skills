---
name: libreoffice-basic
description: Use this skill when the user asks to write, fix, or generate a LibreOffice Basic macro, especially for Calc (spreadsheets). Covers cell selection, range iteration, string manipulation, and UNO API quirks.
version: 1.0.0
allowed-tools: Write, Edit, Read
---

# LibreOffice Basic Macro Skill

Target version: **LibreOffice 25.8.x** (confirmed working on 25.8.5)

## Selection Handling — The Correct Pattern

This is the most error-prone area. Follow this pattern exactly.

### Step 1: Get selection and active sheet

```vba
Dim oDoc As Object
Dim oSel As Object
Dim oSheet As Object

oDoc = ThisComponent
oSel = oDoc.getCurrentSelection()
If IsNull(oSel) Or IsEmpty(oSel) Then Exit Sub

oSheet = oDoc.getCurrentController().getActiveSheet()
```

### Step 2: Get range addresses as an array

Use `getRangeAddresses()` (plural) for multi-range selections, `getRangeAddress()` (singular) for single range. Check with `supportsService`:

```vba
Dim aAddrs As Variant

If oSel.supportsService("com.sun.star.sheet.SheetCellRanges") Then
    aAddrs = oSel.getRangeAddresses()
Else
    ReDim aAddrs(0)
    aAddrs(0) = oSel.getRangeAddress()
End If
```

### Step 3: Iterate using absolute indices via the sheet

```vba
Dim oCell As Object
Dim nRow As Long, nCol As Long
Dim k As Long

For k = 0 To UBound(aAddrs)
    For nRow = aAddrs(k).StartRow To aAddrs(k).EndRow
        For nCol = aAddrs(k).StartColumn To aAddrs(k).EndColumn
            oCell = oSheet.getCellByPosition(nCol, nRow)
            ' work with oCell here
        Next nCol
    Next nRow
Next k
```

## Cell Access

- `oCell.getString()` — get cell value as string (works for any cell type)
- `oCell.setString(s)` — set string value
- `oCell.getValue()` — get numeric value
- `oCell.setValue(n)` — set numeric value
- Check if cell has content: `oCell.getString() <> ""`

## Known API Pitfalls (25.8.x)

| What fails | Why | What to use instead |
|---|---|---|
| `oCell.getType()` | "property not found" at runtime | Check `getString() <> ""` or just attempt operation |
| `oRange.Rows.Count` | "argument is not optional" | `aAddrs(k).EndRow - aAddrs(k).StartRow + 1` |
| `oRange.getRangeAddress()` inside a called Sub | Fails when object passed as parameter | Always resolve addresses in the main sub, pass primitives or iterate inline |
| Enumeration (createEnumeration) for cell iteration | LO 25.8 ranges enumerate rows, not cells — double nesting needed, brittle | Use index-based `getCellByPosition` instead |
| Calling Sub with `SubName(obj)` then using UNO methods on obj | Object may not behave correctly when passed | Inline the logic or assign to local var with `Dim x As Object : x = obj` |

## Full Working Template

```vba
Sub MyMacro()
    Dim oDoc As Object
    Dim oSel As Object
    Dim oSheet As Object
    Dim oCell As Object
    Dim aAddrs As Variant
    Dim nRow As Long, nCol As Long
    Dim k As Long

    oDoc = ThisComponent
    oSel = oDoc.getCurrentSelection()
    If IsNull(oSel) Or IsEmpty(oSel) Then Exit Sub

    oSheet = oDoc.getCurrentController().getActiveSheet()

    If oSel.supportsService("com.sun.star.sheet.SheetCellRanges") Then
        aAddrs = oSel.getRangeAddresses()
    Else
        ReDim aAddrs(0)
        aAddrs(0) = oSel.getRangeAddress()
    End If

    For k = 0 To UBound(aAddrs)
        For nRow = aAddrs(k).StartRow To aAddrs(k).EndRow
            For nCol = aAddrs(k).StartColumn To aAddrs(k).EndColumn
                oCell = oSheet.getCellByPosition(nCol, nRow)
                ' --- your logic here ---
            Next nCol
        Next nRow
    Next k
End Sub
```
