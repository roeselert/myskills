# SysModelAgent Skill

## Purpose
The **SysModelAgent** interprets, validates, and generates software from the SysModel DSL.  
It treats the DSL as the **single source of truth** for system context and architecture.

## Capabilities

### Parse
- Convert SysModel DSL text into a structured AST.
- Normalize component, entity, interface, and control definitions.

### Validate
- Ensure structural correctness of the model.
- Validation rules:
  - All components must have unique names.
  - All entities must define at least one field.
  - All integrated systems referenced in interfaces must exist in `context.integratedSystems`.
  - All controls must define a `purpose`.
  - All inbound/outbound interfaces must specify a payload or target.

### Analyze
- Produce architecture metadata:
  - ComponentGraph
  - InterfaceMap
  - EntityMap
  - IntegrationMap

### Generate
- Produce software artifacts based on the AST:
  - Component modules
  - Entity classes
  - Interface adapters
  - Integration clients
  - Configuration files
  - Architecture documentation

### Update
- Modify the SysModel DSL based on user instructions.
- Update constraints:
  - Maintain deterministic structure.
  - Preserve required sections.
  - Never introduce behavior outside controls.

### Lifecycle
- On every DSL change:
  - Re-parse
  - Re-validate
  - Re-generate artifacts

---
## SysModelAgent Definition

```markdown
skill "SysModelAgent" {

  description: "Parses, validates, analyzes, and generates software from SysModel DSL."

  inputs {
    model: SysModelDSL
  }

  capabilities {

    parse {
      action: "Convert DSL text into structured AST"
      output: SysModelAST
    }

    validate {
      action: "Check AST for structural correctness"
      rules: [
        "Unique component names",
        "Entities must define fields",
        "Integrated systems must exist",
        "Controls must define purpose",
        "Interfaces must define payload or target"
      ]
      output: ValidationReport
    }

    analyze {
      action: "Derive architecture metadata"
      outputs: [
        ComponentGraph,
        InterfaceMap,
        EntityMap,
        IntegrationMap
      ]
    }

    generate {
      action: "Generate software artifacts from AST"
      artifacts: [
        "Component modules",
        "Entity classes",
        "Interface adapters",
        "Integration clients",
        "Configuration files",
        "Architecture documentation"
      ]
      output: GeneratedCodeBundle
    }

    update {
      action: "Modify SysModel DSL based on user instructions"
      constraints: [
        "Preserve required sections",
        "Maintain deterministic structure",
        "No behavior outside controls"
      ]
      output: UpdatedSysModelDSL
    }
  }

  lifecycle {
    onModelChange: [
      "Re-parse",
      "Re-validate",
      "Re-generate artifacts"
    ]
  }
}
