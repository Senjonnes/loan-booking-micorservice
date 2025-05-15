# Loan Update Endpoint

This code implements a RESTful API endpoint for updating loans with the following components:

### Controller Layer

- Defines a PUT endpoint at `/{loanId}`
- Accepts a loan ID path variable, request body, and loan type parameter
- Validates the request and handles validation errors
- Routes the request to either corporate or individual loan service based on the loan type

### Service Layer (Corporate and Individual)

Both services follow a similar pattern with extensive validation and business logic:

1. **User Context**: Retrieves the current user profile and authentication token
2. **Product Validation**:

   - Retrieves product details for the loan
   - Checks obligor limits

3. **Entity Retrieval**:

   - Attempts to find either a loan request or a loan entity by ID
   - Throws NotFoundException if neither exists

4. **Data Transformation**:

   - Copies non-null properties from the request to the entity
   - Captures the original state for change tracking

5. **Portfolio Management** (Individual service only):

   - Updates consumed portfolio if principal amount changes

6. **Business Rule Validation**:

   - Validates principal amount is within product min/max
   - Validates loan tenor is within product min/max
   - Validates disbursement date if required
   - Validates currency matches product currency
   - Validates moratorium value if requested

7. **Entity Update Logic**:

   - Different handling based on entity type (loan request vs. loan)
   - Different handling based on application stage (approved vs. not approved)
   - Updates metadata (last modified info)
   - Sets application stage to IN_REVIEW if not already approved
   - For loan requests, updates interest rate information

8. **Change Tracking**:

   - Logs the fields that have changed between the original and updated entity
   - Stores the changed fields in the entity

9. **Persistence**: Saves the updated entity
10. **Related Entity Management**:

    - Updates charges if provided
    - Removes old collaterals and adds new ones if provided
    - Updates additional facility details if provided
    - For corporate loans, handles tranches (removing old ones and adding new ones)

11. **Auditing and Logging**:

    - Logs loan activities
    - Records audit information for successful operations

12. **Exception Handling**: Catches exceptions, logs audit information for failures, and rethrows the exception

## Detailed Flow Diagram

```mermaid

graph TD
    A["Client Request"] -->|"PUT /{loanId}"| B["Controller"]
    B -->|"Validate Request"| C{"Has Validation Errors?"}
    C -->|"Yes"| D["Return Bad Request"]
    C -->|"No"| E{"Loan Type?"}

    E -->|"CORPORATE"| F["Corporate Loan Service"]
    E -->|"INDIVIDUAL"| G["Individual Loan Service"]

    F & G --> H["Get User Profile"]
    H --> I["Retrieve Product Details"]
    I --> J["Check Obligor Limits"]

    J --> K["Find Existing Loan/LoanRequest"]
    K -->|"Not Found"| L["Throw NotFoundException"]
    K -->|"Found"| M["Copy Non-null Properties"]

    M --> N["Capture Original State"]

    G -->|"Individual Service"| O{"Update Portfolio?"}
    O -->|"Yes"| P["Update Consumed Portfolio"]
    O -->|"No"| Q["Validate Principal Amount"]
    P --> Q

    F -->|"Corporate Service"| Q

    Q -->|"Invalid"| R["Throw BadRequestException"]
    Q -->|"Valid"| S["Validate Loan Tenor"]

    S -->|"Invalid"| T["Throw BadRequestException"]
    S -->|"Valid"| U["Validate Disbursement Date"]

    U -->|"Invalid"| V["Throw BadRequestException"]
    U -->|"Valid"| W["Validate Currency"]

    W -->|"Invalid"| X["Throw BadRequestException"]
    W -->|"Valid"| Y["Validate Moratorium"]

    Y -->|"Invalid"| Z["Throw BadRequestException"]
    Y -->|"Valid"| AA["Get Customer Notification Emails"]

    AA --> AB{"Entity Type?"}

    AB -->|"LoanRequest"| AC{"Application Stage?"}
    AB -->|"Loan"| AD{"Application Stage?"}

    AC -->|"Not APPROVED"| AE["Update Metadata"]
    AC -->|"APPROVED"| AF["Update Loan Request"]

    AD -->|"Not APPROVED"| AG["Update Metadata"]
    AD -->|"APPROVED"| AH["Update Loan"]

    AE & AF & AG & AH --> AI["Log Changed Fields"]

    AI --> AJ["Save Entity"]

    AJ --> AK{"Has Charges?"}
    AK -->|"Yes"| AL["Update Charges"]
    AK -->|"No"| AM{"Has Collaterals?"}

    AL --> AM
    AM -->|"Yes"| AN["Remove Old Collaterals"]
    AN --> AO["Add New Collaterals"]
    AM -->|"No"| AP{"Has Additional Facility?"}

    AO --> AP
    AP -->|"Yes"| AQ["Update Additional Facility"]
    AP -->|"No"| AR["Log Loan Activities"]

    AQ --> AR

    F -->|"Corporate Service"| AS{"Has Tranches?"}
    AS -->|"Yes"| AT["Update Tranches"]
    AS -->|"No"| AR

    AT --> AR

    AR --> AU["Audit Success"]
    AU --> AV["Return Response"]

    F & G -.->|"Exception"| AW["Audit Failure"]
    AW --> AX["Throw Exception"]
```
