# Granular Explanation of the Loan Creation Process Endpoint

The loan creation API follows these key steps:

### 1. API Endpoint (Controller)

- The process begins with a POST request to the loan creation endpoint
- The endpoint is annotated with Swagger documentation (@Operation, @ApiResponses)
- It consumes and produces JSON data (APPLICATION_JSON_VALUE)
- The endpoint accepts three parameters:

  - RequestBodyWrapperDto (request body with loan details)
  - BindingResult (validation results)
  - LoanTypesEnum (optional query parameter for loan type)

### 2. Request Validation

- The controller first checks for validation errors in the request body
- If validation errors exist, it creates a ResponseStatusDto with error details
- It returns a 400 Bad Request response with the error details

### 3. Loan Type Determination

- The controller checks if the loan is a corporate loan using the isCorporateLoan method
- Based on the loan type, it routes to either:

  - corporateLoanService.createLoanRequest (for corporate loans)
  - loanService.createLoan (for individual loans)

### 4. Corporate Loan Processing

#### 4.1 Initial Setup

- Initializes JavaTimeModule for date handling
- Retrieves the current user profile
- Gets the authentication token
- Stores the token in a map for later use

### 4.2 Credit Line Validation (for Loan Contracts)

- Checks if the loan is a loan contract
- If yes, validates that the credit line ID is not null
- Retrieves the global credit line from the database
- Calculates the sum of existing loan contracts for the credit line
- Checks if adding the new loan would exceed the credit limit
- Calculates and compares tenor durations to ensure compliance

### 4.3 Customer and Product Validation

- Validates the customer's KYC profile
- Retrieves product details using the product ID
- Validates portfolio limits to ensure the loan doesn't exceed portfolio constraints
- Retrieves the global obligor limit
- Checks if the customer is within the allowed obligor limit

### 4.4 Disbursement Validation

- Validates disbursement tranches if tranched disbursement is enabled
- For adhoc tranches, checks percentages and dates
- For scheduled tranches, validates tenor periods and disbursement dates
- Gets customer notification emails for later use

#### 4.5 Principal and Tenor Validation

- Checks if the loan principal amount is within the product's min/max limits
- Checks if the loan tenor is within the product's min/max limits
- If either check fails, throws a BadRequestException

#### 4.6 Loan Request Model Creation

- Creates a new LoanRequestModel from the request data
- Sets numerous properties including:

  - Tenor period and tranched disbursement settings
  - Loan status (PENDING_APPROVAL)
  - Various flags (isLoanDraft, isLoanRequest, etc.)
  - Product and loan type details
  - Interest rate information
  - Application stage (INITIATED)

#### 4.7 Fee Configuration

- Checks if charges were provided in the request
- If not, sets applyDefaultProductFees to true to use product-defined fees
- Validates disbursement details including required dates
- Validates that the currency matches the product currency

#### 4.8 Save and Process

- Saves the loan request to the database
- Formulates and persists tranches if applicable
- Processes charges if provided
- Prepares the loan response
- Processes collaterals if provided
- Processes additional facility details if provided

#### 4.9 Notifications

- Generates in-app notification events
- Sends email notifications to reviewers/approvers
- Returns a success response with the loan details

### 5. Individual Loan Processing

#### 5.1 Initial Setup

- Similar to corporate loans: initializes JavaTimeModule, gets user profile and token

#### 5.2 Validations

- Validates the customer's KYC profile
- Retrieves product details
- Validates obligor limit
- Validates portfolio limits
- Validates disbursement date

#### 5.3 Principal and Tenor Validation

- Checks if the loan principal amount is within the product's min/max limits
- Checks if the loan tenor is within the product's min/max limits
- If either check fails, throws a BadRequestException

#### 5.4 Loan Request Model Creation

- Creates a new LoanRequestModel from the request data
- Sets numerous properties similar to corporate loans
- Disables tranched disbursement (not available for individual loans)

#### 5.5 Special Handling

- Checks if the loan is a placement loan
- If yes, extracts account number from product and sets as loan account number
- Updates gatekeeper fields
- Sets up audit meta information

#### 5.6 Save and Process

- Validates disbursement details and currency
- Saves the loan request to the database
- Processes charges, collaterals, and additional facility details
- Generates notifications
- Returns a success response

### 6. Error Handling

- Both flows include extensive error handling
- Specific exceptions are thrown for different validation failures:

  - BadRequestException for invalid inputs
  - NotFoundException for missing resources
  - InsufficientFundsException for limit violations
  - InternalErrorException for system errors

- All exceptions eventually result in appropriate HTTP responses

## Detailed Flow Diagram

```mermaid

flowchart TD
    A["POST /loans API Endpoint"] --> B["Validate Request Body"]
    B -->|"Validation Errors"| C["Return 400 Bad Request with Error Details"]
    B -->|"Valid Request"| D["Check Loan Type"]
    D -->|"Corporate Loan"| E["Corporate Loan Service"]
    D -->|"Individual Loan"| F["Individual Loan Service"]

    %% Corporate Loan Flow - Initial Setup
    E --> E1["Initialize JavaTimeModule"]
    E1 --> E2["Get User Profile"]
    E2 --> E3["Get Authentication Token"]
    E3 --> E4["Store Token in Map"]

    %% Corporate Loan Flow - Credit Line Validation
    E4 --> E5["Check if Loan Contract"]
    E5 -->|"Yes"| E6["Validate Credit Line ID"]
    E6 --> E7["Retrieve Global Credit Line"]
    E7 --> E8["Calculate Sum of Existing Contracts"]
    E8 --> E9["Check if Credit Limit Exceeded"]
    E9 --> E10["Calculate Tenor Durations"]
    E10 --> E11["Compare Tenor Durations"]
    E11 --> E12["Validate Customer KYC Profile"]
    E5 -->|"No"| E12

    %% Corporate Loan Flow - Product Validation
    E12 --> E13["Retrieve Product Details"]
    E13 --> E14["Validate Portfolio Limits"]
    E14 --> E15["Retrieve Global Obligor Limit"]
    E15 --> E16["Check Customer Within Obligor Limit"]

    %% Corporate Loan Flow - Disbursement Validation
    E16 --> E17["Validate Disbursement Tranches"]
    E17 --> E18["Get Customer Notification Emails"]

    %% Corporate Loan Flow - Principal and Tenor Validation
    E18 --> E19["Check Principal Amount Within Limits"]
    E19 -->|"No"| E19E["Throw Bad Request Exception"]
    E19 -->|"Yes"| E20["Check Loan Tenor Within Limits"]
    E20 -->|"No"| E20E["Throw Bad Request Exception"]

    %% Corporate Loan Flow - Loan Request Model Creation
    E20 -->|"Yes"| E21["Create Loan Request Model"]
    E21 --> E22["Set Tranches to null"]
    E22 --> E23["Set Tenor Period"]
    E23 --> E24["Set Tranched Disbursement Flag"]
    E24 --> E25["Set Super User Flag"]
    E25 --> E26["Set Loan Account to null"]
    E26 --> E27["Set Is Loan Draft to false"]
    E27 --> E28["Set Notification Emails"]
    E28 --> E29["Set Status to PENDING_APPROVAL"]
    E29 --> E30["Set Loan Request Flags"]
    E30 --> E31["Set Credit Line ID"]
    E31 --> E32["Set Product Details"]
    E32 --> E33["Set Loan Type Details"]
    E33 --> E34["Set Loan ID to null"]
    E34 --> E35["Set Loan Request Type"]
    E35 --> E36["Set Credit Line Flags"]
    E36 --> E37["Set Interest Rate Details"]
    E37 --> E38["Set Application Stage"]
    E38 --> E39["Set Disbursement Account Source"]
    E39 --> E40["Update Gatekeeper Fields"]

    %% Corporate Loan Flow - Fee Configuration
    E40 --> E41["Check if Charges Provided"]
    E41 -->|"No"| E42["Set Apply Default Product Fees to true"]
    E41 -->|"Yes"| E43["Continue with Provided Charges"]
    E42 --> E44["Validate Disbursement Details"]
    E43 --> E44

    %% Corporate Loan Flow - Final Validations
    E44 --> E45["Check if Disbursement Date Required"]
    E45 --> E46["Validate Currency Match"]
    E46 --> E47["Check Moratorium Request"]
    E47 --> E48["Set Request Status to ACTIVE"]

    %% Corporate Loan Flow - Save and Process
    E48 --> E49["Save Loan Request to Database"]
    E49 --> E50["Formulate and Persist Tranches"]
    E50 --> E51["Process Charges if Provided"]
    E51 --> E52["Prepare Loan Response"]
    E52 --> E53["Process Collaterals if Provided"]
    E53 --> E54["Process Additional Facility Details if Provided"]

    %% Corporate Loan Flow - Notifications
    E54 --> E55["Generate In-App Notification"]
    E55 --> E56["Send Email Notifications"]
    E56 --> Z["Return Success Response"]

    %% Individual Loan Flow - Initial Setup
    F --> F1["Initialize JavaTimeModule"]
    F1 --> F2["Get User Profile"]
    F2 --> F3["Get Authentication Token"]
    F3 --> F4["Store Token in Map"]

    %% Individual Loan Flow - Validations
    F4 --> F5["Validate Customer KYC Profile"]
    F5 --> F6["Retrieve Product Details"]
    F6 --> F7["Validate Obligor Limit"]
    F7 --> F8["Validate Portfolio Limits"]

    %% Individual Loan Flow - Disbursement and Notification
    F8 --> F9["Validate Disbursement Date"]
    F9 --> F10["Get Customer Notification Emails"]

    %% Individual Loan Flow - Principal and Tenor Validation
    F10 --> F11["Check Principal Amount Within Limits"]
    F11 -->|"No"| F11E["Throw Bad Request Exception"]
    F11 -->|"Yes"| F12["Check Loan Tenor Within Limits"]
    F12 -->|"No"| F12E["Throw Bad Request Exception"]

    %% Individual Loan Flow - Loan Request Model Creation
    F12 -->|"Yes"| F13["Create Loan Request Model"]
    F13 --> F14["Set Tenor Period"]
    F14 --> F15["Set Loan Account to null"]
    F15 --> F16["Set Is Loan Draft to false"]
    F16 --> F17["Set Status to PENDING_APPROVAL"]
    F17 --> F18["Set Loan Request Flags"]
    F18 --> F19["Set Product Details"]
    F19 --> F20["Set Loan Type Details"]
    F20 --> F21["Disable Tranched Disbursement"]
    F21 --> F22["Set Loan ID to null"]
    F22 --> F23["Set Notification Emails"]
    F23 --> F24["Set Interest Rate Details"]

    %% Individual Loan Flow - Fee Configuration
    F24 --> F25["Check if Charges Provided"]
    F25 -->|"No"| F26["Set Apply Default Product Fees to true"]
    F25 -->|"Yes"| F27["Continue with Provided Charges"]
    F26 --> F28["Set Application Stage"]
    F27 --> F28

    %% Individual Loan Flow - Special Handling
    F28 --> F29["Set Disbursement Account Source"]
    F29 --> F30["Check if Placement Loan"]
    F30 -->|"Yes"| F31["Insert Loan Account for Placement Type"]
    F30 -->|"No"| F32["Update Gatekeeper Fields"]
    F31 --> F32

    %% Individual Loan Flow - Audit and Validation
    F32 --> F33["Setup Audit Meta Info"]
    F33 --> F34["Validate Disbursement Details"]
    F34 --> F35["Validate Currency"]
    F35 --> F36["Check Moratorium Request"]
    F36 --> F37["Set Request Status to ACTIVE"]

    %% Individual Loan Flow - Save and Process
    F37 --> F38["Save Loan Request to Database"]
    F38 --> F39["Process Charges if Provided"]
    F39 --> F40["Prepare Loan Response"]
    F40 --> F41["Process Collaterals if Provided"]
    F41 --> F42["Process Additional Facility Details if Provided"]

    %% Individual Loan Flow - Notifications
    F42 --> F43["Generate In-App Notification"]
    F43 --> F44["Send Email Notifications"]
    F44 --> Z

    %% Error paths
    E19E --> C
    E20E --> C
    F11E --> C
    F12E --> C

    %% Exception handling
    E6 -->|"Credit Line ID null"| E6E["Throw Bad Request Exception"]
    E7 -->|"Credit Line Not Found"| E7E["Throw Not Found Exception"]
    E9 -->|"Credit Limit Exceeded"| E9E["Throw Bad Request Exception"]
    E11 -->|"Tenor Too Long"| E11E["Throw Bad Request Exception"]
    E13 -->|"Product Not Found"| E13E["Throw Not Found Exception"]
    E14 -->|"Portfolio Limit Exceeded"| E14E["Throw Bad Request Exception"]
    E16 -->|"Obligor Limit Exceeded"| E16E["Throw Insufficient Funds Exception"]
    E17 -->|"Invalid Tranches"| E17E["Throw Bad Request Exception"]
    E45 -->|"Missing Disbursement Date"| E45E["Throw Bad Request Exception"]
    E46 -->|"Currency Mismatch"| E46E["Throw Bad Request Exception"]
    E49 -->|"Database Error"| E49E["Throw Internal Error Exception"]

    F6 -->|"Product Not Found"| F6E["Throw Not Found Exception"]
    F7 -->|"Obligor Limit Exceeded"| F7E["Throw Insufficient Funds Exception"]
    F8 -->|"Portfolio Limit Exceeded"| F8E["Throw Bad Request Exception"]
    F34 -->|"Invalid Details"| F34E["Throw Bad Request Exception"]
    F35 -->|"Currency Mismatch"| F35E["Throw Bad Request Exception"]
    F38 -->|"Database Error"| F38E["Throw Internal Error Exception"]

    E6E --> C
    E7E --> C
    E9E --> C
    E11E --> C
    E13E --> C
    E14E --> C
    E16E --> C
    E17E --> C
    E45E --> C
    E46E --> C
    E49E --> C
    F6E --> C
    F7E --> C
    F8E --> C
    F34E --> C
    F35E --> C
    F38E --> C
```
