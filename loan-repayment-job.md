# EOD Loan Repayment Processing Endpoint

This system handles End-of-Day (EOD) processing for loan repayments through several key components

1. **API Endpoint**: The process starts with a POST request to `/run` with EOD trigger data.
2. **EOD Service**: Filters the triggers to find loan repayment activities and initiates processing.
3. **Repayment Processing**: The core process involves:

   - Creating a tracking record for the EOD run
   - Retrieving all due loan repayment schedules
   - Processing each repayment in parallel using a ForkJoinPool

4. **Individual Repayment Processing**:

   - Checks if the repayment is already paid
   - Retrieves account details and balance
   - Removes any existing liens
   - Determines account funding state (fully funded, partially funded, not funded)
   - Takes appropriate action based on funding state:

     - For fully funded accounts: Prepares journal posting details
     - For partially funded: Attempts to source funds from other accounts
     - For unfunded accounts: Places a lien or tries other funding sources

- Updates processing counts

6. **Scheduled Jobs**: Several background jobs run at regular intervals to:

   - Process batch journal postings (every 5 minutes)
   - Check status of batch postings (every 15 minutes)
   - Process single postings (every 15 minutes)
   - Update repayment schedules (every 6 minutes)

## Detailed Flow Diagram

```mermaid

graph TD
    A[Client] -->|"POST /run"| B[EOD Controller]
    B -->|"Count unpaid schedules"| C[Loan Repayment Repository]
    C -->|"Return count"| B
    B -->|"runEODX"| D[EOD Service]

    D -->|"Filter for loan repayment trigger"| E[Process EOD Repayments]
    E -->|"Create tracker"| F[Create and lock tracker]
    F -->|"Get due schedules"| G[Get due payment schedules]

    G -->|"Process in parallel"| H[ForkJoinPool]
    H -->|"For each schedule"| I[Process individual repayment]

    I -->|"Get account details"| J[Account Service]
    I -->|"Remove lien if exists"| K[Lien Service]
    I -->|"Process repayment"| L[Process Repayment]

    L -->|"Check payment status"| M{Already paid?}
    M -->|"Yes"| N[Skip processing]
    M -->|"No"| O[Check account balance]

    O -->|"Determine account state"| P{Account state?}
    P -->|"FULLY_FUNDED"| Q[Prepare journal posting]
    P -->|"PARTIALLY_FUNDED"| R[Try other accounts]
    P -->|"NOT_FUNDED"| S[Place lien or try other accounts]
    P -->|"NOT_OWING"| T[Skip processing]

    R -->|"Update balance"| Q
    S -->|"If other sources available"| R
    S -->|"No other sources"| U[Place lien]

    Q -->|"Return journal model"| I
    N -->|"Return detail process"| I
    T -->|"Return detail process"| I
    U -->|"Return detail process"| I

    I -->|"Update counts"| V[Update tracker]
    V -->|"Complete EOD process"| W[Return response]

    subgraph "Scheduled Jobs"
        X[Batch Posting Job]
        Y[Status Check Job]
        Z[Single Posting Job]
        AA[Update Schedule Job]
    end

    X -.->|"Every 5 minutes"| BB[Batch Process Service]
    Y -.->|"Every 15 minutes"| CC[Status Check Service]
    Z -.->|"Every 15 minutes"| DD[Single Process Service]
    AA -.->|"Every 6 minutes"| EE[Repayment Post Service]
```
