# Loan Booking Endpoint

The loan creation API follows these key steps:

### Request Validation:

1. The API first validates the incoming request body
1. If validation errors are found, it returns a 400 Bad Request response

### Loan Type Determination:

1. The system checks if the loan is a corporate or individual loan
1. Based on the loan type, it routes to the appropriate service

### Corporate Loan Processing :

1. Validates user profile and KYC information
1. For loan contracts, checks credit line details and limits
1. Retrieves product details and validates portfolio limits
1. Validates obligor limits, disbursement tranches, and notification details
1. Performs principal amount and loan tenor validation against product parameters
1. Creates and saves the loan request model with all necessary parameters
1. Processes tranches, charges, collaterals, and additional facility details
1. Generates notifications for reviewers/approvers

### Individual Loan Processing :

1. Similar to corporate loans but with some differences
1. Validates user profile, KYC, product details, and obligor limits
1. Validates disbursement date and portfolio limits
1. Handles placement loan types differently
1. Creates and saves the loan request model
1. Processes charges, collaterals, and additional facility details
1. Generates notifications

### Response Handling :

1. Both paths return a success response with loan details if successful
1. Various validation errors can occur at different stages, resulting in Bad Request responses

## Detailed Flow Diagram

```mermaid
flowchart TD
    A["POST /loans API Endpoint"] --> B["Validate Request Body"]
    B -->|"Validation Errors"| C["Return 400 Bad Request"]
    B -->|"Valid Request"| D["Check Loan Type"]
    D -->|"Corporate Loan"| E["Corporate Loan Service"]
    D -->|"Individual Loan"| F["Individual Loan Service"]

    %% Corporate Loan Flow
    E --> E1["Validate User Profile"]
    E1 --> E2["Check Credit Line (if Loan Contract)"]
    E2 --> E3["Validate KYC Profile"]
    E3 --> E4["Retrieve Product Details"]
    E4 --> E5["Validate Portfolio Limits"]
    E5 --> E6["Validate Obligor Limit"]
    E6 --> E7["Validate Disbursement Tranches"]
    E7 --> E8["Get Customer Notification Emails"]
    E8 --> E9["Validate Principal Amount"]
    E9 --> E10["Validate Loan Tenor"]
    E10 --> E11["Create Loan Request Model"]
    E11 --> E12["Set Loan Parameters"]
    E12 --> E13["Validate Disbursement Details"]
    E13 --> E14["Validate Currency"]
    E14 --> E15["Save Loan Request"]
    E15 --> E16["Process Tranches"]
    E16 --> E17["Process Charges"]
    E17 --> E18["Process Collaterals"]
    E18 --> E19["Process Additional Facility Details"]
    E19 --> E20["Generate Notifications"]
    E20 --> Z["Return Success Response"]

    %% Individual Loan Flow
    F --> F1["Validate User Profile"]
    F1 --> F2["Validate KYC Profile"]
    F2 --> F3["Retrieve Product Details"]
    F3 --> F4["Validate Obligor Limit"]
    F4 --> F5["Validate Portfolio Limits"]
    F5 --> F6["Validate Disbursement Date"]
    F6 --> F7["Get Customer Notification Emails"]
    F7 --> F8["Validate Principal Amount"]
    F8 --> F9["Validate Loan Tenor"]
    F9 --> F10["Create Loan Request Model"]
    F10 --> F11["Set Loan Parameters"]
    F11 --> F12["Handle Placement Loan Type"]
    F12 --> F13["Validate Disbursement Details"]
    F13 --> F14["Validate Currency"]
    F14 --> F15["Save Loan Request"]
    F15 --> F16["Process Charges"]
    F16 --> F17["Process Collaterals"]
    F17 --> F18["Process Additional Facility Details"]
    F18 --> F19["Generate Notifications"]
    F19 --> Z

    %% Error paths
    E9 -->|"Invalid Amount"| E9E["Throw Bad Request Exception"]
    E10 -->|"Invalid Tenor"| E10E["Throw Bad Request Exception"]
    E13 -->|"Invalid Details"| E13E["Throw Bad Request Exception"]
    E14 -->|"Currency Mismatch"| E14E["Throw Bad Request Exception"]

    F8 -->|"Invalid Amount"| F8E["Throw Bad Request Exception"]
    F9 -->|"Invalid Tenor"| F9E["Throw Bad Request Exception"]
    F13 -->|"Invalid Details"| F13E["Throw Bad Request Exception"]
    F14 -->|"Currency Mismatch"| F14E["Throw Bad Request Exception"]

    E9E --> C
    E10E --> C
    E13E --> C
    E14E --> C
    F8E --> C
    F9E --> C
    F13E --> C
    F14E --> C
```



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