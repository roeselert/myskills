
## name: Business Analyst Skill
description: This guide provides a standardized approach for Business Analysts to write clear, high-quality User Stories and Acceptance Criteria


## The User Story Template

Every User Story must capture **who** wants the feature, **what** they need, and **why** it delivers business value.

*   **Title:** A concise, action-oriented phrase (e.g., *Empty Shopping Cart*).
*   **Role (Who?):** As a `[user role]`...
*   **Need (What?):** ...I need `[action/feature]`...
*   **Benefit (Why?):** ...because `[value/reason]`.

## Acceptance Criteria (Given-When-Then)

Acceptance criteria define the boundaries of a user story. They use the Behavior-Driven Development (BDD) framework to describe how the system should react from the user's perspective.

*   **Given:** The initial context, setup, or pre-condition.
*   **When:** The specific action or event triggered by the user.
*   **Then:** The expected outcome, result, or system reaction.

## Grammar & Syntax rules

DSL to specify processes and data 
Do not use boilerplate keywords (`export`, `class`, `function`), async annotations (`Promise`, `await`), or unnecessary curly braces in flows.

*   `entity Name { ... }`: Defines raw core domain models.
*   `flow name(params) { ... }`: Indentation-based, sequential pseudo-code describing the exact logic path.
*   `transaction { ... }`: Explicit wrapper within a flow ensuring all inner operations are atomic (all-or-nothing database transactions).

---

## 💡 Practical Examples

### Example 1: Password Reset
*   **Title:** Password Reset via Email Link
*   **Story:** **As a** registered customer, **I need** to request a password reset link via email, **because** I want to regain access to my account if I forget my login details.
*   **Scenario 1: Successful request**
    *   **Given:** The user is on the "Forgot Password" page.
    *   **When:** The user enters their registered email address and clicks "Submit".
    *   **Then:** The system sends an email containing a secure, time-sensitive reset link.

### Example 2: Form Validation
*   **Title:** Mandatory Field Validation during Signup
*   **Story:** **As a** new visitor, **I need** to see instant validation errors on mandatory fields, **because** I want to fix mistakes immediately and complete my registration successfully.
*   **Scenario 1: Missing required field**
    *   **Given:** The user leaves the "Last Name" field empty.
    *   **When:** The user clicks the "Register" button.
    *   **Then:** The system highlights the field in red and displays the message "Last Name is required".

```typescript
  entity Order {
    id: string
    total: number
    status: "PENDING" | "PAID"
  }

 
  flow execute(orderId: string, amount: number) {
    transaction {
        isPaid = payment.charge(amount)
        if not isPaid
          fail "Payment failed"

        db.updateStatus(orderId, "PAID")
     }
   }
 
 
```


---

## 🛠️ Best Practices for Business Analysts

*   **Apply INVEST:** Ensure your stories are Independent, Negotiable, Valuable, Estimable, Small, and Testable.
*   **Define Specific Roles:** Use detailed personas like "Premium Subscriber" or "System Administrator" instead of a generic "User".
*   **Focus on Business Value:** Avoid technical implementation details (e.g., write "The system saves the preferences" instead of "Insert data into PostgreSQL").
*   **Include Negative Scenarios:** Always write acceptance criteria for edge cases and error states, not just the happy path.
kk