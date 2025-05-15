# Full Loan Liquidation Request Endpoint

This code implements a RESTful API endpoint for requesting a full loan liquidation with the following components:

### Controller Layer

- Defines a POST endpoint at `/full-liquidation/request`
- Accepts a loan liquidation request DTO
- Validates the request body using `@Valid` annotation
- Routes the request to the loan liquidation service's requestFullLoanLiquidation method

### Service Layer

The service method is annotated with `@Transactional(rollbackFor = Exception.class)` to ensure database consistency and `@SneakyThrows` to handle exceptions. It follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority to request a loan liquidation

3. **Pending Approval Validation**:

   - Validates that there are no pending approval requests for the loan

4. **Loan Retrieval**:

   - Gets the disbursed loan by ID

5. **Moratorium Handling**:

   - If the loan has a moratorium requirement but no moratorium type, sets it to "Principal_and_interest"

6. **Balance Calculation**:

   - Calculates the outstanding balance of the loan

7. **Validation**:

   - Validates the outstanding balance
   - Validates that the prepayment amount is valid for a full liquidation
   - If not an overdraw liquidation, validates that the customer can liquidate the loan

8. **Request Creation**:

   - Builds and persists a loan request model with the appropriate request type (OVERDRAW_LIQUIDATION or FULL_LIQUIDATION)

9. **Liquidation Model Creation**:

   - Builds and persists a loan liquidation model with details of the liquidation

10. **Super User Logic**:

    - If the user is a super user:

      - Automatically approves the loan liquidation
      - Returns the approval response

    - If the user is not a super user:

      - Logs audit information
      - Logs loan activities
      - Returns a success response

11. **Exception Handling**: Catches exceptions, logs audit information for failures, and rethrows the exception

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /full-liquidation/request"| B["Controller"]
    B -->|"Validate Request Body"| C["Loan Liquidation Service"]

    C --> D["Get User Profile"]
    D --> E["Validate User Authority"]
    E -->|"Invalid Authority"| F["Throw Exception"]
    E -->|"Valid Authority"| G["Validate No Pending Approval"]

    G -->|"Pending Approval Exists"| H["Throw Exception"]
    G -->|"No Pending Approval"| I["Get Disbursed Loan"]

    I -->|"Loan Not Found"| J["Throw Exception"]
    I -->|"Loan Found"| K["Handle Loan Moratorium"]

    K --> L["Calculate Outstanding Balance"]
    L --> M["Validate Outstanding Balance"]

    M -->|"Invalid Balance"| N["Throw Exception"]
    M -->|"Valid Balance"| O["Validate Prepayment Amount"]

    O -->|"Invalid Amount"| P["Throw Exception"]
    O -->|"Valid Amount"| Q{"Is Overdraw?"}

    Q -->|"No"| R["Validate Customer Can Liquidate"]
    R -->|"Cannot Liquidate"| S["Throw Exception"]
    R -->|"Can Liquidate"| T["Build & Persist Loan Request"]

    Q -->|"Yes"| T

    T --> U["Build & Persist Loan Liquidation Model"]

    U --> V{"Is Super User?"}

    V -->|"Yes"| W["Automatically Approve Liquidation"]
    W --> X["Return Approval Response"]

    V -->|"No"| Y["Audit Success"]
    Y --> Z["Log Loan Activities"]
    Z --> AA["Return Success Response"]

    C -.->|"Exception"| AB["Audit Failure"]
    AB --> AC["Throw Exception"]
```

# Partial Loan Liquidation Request Endpoint

This code implements a RESTful API endpoint for requesting a partial loan liquidation with the following components:

### Controller Layer

- Defines a POST endpoint at `/partial-liquidation/request`
- Accepts a loan liquidation request DTO
- Validates the request body using `@Valid` annotation
- Routes the request to the loan liquidation service's requestPartialLoanLiquidation method

### Service Layer

The service method is annotated with `@Transactional(rollbackFor = Exception.class)` to ensure database consistency and `@SneakyThrows` to handle exceptions. It follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority to request a loan liquidation

3. **Pending Approval Validation**:

   - Validates that there are no pending approval requests for the loan

4. **Loan Retrieval**:

   - Gets the disbursed loan by ID

5. **Moratorium Handling**:

   - If the loan has a moratorium requirement but no moratorium type, sets it to "Principal_and_interest"

6. **Balance Calculation**:

   - Calculates the outstanding balance of the loan

7. **Validation**:

   - Validates the outstanding balance
   - Validates that the prepayment amount is valid for a partial liquidation
   - Validates that the customer can liquidate the loan
   - If it's a principal repayment, validates that there are no past due obligations

8. **Request Creation**:

   - Builds and persists a loan request model with the request type PARTIAL_LIQUIDATION

9. **Liquidation Model Creation**:

   - Builds and persists a loan liquidation model with details of the liquidation

10. **Super User Logic**:

    - If the user is a super user:

      - Automatically approves the loan liquidation
      - Returns the approval response

    - If the user is not a super user:

      - Logs audit information
      - Logs loan activities
      - Returns a success response

11. **Exception Handling**: Catches exceptions, logs audit information for failures, and rethrows the exception

## Detailed Flow Diagram

```mermaid

graph TD
    A["Client Request"] -->|"POST /partial-liquidation/request"| B["Controller"]
    B -->|"Validate Request Body"| C["Loan Liquidation Service"]

    C --> D["Get User Profile"]
    D --> E["Validate User Authority"]
    E -->|"Invalid Authority"| F["Throw Exception"]
    E -->|"Valid Authority"| G["Validate No Pending Approval"]

    G -->|"Pending Approval Exists"| H["Throw Exception"]
    G -->|"No Pending Approval"| I["Get Disbursed Loan"]

    I -->|"Loan Not Found"| J["Throw Exception"]
    I -->|"Loan Found"| K["Handle Loan Moratorium"]

    K --> L["Calculate Outstanding Balance"]
    L --> M["Validate Outstanding Balance"]

    M -->|"Invalid Balance"| N["Throw Exception"]
    M -->|"Valid Balance"| O["Validate Prepayment Amount"]

    O -->|"Invalid Amount"| P["Throw Exception"]
    O -->|"Valid Amount"| Q["Validate Customer Can Liquidate"]

    Q -->|"Cannot Liquidate"| R["Throw Exception"]
    Q -->|"Can Liquidate"| S{"Is Principal Repayment?"}

    S -->|"Yes"| T{"Has Past Due Obligations?"}
    T -->|"Yes"| U["Throw LoanLiquidationException"]
    T -->|"No"| V["Build & Persist Loan Request"]

    S -->|"No"| V

    V --> W["Build & Persist Loan Liquidation Model"]

    W --> X{"Is Super User?"}

    X -->|"Yes"| Y["Automatically Approve Liquidation"]
    Y --> Z["Return Approval Response"]

    X -->|"No"| AA["Audit Success"]
    AA --> AB["Log Loan Activities"]
    AB --> AC["Return Success Response"]

    C -.->|"Exception"| AD["Audit Failure"]
    AD --> AE["Throw Exception"]
```

# Full Loan Liquidation Review Endpoint

This code implements a RESTful API endpoint for reviewing a full loan liquidation request with the following components:

### Controller Layer

- Defines a POST endpoint at `/full-liquidation/review`
- Uses `@DuplicatePrevention` annotation to prevent duplicate submissions
- Accepts a request review DTO
- Validates the request body using `@Valid` annotation
- Routes the request to the loan liquidation service's reviewFullLoanLiquidation method

### Service Layer

The service method is annotated with `@Transactional(rollbackFor = Exception.class)` to ensure database consistency and `@SneakyThrows` to handle exceptions. It follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority to review loan liquidations

3. **Liquidation Retrieval**:

   - Gets the loan liquidation model by request ID

4. **Loan Retrieval**:

   - Gets the disbursed loan by ID from the liquidation model

5. **Moratorium Handling**:

   - If the loan has a moratorium requirement but no moratorium type, sets it to "Principal_and_interest"

6. **Reviewer Information**:

   - Updates the loan liquidation model with reviewer information (name, ID, date)

7. **Approval Logic**:

   - If the approval status is REJECTED:

     - Sets the liquidation status to DECLINED
     - Updates the loan request status to CLOSED_REJECTED and state to IN_ISSUE
     - Sets appropriate response messages and activity logs based on liquidation event type (OVERDRAW_LIQUIDATION or FULL_LIQUIDATION)

   - If the approval status is APPROVED:

     - Calculates the outstanding balance of the loan
     - Processes the payment
     - Persists the loan repayment schedule
     - Sets the liquidation status to APPROVED
     - Updates the liquidation with repaid amounts (principal, penalty, interest)
     - Closes the paid off loan
     - Sets the loan status to CLOSED_PAID_OFF and application stage to CLOSED
     - Updates the loan request type based on liquidation event
     - Updates the loan request status to APPROVED and state to APPROVED
     - Sets appropriate response messages and activity logs based on liquidation event type

8. **Persistence**:

   - Saves the updated loan liquidation model

9. **Auditing and Logging**:

   - Logs audit information
   - Logs loan activities

10. **Exception Handling**: Catches exceptions, logs errors, logs audit information for failures, and rethrows the exception

## Detailed Flow Diagram

```mermaid

graph TD
    A["Client Request"] -->|"POST /full-liquidation/review"| B["Controller"]
    B -->|"@DuplicatePrevention"| C["Validate Request Body"]
    C --> D["Loan Liquidation Service"]

    D --> E["Get User Profile"]
    E --> F["Validate User Authority"]
    F -->|"Invalid Authority"| G["Throw Exception"]
    F -->|"Valid Authority"| H["Get Loan Liquidation Model"]

    H -->|"Not Found"| I["Throw Exception"]
    H -->|"Found"| J["Get Disbursed Loan"]

    J -->|"Loan Not Found"| K["Throw Exception"]
    J -->|"Loan Found"| L["Handle Loan Moratorium"]

    L --> M["Update Liquidation with Reviewer Info"]

    M --> N{"Approval Status?"}

    N -->|"REJECTED"| O["Set Liquidation Status to DECLINED"]
    O --> P["Update Loan Request Status to CLOSED_REJECTED"]
    P --> Q["Set Response Messages for Rejection"]

    N -->|"APPROVED"| R["Calculate Outstanding Balance"]
    R --> S["Process Payment"]
    S --> T["Persist Loan Repayment Schedule"]
    T --> U["Set Liquidation Status to APPROVED"]
    U --> V["Update Liquidation with Repaid Amounts"]
    V --> W["Close Paid Off Loan"]
    W --> X["Set Loan Status to CLOSED_PAID_OFF"]
    X --> Y["Update Loan Request Status to APPROVED"]
    Y --> Z["Set Response Messages for Approval"]

    Q & Z --> AA["Save Loan Liquidation Model"]
    AA --> AB["Audit Success"]
    AB --> AC["Log Loan Activities"]
    AC --> AD["Return Success Response"]

    D -.->|"Exception"| AE["Log Error"]
    AE --> AF["Audit Failure"]
    AF --> AG["Throw Exception"]
```

# Partial Loan Liquidation Review Endpoint

This code implements a RESTful API endpoint for reviewing a full loan liquidation request with the following components:

### Controller Layer

- Defines a POST endpoint at `/partial-liquidation/review`
- Uses `@DuplicatePrevention` annotation to prevent duplicate submissions
- Accepts a request review DTO
- Validates the request body using `@Valid` annotation
- Routes the request to the loan liquidation service's reviewPartialLoanLiquidation method

### Service Layer

The service method is annotated with `@Transactional(rollbackFor = Exception.class)` to ensure database consistency and `@SneakyThrows` to handle exceptions. It follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority to review loan liquidations

3. **Liquidation Retrieval**:

   - Gets the loan liquidation model by request ID

4. **Loan Retrieval**:

   - Gets the disbursed loan by ID from the liquidation model

5. **Moratorium Handling**:

   - If the loan has a moratorium requirement but no moratorium type, sets it to "Principal_and_interest"

6. **Reviewer Information**:

   - Updates the loan liquidation model with reviewer information (name, ID, date)

7. **Approval Logic**:

   - If the approval status is REJECTED:

     - Calls `handlePartLiquidationRejectedRequest` to handle the rejection
     - This method likely updates the loan liquidation status to DECLINED and the loan request status to CLOSED_REJECTED
     - Sets activity state to "REJECTED PARTIAL LOAN LIQUIDATION"
     - Sets activity comment to "Rejected partial loan liquidation by "
     - Sets next action to "PERFORMING"

   - If the approval status is APPROVED:

     - Calls `handlePartLiquidationApprovedRequest` to handle the approval
     - This method likely calculates the outstanding balance, processes the payment, updates repayment schedules, and updates the loan status
     - Sets activity state to "APPROVED PARTIAL LOAN LIQUIDATION"
     - Sets activity comment to "Approved partial loan liquidation by "
     - Sets next action to "PERFORMING"

8. **Persistence**:

   - Saves the updated loan liquidation model

9. **Auditing and Logging**:

   - Logs audit information
   - Logs loan activities

10. **Exception Handling**: Catches exceptions, logs errors, logs audit information for failures, and rethrows the exception

## Detailed Flow Diagram

```mermaid

graph TD
    A["Client Request"] -->|"POST /partial-liquidation/review"| B["Controller"]
    B -->|"@DuplicatePrevention"| C["Validate Request Body"]
    C --> D["Loan Liquidation Service"]

    D --> E["Get User Profile"]
    E --> F["Validate User Authority"]
    F -->|"Invalid Authority"| G["Throw Exception"]
    F -->|"Valid Authority"| H["Get Loan Liquidation Model"]

    H -->|"Not Found"| I["Throw Exception"]
    H -->|"Found"| J["Get Disbursed Loan"]

    J -->|"Loan Not Found"| K["Throw Exception"]
    J -->|"Loan Found"| L["Handle Loan Moratorium"]

    L --> M["Update Liquidation with Reviewer Info"]

    M --> N{"Approval Status?"}

    N -->|"REJECTED"| O["Handle Partial Liquidation Rejected Request"]
    O --> P["Set Activity State to 'REJECTED PARTIAL LOAN LIQUIDATION'"]
    P --> Q["Set Activity Comment"]
    Q --> R["Set Next Action to 'PERFORMING'"]

    N -->|"APPROVED"| S["Handle Partial Liquidation Approved Request"]
    S --> T["Set Activity State to 'APPROVED PARTIAL LOAN LIQUIDATION'"]
    T --> U["Set Activity Comment"]
    U --> V["Set Next Action to 'PERFORMING'"]

    R & V --> W["Save Loan Liquidation Model"]
    W --> X["Audit Success"]
    X --> Y["Log Loan Activities"]
    Y --> Z["Return Success Response"]

    D -.->|"Exception"| AA["Log Error"]
    AA --> AB["Audit Failure"]
    AB --> AC["Throw Exception"]
```
