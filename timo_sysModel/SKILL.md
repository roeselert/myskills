
---
name: SysModelAgent Skill
description: This skill can be used to create an intemediate representation of a software system based on arc42 and bce

---
## Composition
- compose with `bce` on how to design boundary-control-entity


## Purpose
The **SysModelAgent** interprets, validates, and generates software from the SysModel DSL.  
It treats the DSL as the **single source of truth** for system context and architecture.

## Syntax
- Use mermaid.js as dsl kl
- c4 diagram 
- Entity Relationship
- Class diagram
- sequence diagram

### Validate
- Ensure structural correctness of the model.
- Validation rules:
  - All components must have unique names.
  - All entities must define at least one field.
  - All integrated systems referenced in interfaces must exist in `context.integratedSystems`.

 IntegrationMap


### Update
- Modify the SysModel DSL based on user instructions.
- Update constraints:
  - Maintain deterministic structure.
  - Preserve required sections.
  - Never introduce behavior outside controls.
- Every change in the source code must be reflected in the model. For controls it is important to capture the main flow and not every detail. 


---
## SysModelAgent Definition

```
C4Context
    title System Context diagram for Internet Banking System
    Enterprise_Boundary(b0, "BankBoundary0") {
        Person(customerA, "Banking Customer A", "A customer of the bank, with personal bank accounts.")
        Person(customerB, "Banking Customer B")
        Person_Ext(customerC, "Banking Customer C", "desc")

        Person(customerD, "Banking Customer D", "A customer of the bank, <br/> with personal bank accounts.")

        System(SystemAA, "Internet Banking System", "Allows customers to view information about their bank accounts, and make payments.")

        Enterprise_Boundary(b1, "BankBoundary") {
            SystemDb_Ext(SystemE, "Mainframe Banking System", "Stores all of the core banking information about customers, accounts, transactions, etc.")

            System_Boundary(b2, "BankBoundary2") {
                System(SystemA, "Banking System A")
                System(SystemB, "Banking System B", "A system of the bank, with personal bank accounts. next line.")
            }

            System_Ext(SystemC, "E-mail system", "The internal Microsoft Exchange e-mail system.")
            SystemDb(SystemD, "Banking System D Database", "A system of the bank, with personal bank accounts.")

            Boundary(b3, "BankBoundary3", "boundary") {
                SystemQueue(SystemF, "Banking System F Queue", "A system of the bank.")
                SystemQueue_Ext(SystemG, "Banking System G Queue", "A system of the bank, with personal bank accounts.")
            }
        }
    }

    BiRel(customerA, SystemAA, "Uses")
    BiRel(SystemAA, SystemE, "Uses")
    Rel(SystemAA, SystemC, "Sends e-mails", "SMTP")
    Rel(SystemC, customerA, "Sends e-mails
```
---
```
classDiagram
    direction LR
    class Customer {
        +String name
        +String email
        +register()
        +placeOrder() Order
    }
    class Order {
        +Date createdAt
        +List~OrderItem~ items
        +addItem(Product product, int quantity)
        +total() float
    }
    class OrderItem {
        +int quantity
        +float unitPrice
    }
    class Product {
        +String name
        +float price
    }
    class PaymentMethod {
        <<interface>>
        +authorize(float amount) bool
    }
    class CreditCard {
        +String maskedNumber
        +authorize(float amount) bool
    }
    class GiftCard {
        +float balance
        +authorize(float amount) bool
    }

    Customer "1" --> "0..*" Order : places
    Order "1" *-- "1..*" OrderItem : contains
    OrderItem "0..*" --> "1" Product : refers to
    PaymentMethod <|.. CreditCard
    PaymentMethod <|.. GiftCard
    Order --> PaymentMethod : paid via

    note for PaymentMethod "New payment providers only
need to implement authorize()"
```
---

```
sequenceDiagram
    autonumber
    actor Customer
    participant Shop as Web Shop
    participant Pay as Payment Service
    participant Bank

    Customer->>Shop: Place order
    activate Shop
    Shop->>Pay: Create payment request
    activate Pay
    Pay->>Bank: Authorize card
    Bank-->>Pay: Authorization result
    alt Payment approved
        Pay-->>Shop: Payment confirmed
        Shop-->>Customer: Show receipt
    else Payment declined
        Pay-->>Shop: Payment failed
        Shop-->>Customer: Ask for another card
    end
    deactivate Pay
    deactivate Shop
```


