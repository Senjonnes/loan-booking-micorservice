# Save To Draft Endpoint Granular Explanation

This section implements a RESTful API endpoint for saving loan drafts with the following components:

### Controller Layer

- Defines a POST endpoint at `/loan-drafts/save`
- Accepts a loan type parameter and request body
- Validates the request and handles validation errors
- Routes the request to either corporate or individual loan service based on the loan type

### Service Layer (Corporate and Individual)

Both services follow a similar pattern:

1. **User Context**: Retrieves the current user profile and authentication token
2. **Data Transformation**: Converts the request DTO to a loan entity
3. **Business Logic**:

4. If a product ID is provided:

5. Retrieves product details
6. Checks if the customer is within allowed obligor limits
7. Sets loan properties (application stage, status, type, etc.)

8. **Persistence**: Saves the loan draft to the database
9. **Relationship Handling**:

   - Associates collaterals with the loan if provided
   - Associates additional facility details if provided

10. **Auditing and Logging**:

    - Logs loan activities
    - Records audit information for successful operations

11. **Exception Handling**: Catches exceptions, logs audit information for failures, and throws a user-friendly error

## Detailed Flow Diagram

```mermaid
graph TD
    A["Client Request"] -->|"POST /loan-drafts/save"| B["Controller"]
    B -->|"Validate Request"| C{"Has Validation Errors?"}
    C -->|"Yes"| D["Return Bad Request"]
    C -->|"No"| E{"Loan Type?"}

    E -->|"CORPORATE"| F["Corporate Loan Service"]
    E -->|"INDIVIDUAL"| G["Individual Loan Service"]

    F --> H["Get Current User Profile"]
    G --> H

    H --> I["Convert Request to Draft Entity"]
    I --> J{"Has Product ID?"}

    J -->|"Yes"| K["Retrieve Product Details"]
    K --> L["Check Obligor Limits"]
    J -->|"No"| M["Set Loan Properties"]
    L --> M

    M --> N["Save Draft Loan To Request Table"]
    N --> O["Create Response DTO"]

    O --> P{"Has Collaterals?"}
    P -->|"Yes"| Q["Associate Collaterals"]
    P -->|"No"| R{"Has Additional Facility?"}
    Q --> R

    R -->|"Yes"| S["Associate Additional Facility"]
    R -->|"No"| T["Log Loan Activity"]
    S --> T

    T --> U["Send Logs To Audit Trail"]
    U --> V["Return Success Response"]

    F -.->|"Exception"| W["Audit Logging Failure"]
    G -.->|"Exception"| W
    W --> X["Throw Internal Error Exception"]
```

## Update Draft Endpoint

This section implements a RESTful API endpoint for saving loan drafts with the following components:

### Controller Layer

-Defines a PUT endpoint at `/loan-drafts/{loanId}`
-Accepts a loan ID path variable, loan type parameter, and request body
-Validates the request and handles validation errors
-Routes the request to either corporate or individual loan service based on the loan type

### Service Layer (Corporate and Individual)

Both services follow a similar pattern:

1. **User Context**: Retrieves the current user profile and authentication token
2. **Data Transformation**: Converts the request DTO to a loan entity
3. **Business Logic**:

4. If a product ID is provided:

5. Retrieves product details
6. Checks if the customer is within allowed obligor limits
7. Sets loan properties (application stage, status, type, etc.)

8. **Persistence**: Saves the loan draft to the database
9. **Relationship Handling**:

   - Associates collaterals with the loan if provided
   - Associates additional facility details if provided

10. **Auditing and Logging**:

    - Logs loan activities
    - Records audit information for successful operations

11. **Exception Handling**: Catches exceptions, logs audit information for failures, and throws a user-friendly error

## Detailed Flow Diagram

```mermaid

graph TD
    A["Client Request"] -->|"PUT /loan-drafts/{loanId}"| B["Controller"]
    B -->|"Validate Request"| C{"Has Validation Errors?"}
    C -->|"Yes"| D["Return Bad Request"]
    C -->|"No"| E{"Loan Type?"}

    E -->|"CORPORATE"| F["Corporate Loan Service"]
    E -->|"INDIVIDUAL"| G["Individual Loan Service"]

    F --> H["Find Existing Loan"]
    G --> H

    H -->|"Not Found"| I["Throw NotFoundException"]
    H -->|"Found"| J["Get Current User Profile"]

    J --> K["Check Obligor Limits"]
    K --> L["Capture Original Loan State"]

    L --> M{"Has Product ID?"}
    M -->|"Yes"| N["Retrieve Product Details"]
    M -->|"No"| O["Update Loan Properties"]
    N --> O

    O --> P["Set Metadata Fields"]
    P --> Q["Log Changed Fields"]
    Q --> R["Save Updated Loan"]

    R --> S["Create Response DTO"]

    G -->|"Individual Loan"| T{"Has Collaterals?"}
    T -->|"Yes"| U["Remove Old Collaterals"]
    U --> V["Add New Collaterals"]
    T -->|"No"| W["Log Loan Activity"]
    V --> W

    S --> W
    F -->|"Corporate Loan"| W

    W --> X["Audit Success"]
    X --> Y["Return Success Response"]

    F -.->|"Exception"| Z["Audit Failure"]
    G -.->|"Exception"| Z
    Z --> AA["Throw Internal Error Exception"]

```
