---
name: plantuml
description: Use this skill when the user asks to create, edit, or generate a PlantUML or plantuml diagram (.puml file), or asks for a sequence diagram, class diagram, component diagram, or any other PlantUML or plantuml diagram type. Also use when the user wants to render or export a .puml file to PNG or ASCII.
version: 1.0.0
allowed-tools: Write, Edit, Read, Bash
---

# PlantUML Skill

PlantUML jar location: `~/progs/plantuml-1.2026.2.jar`

## Workflow

When creating or editing a `.puml` file, follow these steps in order:

### 1. Write or edit the `.puml` file
Create or update the file as requested by the user.

### 2. Validate by generating ASCII
Before confirming the diagram is ready, always run the ASCII render and check for errors:

```
java -jar ~/progs/plantuml-1.2026.2.jar <file>.puml -ttxt 2>&1; echo "Exit: $?"
```

- If **exit code is 0** and no `Error line` appears in output: diagram is valid. Proceed.
- If **exit code is non-zero** or output contains `Error line N`: fix the syntax error at the reported line and re-validate. Do NOT confirm the diagram is ready until this passes cleanly.

### 3. Generate requested outputs
Once validation passes, generate the requested formats:

- PNG:   `java -jar ~/progs/plantuml-1.2026.2.jar <file>.puml -tpng`
- ASCII: `java -jar ~/progs/plantuml-1.2026.2.jar <file>.puml -ttxt`
- SVG:   `java -jar ~/progs/plantuml-1.2026.2.jar <file>.puml -tsvg`

### 4. Confirm to the user
Report the generated files and their sizes. Show a brief excerpt of the ASCII output so the user can see the diagram structure at a glance.

## Notes

- Default theme: use `!theme blueprint` unless the user specifies otherwise.
- Output files are placed in the same directory as the `.puml` source file.
- ASCII output extension is `.atxt`.
- PlantUML exit code 200 means diagram has errors; exit code 0 means success (even if warnings are present).
