# Granular Explanation Of Bulk Loan Interest Rate Change Endpoint

This code implements a RESTful API endpoint for uploading an Excel file
containing loan accounts and interest rates, and then processing the loan repayment schedules
with the updated interest rates. Here's a detailed explanation:

### Controller Layer

- Defines a POST endpoint at `/upload`
- Accepts a MultipartFile parameter named "file"
- Calls the loan repayment schedule service's updateBulkLoanInterestRates method
- Wraps the result in a response map with status code, status description, and data
- Returns the response

### Service Layer - processAndLogInterestRateUpdate

- Logs the interest rate update
- Stores the old interest rate
- Updates the loan interest rate
- Calls processRepaymentSchedule to update the repayment schedule
- Logs the completion of the interest rate update

### Service Layer - processRepaymentSchedule

- Sets the recalculation start date to today
- Implements pagination to handle large numbers of schedules efficiently
- Fetches unpaid schedules in paginated batches
- For each batch:

  - If there are no schedules, breaks the loop
  - Calls updateSchedules to update the schedules
  - Increments the page counter
  - Continues until all pages are processed

### Service Layer - updateSchedules

- Creates a list for new schedules
- Handles different repayment patterns:

  - For REDUCING_BALANCE pattern:

    - Gets the effective loan details
    - Finds the previous outstanding principal
    - Calculates the first installment interest and total amount
    - For each schedule:

      - Marks the old schedule as deleted
      - Calculates the installment interest based on position
      - Creates a new schedule with updated values
      - Adds it to the list of new schedules

    - For other patterns (e.g.,FIXED or EQUAL REPAYMENT):

      - Calculates the effective rate
      - For each schedule:

        - Marks the old schedule as deleted
        - Calculates the installment interest and total amount
        - Creates a new schedule with updated values
        - Adds it to the list of new schedules

- Saves all the old schedules (marked as deleted) using soft delete principles
- Saves all the new schedules
- Logs the successful update

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /upload"| B["Controller"]
    B -->|"MultipartFile"| C["Loan Repayment Schedule Service"]

    C --> D["Convert Excel to LoanInterestRateDto List"]
    D -->|"Empty List"| E["Throw ExcelProcessingException"]
    D -->|"Valid List"| F["Process Bulk Interest Rate"]

    F --> G["Extract Distinct Loan Accounts"]
    G --> H["Fetch Loans by Account Numbers"]
    H --> I["Create Loan Account Map"]

    I --> J["For Each Loan Interest Rate DTO"]
    J --> K["Get Loan from Map"]

    K -->|"Loan Not Found"| L["Log Error & Continue"]
    K -->|"Loan Found"| M["Validate Interest Rate Limit"]

    M -->|"Invalid Rate"| N["Log Warning & Continue"]
    M -->|"Valid Rate"| O["Process and Log Interest Rate Update"]

    O --> P["Store Old Interest Rate"]
    P --> Q["Update Loan Interest Rate"]
    Q --> R["Process Repayment Schedule"]

    R --> S["Set Recalculation Start Date to Today"]
    S --> T["Fetch Unpaid Schedules in Batches"]

    T -->|"No Schedules"| U["Log Info & End"]
    T -->|"Has Schedules"| V["Update Schedules"]

    V --> W{"Repayment Pattern?"}

    W -->|"REDUCING_BALANCE"| X["Get Effective Loan Details"]
    X --> Y["Find Previous Outstanding Principal"]
    Y --> Z["Calculate First Installment Interest"]
    Z --> AA["For Each Schedule"]

    W -->|"Other Pattern"| AB["Calculate Effective Rate"]
    AB --> AC["For Each Schedule"]

    AA & AC --> AD["Mark Old Schedule as Deleted"]
    AD --> AE["Calculate New Interest & Total Amount"]
    AE --> AF["Create New Schedule"]
    AF --> AG["Add to New Schedules List"]

    AG --> AH["Save All Old Schedules"]
    AH --> AI["Save All New Schedules"]
    AI --> AJ["Log Success"]

    AJ --> AK["Return to Next DTO or Complete"]
    AK --> AL["Return Result to Controller"]
    AL --> AM["Return Response"]
```

# Granular Explanation Of Single Loan Interest Rate Change Request Endpoint

The service method is annotated with `@Transactional` to ensure database consistency and follows these steps:

### Controller Layer

- Defines a POST endpoint at `/single/interest-rate/request` for the maker of this request
- Accepts a LoanInterestRateDto in the request body
- Calls the loan repayment schedule service's updateSingleInterestRate method
- Returns the response

### Service Layer

The service method follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority of INTEREST_RATE_MODIFICATION_REQUEST_AUTHORITY to update loan interest rates

3. **Loan Retrieval**:

   - Fetches the loan by account number
   - If not found, an exception would be thrown by the fetchLoan method

4. **Interest Rate Information**:

   - Stores the old interest rate from the loan
   - Gets the new interest rate from the DTO
   - Logs the interest rate change

5. **Request Creation**:

   - Builds and persists a loan request model with:

     - The loan information
     - The user profile
     - Request type INTEREST_RATE_MODIFICATION
     - The new interest rate

6. **Interest Rate Validation**:

   - Validates the new interest rate against the product maximum limit
   - If invalid, throws an exception

7. **Super User Logic**:

   - If the user is a super user:

     - Creates a request review DTO with APPROVED status
     - Calls automaticallyApproveInterestRate to approve the interest rate change immediately
     - Returns the approval response

   - If the user is not a super user:

     - Logs audit information for the successful request
     - Logs loan activities with state "INTEREST RATE MODIFICATION REQUEST"
     - Returns a success response indicating the request was created

8. **Exception Handling**:

   - Catches exceptions
   - Logs audit information for the failure
   - Rethrows the exception

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /single/interest-rate/request"| B["Controller"]
    B -->|"LoanInterestRateDto"| C["Loan Repayment Schedule Service"]

    C --> D["Get User Profile"]
    D --> E["Validate User Authority"]
    E -->|"Invalid Authority"| F["Throw Exception"]
    E -->|"Valid Authority"| G["Fetch Loan by Account Number"]

    G -->|"Loan Not Found"| H["Throw Exception"]
    G -->|"Loan Found"| I["Store Old Interest Rate"]

    I --> J["Get New Interest Rate from DTO"]
    J --> K["Build & Persist Loan Request Model"]
    K --> L["Validate New Interest Rate Against Product Max Limit"]

    L -->|"Invalid Rate"| M["Throw Exception"]
    L -->|"Valid Rate"| N{"Is Super User?"}

    N -->|"Yes"| O["Create Request Review DTO with APPROVED Status"]
    O --> P["Automatically Approve Interest Rate Change"]
    P --> Q["Return Approval Response"]

    N -->|"No"| R["Audit Success"]
    R --> S["Log Loan Activities"]
    S --> T["Return Success Response"]

    C -.->|"Exception"| U["Audit Failure"]
    U --> V["Throw Exception"]
```

# Granular Explanation Of Single Loan Interest Rate Change Review Endpoint

This code implements a RESTful API endpoint for reviewing and approving/rejecting an interest rate change request for a single loan with the following components:

### Controller Layer

- Defines a POST endpoint at `/single/interest-rate/review` for the checker
- Accepts a RequestReviewDto in the request body
- Calls the loan repayment schedule service's reviewSingleInterestRate method
- Returns the response

### Service Layer - reviewSingleInterestRate

The service method is annotated with `@Transactional` to ensure database consistency and follows these steps:

1. **User Context**: Retrieves the current user profile
2. **Authorization Validation**:

   - Validates that the user has the right authority of INTEREST_RATE_MODIFICATION_REVIEW_AUTHORITY to review interest rate changes

3. **Request Retrieval**:

   - Gets the loan request using the request ID from the DTO
   - If not found, an exception would be thrown by the getTheLoanRequest method

4. **Approval Logic**:

   - If the approval status is REJECTED:

     - Updates and stores the loan request with status CLOSED_REJECTED and state IN_ISSUE
     - Sets response message to "Interest Rate Change Request Rejected"
     - Sets activity state to "REJECTED INTEREST RATE CHANGE REQUEST"
     - Sets activity comment to "Rejected interest rate change by "
     - Sets next action to "REJECTED"

   - If the approval status is APPROVED:

     - Finds the loan by loan account number
     - If not found, throws a BadRequestException
     - Updates the loan interest rate
     - Calls processRepaymentSchedule to update the repayment schedule
     - Updates and stores the loan request with status APPROVED and state APPROVED
     - Sets response message to "Interest Rate Change Request approved"
     - Sets activity state to "APPROVED INTEREST RATE CHANGE"
     - Sets activity comment to "Approved interest rate change by "
     - Sets next action to "APPROVED"

5. **Auditing and Logging**:

   - Logs audit information
   - Logs loan activities

6. **Exception Handling**:

   - Catches exceptions
   - Logs errors
   - Logs audit information for failures
   - Rethrows the exception

### Service Layer - processRepaymentSchedule

This method handles the updating of repayment schedules:

- Sets the recalculation start date to today
- Implements pagination to handle large numbers of schedules efficiently
- Fetches unpaid schedules in paginated batches
- For each batch:

  - If there are no schedules, breaks the loop
  - Calls updateSchedules to update the schedules
  - Increments the page counter
  - Continues until all pages are processed

### Service Layer - updateSchedules

This method handles the actual schedule updates based on the loan's repayment pattern:

- Creates a list for new schedules
- For REDUCING_BALANCE pattern:

- Gets the effective loan details
- Finds the previous outstanding principal
- Calculates the first installment interest and total amount
- For each schedule:

  - Marks the old schedule as deleted
  - Calculates the installment interest based on position
  - Creates a new schedule with updated values
  - Adds it to the list of new schedules

- For other patterns (e.g., FIXED or EQUAL REPAYMENT):

- Calculates the effective rate
- For each schedule:

  - Marks the old schedule as deleted
  - Calculates the installment interest and total amount
  - Creates a new schedule with updated values
  - Adds it to the list of new schedules

- Saves all the old schedules (marked as deleted) using soft delete principle
- Saves all the new schedules
- Logs the successful update

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /single/interest-rate/review"| B["Controller"]
    B -->|"RequestReviewDto"| C["Loan Repayment Schedule Service"]

    C --> D["Get User Profile"]
    D --> E["Validate User Authority"]
    E -->|"Invalid Authority"| F["Throw Exception"]
    E -->|"Valid Authority"| G["Get Loan Request by ID"]

    G -->|"Request Not Found"| H["Throw Exception"]
    G -->|"Request Found"| I{"Approval Status?"}

    I -->|"REJECTED"| J["Update Loan Request Status to CLOSED_REJECTED"]
    J --> K["Set Response for Rejection"]
    K --> L["Set Activity State to 'REJECTED INTEREST RATE CHANGE REQUEST'"]

    I -->|"APPROVED"| M["Find Loan by Account Number"]
    M -->|"Loan Not Found"| N["Throw BadRequestException"]
    M -->|"Loan Found"| O["Update Loan Interest Rate"]

    O --> P["Process Repayment Schedule"]
    P --> Q["Update Loan Request Status to APPROVED"]
    Q --> R["Set Response for Approval"]
    R --> S["Set Activity State to 'APPROVED INTEREST RATE CHANGE'"]

    L & S --> T["Audit Success"]
    T --> U["Log Loan Activities"]
    U --> V["Return Success Response"]

    C -.->|"Exception"| W["Log Error"]
    W --> X["Audit Failure"]
    X --> Y["Throw Exception"]

    P --> P1["Set Recalculation Start Date to Today"]
    P1 --> P2["Fetch Unpaid Schedules in Batches"]

    P2 -->|"No Schedules"| P3["Log Info & Continue"]
    P2 -->|"Has Schedules"| P4["Update Schedules"]

    P4 --> P5{"Repayment Pattern?"}

    P5 -->|"REDUCING_BALANCE"| P6["Get Effective Loan Details"]
    P6 --> P7["Find Previous Outstanding Principal"]
    P7 --> P8["Calculate First Installment Interest"]
    P8 --> P9["For Each Schedule"]

    P5 -->|"Other Pattern"| P10["Calculate Effective Rate"]
    P10 --> P11["For Each Schedule"]

    P9 & P11 --> P12["Mark Old Schedule as Deleted"]
    P12 --> P13["Calculate New Interest & Total Amount"]
    P13 --> P14["Create New Schedule"]
    P14 --> P15["Add to New Schedules List"]

    P15 --> P16["Save All Old Schedules"]
    P16 --> P17["Save All New Schedules"]
    P17 --> P18["Return to Main Flow"]

    P3 --> P18
    P18 --> Q
```
