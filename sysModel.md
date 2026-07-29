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
context {
  system "OrderManagement" purpose: "Order lifecycle"

  stakeholders {
    "Customer" role: "Initiates orders"
    "SupportAgent" role: "Manual handling"
    "FinanceDept" role: "Billing"
  }

  integratedSystems {
    "CRM" type: external
    "PaymentGateway" type: external
    "WarehouseSystem" type: external
  }

  businessComponents {
    "Order" meaning: "Purchase request"
    "Invoice" meaning: "Billing document"
  }
}

architecture {
  component "OrderProcessing" type: business {

    interfaces {
      inbound "CreateOrder" payload: OrderRequest
      outbound "PaymentRequest" target: PaymentGateway
    }

    entities {
      "Order" {
        id: UUID
        items: List<Item>
        totalAmount: Money
        status: OrderStatus
      }
    }

    controls {
      "OrderValidator" purpose: "Check completeness"
      "PaymentOrchestrator" purpose: "Coordinate payment"
    }
  }

  component "InventorySync" type: business {

    interfaces {
      outbound "StockQuery" target: WarehouseSystem
    }

    entities {
      "Item" {
        id: UUID
        stock: Int
      }
    }

    controls {
      "StockUpdater" purpose: "Update item stock"
    }
  }
}

