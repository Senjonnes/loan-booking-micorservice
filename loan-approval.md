# Loan Approval Endpoint

This code implements a RESTful API endpoint for confirming loan actions (approval or rejection) with the following components:

### Controller Layer

- Defines a POST endpoint at `/confirm-action`
- Uses `@DuplicatePrevention` annotation to prevent duplicate submissions
- Accepts a loan checker request body and loan type parameter
- Routes the request to either corporate or individual loan service based on the loan type

### Service Layer (Corporate and Individual)

Both services follow a similar pattern:

1. **User Context**: Retrieves the current user profile and authentication token.
2. **Concurrency Control**:
   - Uses Redis to implement a distributed lock mechanism
   - Prevents multiple users from reviewing the same loan simultaneously
   - Sets an expiration time for the lock
3. **Loan Retrieval**: Finds the existing loan request by ID, application stage (IN_REVIEW), and active status
4. **Reviewer Information**: Sets reviewer details (name, ID, branch, timestamp)
5. **Audit Preparation**: Creates an audit meta info object with loan details
6. **Approval Logic**:

   - If approval status is `IN_ISSUE` (rejection):

     - Updates loan status to `CLOSED_REJECTED`
     - Sends notifications (in-app and email)
     - Logs audit information

   - If approval status is for approval:

     - For corporate loans that are credit lines:

     - Changes loan status to approved
     - Creates loan record without disbursement

   - For other loans:
     - Initiates disbursement and approval

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /confirm-action"| B["Controller"]
    B -->|"@DuplicatePrevention"| C{"Loan Type?"}

    C -->|"CORPORATE"| D["Corporate Loan Service"]
    C -->|"INDIVIDUAL"| E["Individual Loan Service"]

    D --> F["Get User Profile"]
    E --> F

    F --> G["Attempt to Acquire Lock"]
    G -->|"Lock Failed"| H["Throw NotFoundException"]
    G -->|"Lock Acquired"| I["Find Loan Request in Review State"]

    I -->|"Not Found"| J["Throw NotFoundException"]
    I -->|"Found"| K["Set Reviewer Information"]

    E -->|"Individual Loan"| L{"Is Booking Type?"}
    L -->|"No"| M["Throw BadRequestException"]
    L -->|"Yes"| N["Continue Processing"]

    K --> O{"Approval Status?"}

    O -->|"IN_ISSUE (Rejection)"| P["Set Status to CLOSED_REJECTED"]
    P --> Q["Update Last Modified Info"]
    Q --> R["Save Loan Request"]
    R --> S["Send Notifications"]
    S --> T["Audit Success"]
    T --> U["Return Response"]

    O -->|"Approval"| V{"Is Credit Line?"}

    V -->|"Yes (Corporate Only)"| W["Change Status to Approved"]
    W --> X["Create Loan Record Without Disbursement"]
    X --> Y["Save Loan Request"]
    Y --> Z["Log Loan Activities"]
    Z --> AA["Audit Success"]
    AA --> AB["Return Response"]

    V -->|"No"| AC["Initiate Disbursement and Approval"]
    AC --> AA

    N --> O

    D & E -.->|"Finally"| AD["Release Lock"]
```

# Loan Review Endpoint

This code implements a RESTful API endpoint for requesting loan approval with the following components:

### Controller Layer

- Defines a POST endpoint at `/request-approval/{loanId}`
- Accepts a loan ID path variable and loan type parameter
- Routes the request to either corporate or individual loan service based on the loan type

### Service Layer (Corporate and Individual)

### Service Layer (Corporate and Individual)

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
    B -->|"Route Request"| C{"Loan Type?"}

    C -->|"CORPORATE"| D["Corporate Loan Service"]
    C -->|"INDIVIDUAL"| E["Individual Loan Service"]

    D --> F["Get User Profile"]
    E --> F

    F --> G["Find Loan Request with PENDING_APPROVAL Status"]
    G -->|"Not Found"| H["Throw NotFoundException"]
    G -->|"Found"| I["Set Application Stage to IN_REVIEW"]

    I --> J["Update Last Modified Information"]
    J --> K["Prepare Audit Meta Info"]

    K --> L{"Is Super User?"}

    L -->|"Yes"| M["Create Checker Request with APPROVED Status"]
    M --> N["Log Loan Activities"]
    N --> O["Initiate Disbursement and Approval"]
    O --> P["Populate Audit Success"]
    P --> Q["Return Response"]

    L -->|"No"| R["Save Updated Loan Request"]
    R --> S["Create Response DTO"]
    S --> T["Populate Audit Success"]

    T --> U["Log Loan Activities"]

    D -->|"Corporate Service"| V["Send Email Notification"]
    V --> W["Return Response"]

    U --> W
```
