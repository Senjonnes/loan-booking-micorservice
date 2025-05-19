
# Granular Explanation Of Loan Approval Endpoint

This code implements a RESTful API endpoint for confirming loan actions (approval or rejection) with the following components:

### Controller Layer

- Defines a POST endpoint at `/confirm-action`
- Accepts a LoanCheckerRequestDto in the request body and an optional loanType query parameter
- Uses the @DuplicatePrevention annotation to prevent duplicate submissions
- Routes to the appropriate service (corporate or individual) based on the loan type
- Returns the response from the service

### Service Layer (Corporate and Individual)

Both services follow a similar pattern:

1. **User Context and Concurrency Control**:

- Get the current user profile and token
- Acquire a Redis lock to prevent concurrent approvals of the same loan
- If lock acquisition fails, throw an exception

2. **Loan Request Retrieval**:

- Find the loan request by ID, application stage (IN_REVIEW), request status (ACTIVE), and isLoanRequest flag
- If not found, throw a NotFoundException

3. **Reviewer Information**:

- Set reviewer information (name, ID, branch, timestamp)

4. **Rejection Flow**:

- If approval status is IN_ISSUE (rejection):

  - Update application stage to IN_ISSUE and status to CLOSED_REJECTED
  - Update last modified information
  - Save the updated loan request
  - Generate in-app notification event (LOAN_APPROVAL_REJECTION_EVENT)
  - Send email notifications to borrower and initiator
  - Log a successful audit
  - Return a response with the updated loan request

5. **Approval Flow**:

- If approval status is APPROVED:

  - For credit line products:

    - Change loan object status to APPROVED
    - Create loan record without disbursement
    - Save the updated loan request
    - Log loan activities

- For regular loans:

  - Initiate disbursement and approval process
  - Log a successful audit
  - Return a response with the updated loan request

### Individual Loan Service - Specific Features

The individual loan service includes additional validation:

- Validates that the loan request type is BOOKING
- If not, throws a BadRequestException

### Corporate Loan Service - Specific Features

The corporate loan service has more complex disbursement handling:

- Supports credit line products (approved without disbursement)
- Handles tranched disbursements with AdhocTranche method

### Disbursement Process

The disbursement process is handled by the initiateDisbursementAndApproval method, which:

1. **Disbursement Method Determination**:

- If initiateDisbursement is false:

  - Change loan object status to APPROVED
  - Create loan record in the request table without disbursement
  - Save the updated loan request
  - Send loan approval notification
  - Log loan activities

- If disbursement date is in the future:

  - Change loan object status to APPROVED
  - Update disburse date
  - Approve and update tranches
  - Save the updated loan request
  - Send loan approval notification
  - Log loan activities

- If immediate disbursement:

  - Change loan object status to APPROVED
  - Approve and update tranches
  - Call disburseLoanToCustomer to handle the actual disbursement
  - Log loan activities

### Loan Disbursement Process

The disburseLoanToCustomer method (from the attachment) handles the actual disbursement with these steps:

1. **Portfolio Validation**:

- Validate that the loan doesn't exceed the portfolio limit

2. **Loan Detail Creation**:

- Create a LoanDetailDto with loan information

3. **Equity Contribution Handling**:

- Check if equity contribution is required and funded

4. **Tranche Processing**:

- Process each tranche that is approved and pending disbursement
- Create credit and debit entries for principal disbursement
- Apply upfront charges and taxes for the first tranche

5. **Journal Posting**:

- Check for failed journal postings
- If found, check the actual transaction status
- If transaction was successful, use the existing transaction
- If no transaction found, post a new journal entry
- Process equity contribution if required

6. **Persistence**:

- Persist loan updates after successful disbursement
- Handle any exceptions with transaction reversal

7. **Error Handling**:

- If any exception occurs, reverse any successful journal transactions
- Reset loan account if not a placement loan
- Log the error and throw an InternalErrorException

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /confirm-action"| B["Controller"]

    B --> C{"Is Corporate Loan?"}

    C -->|"Yes"| D["Corporate Loan Service"]
    C -->|"No"| E["Individual Loan Service"]

    D & E --> F["Get User Context"]

    F --> G["Acquire Redis Lock"]

    G -->|"Lock Failed"| H["Throw Exception"]

    G -->|"Lock Acquired"| I["Find Loan Request"]

    I -->|"Not Found"| J["Throw NotFoundException"]

    I -->|"Found"| K["Set Reviewer Information"]

    K --> L{"Approval Status?"}

    L -->|"IN_ISSUE (Rejected)"| M["Update Request Status to CLOSED_REJECTED"]
    M --> N["Generate Rejection Notifications"]
    N --> O["Log Rejection Audit"]

    L -->|"APPROVED"| P{"Is Corporate Loan?"}

    P -->|"Yes"| Q{"Is Credit Line?"}
    Q -->|"Yes"| R["Change Status to Approved"]
    R --> S["Create Loan Record Without Disbursement"]

    Q -->|"No"| T["Initiate Disbursement and Approval"]

    P -->|"No"| U["Validate Loan Request Type"]
    U -->|"Not Booking"| V["Throw BadRequestException"]
    U -->|"Is Booking"| W["Initiate Disbursement and Approval"]

    T & W --> X["Check Disbursement Method"]

    X --> Y{"Disbursement Method?"}

    Y -->|"Initiate Disbursement = false"| Z["Change Status to Approved"]
    Z --> AA["Create Loan Record Without Disbursement"]
    AA --> AB["Send Approval Notification"]

    Y -->|"Future Scheduled Date"| AC["Change Status to Approved"]
    AC --> AD["Update Disburse Date"]
    AD --> AE["Approve and Update Tranches"]
    AE --> AF["Send Approval Notification"]

    Y -->|"Immediate Disbursement"| AG["Change Status to Approved"]
    AG --> AH["Approve and Update Tranches"]
    AH --> AI["Disburse Loan to Customer"]

    AI --> AJ["Validate Portfolio Limit"]
    AJ -->|"Limit Exceeded"| AK["Throw BadRequestException"]

    AJ -->|"Within Limit"| AL["Create Loan Detail DTO"]

    AL --> AM{"Is Equity Contribution Required?"}
    AM -->|"Yes"| AN["Check if Equity Contribution Funded"]
    AM -->|"No"| AO["Skip Equity Check"]

    AN & AO --> AP["Process Tranches for Disbursement"]

    AP --> AQ["Create Credit and Debit Entries"]

    AQ --> AR["Apply Upfront Charges and Taxes"]

    AR --> AS["Check for Failed Journal Postings"]

    AS -->|"Found Failed Posting"| AT["Check Actual Transaction Status"]
    AT -->|"Transaction Successful"| AU["Use Existing Transaction"]
    AT -->|"No Transaction"| AV["Post New Journal Entry"]

    AS -->|"No Failed Posting"| AV

    AU & AV --> AW["Process Equity Contribution if Required"]

    AW --> AX["Persist Loan Updates"]

    AX --> AY["Log Loan Activities"]

    S & AB & AF & AY --> AZ["Log Success Audit"]

    AZ --> BA["Release Redis Lock"]

    BA --> BB["Return Success Response"]

    O --> BA
```





# Loan Review Endpoint

This code implements a RESTful API endpoint for requesting loan approval with the following components:

### Controller Layer

- Defines a POST endpoint at `/request-approval/{loanId}`
- Accepts a loan ID in the path and an optional loanType query parameter
- Routes to the appropriate service (corporate or individual) based on the loan type
- Returns the response from the service


### Service Layer - Common Flow

Both services follow a similar pattern:

1. **User Context**: Retrieves the current user profile
2. **Loan Retrieval**: Finds the existing loan request by ID and PENDING_APPROVAL status
3. **Status Update**: Sets the application stage to IN_REVIEW
4. **Metadata Update**: Updates the last modified information
5. **Audit Preparation**: Creates an audit meta info object with loan details
6. **Audit Preparation**: Creates an audit meta info object with loan details
7. **Super User Logic**:

   - If the user is a super user:

     - Creates a checker request with APPROVED status
     - Logs loan activities
     - Initiates disbursement and approval directly
     - Records audit information
     - Returns response

   - If the user is not a super user:

     - Saves the updated loan request
     - Creates response DTO
     - Records audit information
     - Logs loan activities
     - For corporate loans, sends email notification
     - Returns response

## Detailed Flow Diagram
```mermaid
graph TD
    A["Client Request"] -->|"POST /request-approval/{loanId}"| B["Controller"]

    B --> C{"Is Corporate Loan?"}

    C -->|"Yes"| D["Corporate Loan Service"]
    C -->|"No"| E["Individual Loan Service"]

    D & E --> F["Get User Context"]

    F --> G["Find Loan Request by ID and Status"]

    G -->|"Not Found"| H["Throw NotFoundException"]

    G -->|"Found"| I["Set Application Stage to IN_REVIEW"]

    I --> J["Update Last Modified Information"]

    J --> K["Set isLoanRequest to true"]

    K --> L{"Is User Superuser?"}

    L -->|"Yes"| M["Create Checker Request DTO"]
    M --> N["Set Approval Status to APPROVED"]
    N --> O["Log Loan Activities (INITIATED)"]
    O --> P["Call initiateDisbursementAndApproval"]

    P --> Q{"Disbursement Method?"}

    Q -->|"Initiate Disbursement = false"| R["Change Status to Approved"]
    R --> S["Create Loan Record Without Disbursement"]
    S --> T["Send Approval Notification"]
    T --> U["Log Loan Activities (APPROVED)"]

    Q -->|"Future Scheduled Date"| V["Change Status to Approved"]
    V --> W["Update Disburse Date"]
    W --> X["Approve and Update Tranches"]
    X --> Y["Send Approval Notification"]
    Y --> Z["Log Loan Activities (APPROVED)"]

    Q -->|"Immediate Disbursement"| AA["Change Status to Approved"]
    AA --> AB["Approve and Update Tranches"]
    AB --> AC["Disburse Loan to Customer"]
    AC --> AD["Log Loan Activities (DISBURSED)"]

    U & Z & AD --> AE["Log Success Audit"]

    L -->|"No"| AF["Save Updated Loan Request"]

    AF --> AG["Create Loan Response DTO"]

    AG --> AH["Log Success Audit"]

    AH --> AI["Log Loan Activities (INITIATED)"]

    AI --> AJ{"Is Corporate Loan?"}

    AJ -->|"Yes"| AK["Send Email Notification"]

    AJ -->|"No"| AL["Skip Email Notification"]

    AK & AL --> AM["Return Success Response"]

    AE --> AM

    P -.->|"Exception"| AN["Reset Loan Status"]
    AN --> AO["Throw InternalErrorException"]
```
