# Save To Draft Endpoint

This section implements a RESTful API endpoint for saving loan drafts with the following components:

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
