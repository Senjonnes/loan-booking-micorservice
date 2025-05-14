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
