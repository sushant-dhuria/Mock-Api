# authorizeCharge API Documentation

## Overview

The `authorizeCharge` endpoint is a REST API that handles payment authorization requests in the Santander GIL GPP (Global Installment Lending - Guaranteed Payment Program) system. This asynchronous endpoint processes authorization tasks for installment payment entities and delegates the actual processing to background handlers.

## API Specification

### Endpoint Details
- **Path**: `/authorize` (defined by [`ServiceConstants.AUTHORIZE`](../../src/com/amazon/santandergilgpparestplugin/constants/ServiceConstants.java#L10))
- **Method**: POST
- **Controller**: [`ChargeController`](../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L63)
- **Content-Type**: `application/x-amzn-ion` (Ion Media Type)
- **Authentication**: AAA (Amazon Authorization Architecture) required
- **Service Name**: `SantanderGILGPPArestPlugin`
- **Operation Name**: `paymentProcessing`

### Request Format
The endpoint accepts an [`AuthorizeTask`](../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L63) object in Ion format containing:
- Payment line items with order IDs
- Amount to authorize
- Processor idempotency key
- Charge ID
- Reference data for the authorization

### Response Format
- **Success Response**: HTTP 202 (Accepted) with Ion content type
- **Processing**: Asynchronous - request is queued for background processing
- **Response Body**: Empty (async processing initiated)

## Implementation Flow

### Main Authorization Flow

```mermaid
flowchart TD
    Start[authorizeCharge Request] --> LogRequest[Log Incoming Request]
    LogRequest --> CreateTransformer[Create PSXTProcessorRequestTransformer]
    CreateTransformer --> TransformRequest[Transform AuthorizeTask to AuthorizeRequest]
    TransformRequest --> CheckPurchaseId{Purchase ID Present?}
    CheckPurchaseId -->|No| CreateDummyLoan[Create Dummy Loan for Decline]
    CheckPurchaseId -->|Yes| GetLoan[Get Latest Loan from Database]
    CreateDummyLoan --> SetLoan[Set Loan on Request]
    GetLoan --> SetLoan
    SetLoan --> AsyncHandle[Submit to Async Handler]
    AsyncHandle --> LogCompletion[Log Async Invocation]
    LogCompletion --> ReturnResponse[Return HTTP 202 Accepted]

    click Start "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L63" "authorizeCharge Method" _blank
    click LogRequest "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L64" "Request Logging" _blank
    click CreateTransformer "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L66" "Transformer Creation" _blank
    click TransformRequest "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L71" "Request Transformation" _blank
    click CheckPurchaseId "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L76" "Purchase ID Check" _blank
    click CreateDummyLoan "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L78" "Dummy Loan Creation" _blank
    click GetLoan "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L80" "Database Loan Retrieval" _blank
    click SetLoan "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L83" "Set Loan on Request" _blank
    click AsyncHandle "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L86" "Async Handler Invocation" _blank
    click LogCompletion "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L88" "Completion Logging" _blank
    click ReturnResponse "../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L90" "HTTP Response Creation" _blank
```

### Request Transformation Flow

```mermaid
flowchart TD
    TransformStart[PSXTProcessorRequestTransformer.transform] --> ExtractOrderId[Extract Order ID from Payment Line Items]
    ExtractOrderId --> GetPurchaseDetails[Get Purchase Details from Database]
    GetPurchaseDetails --> CreateAuthRequest[Create AuthorizeRequest Object]
    CreateAuthRequest --> SetRequestFields[Set Request Fields from Task]
    SetRequestFields --> ConvertCurrency[Convert PayStation Currency to Local Currency]
    ConvertCurrency --> SetPurchaseData[Set Purchase Reference Data]
    SetPurchaseData --> ReturnAuthRequest[Return AuthorizeRequest]

    click TransformStart "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L45" "Transform Method" _blank
    click ExtractOrderId "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L91" "Order ID Extraction" _blank
    click GetPurchaseDetails "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L76" "Purchase Details Retrieval" _blank
    click CreateAuthRequest "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L118" "AuthorizeRequest Creation" _blank
    click SetRequestFields "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L121" "Field Setting" _blank
    click ConvertCurrency "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L124" "Currency Conversion" _blank
    click SetPurchaseData "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L128" "Purchase Data Setting" _blank
    click ReturnAuthRequest "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L133" "Return Statement" _blank
```

### Loan Retrieval Flow

```mermaid
flowchart TD
    LoanCheck{Purchase ID Available?} --> GetLatestLoan[InstallmentPaymentEntityDao.getLatestLoanForPurchaseId]
    LoanCheck --> CreateDummy[LoanUtils.createDummyLoanForDecline]
    GetLatestLoan --> QueryDatabase[Query DynamoDB by Purchase ID Index]
    QueryDatabase --> SortResults[Sort Results by Version]
    SortResults --> ReturnLatest[Return Latest Version]
    CreateDummy --> SetDeclineFlag[Set shouldDeclineAuth Flag]
    SetDeclineFlag --> CreateDummyEntity[Create InstallmentPaymentEntity with Random UUID]
    CreateDummyEntity --> SetCountryCode[Set Country Code to ES]

    click GetLatestLoan "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L117" "getLatestLoanForPurchaseId Method" _blank
    click QueryDatabase "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L264" "Database Query Method" _blank
    click SortResults "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L280" "Results Sorting" _blank
    click ReturnLatest "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L120" "Latest Loan Return" _blank
    click CreateDummy "../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java#L84" "createDummyLoanForDecline Method" _blank
    click SetDeclineFlag "../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java#L85" "Decline Flag Setting" _blank
    click CreateDummyEntity "../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java#L89" "Dummy Entity Creation" _blank
    click SetCountryCode "../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java#L92" "Country Code Setting" _blank
```

### Asynchronous Processing Flow

```mermaid
flowchart TD
    AsyncStart[AsyncRequestHandler.handle] --> ThreadPool{Handler Type}
    ThreadPool -->|ThreadPoolAsyncRequestHandler| SetDateTime[Set Request Start DateTime]
    ThreadPool -->|HerdAsyncRequestHandler| CreateHerdRequest[Create Herd Work Item]
    SetDateTime --> GetTaskBuilder[Get Task Builder from Loan Manager]
    GetTaskBuilder --> BuildTask[Build AsyncTask from Operation]
    BuildTask --> ExecuteTask[Submit to Thread Pool Executor]
    CreateHerdRequest --> SubmitToHerd[Submit Work Item to Orca/Herd]

    click AsyncStart "../../src/com/amazon/santandergilgpparestplugin/async/handler/AsyncRequestHandler.java#L12" "AsyncRequestHandler Interface" _blank
    click SetDateTime "../../src/com/amazon/santandergilgpparestplugin/async/handler/ThreadPoolAsyncRequestHandler.java#L37" "DateTime Setting" _blank
    click GetTaskBuilder "../../src/com/amazon/santandergilgpparestplugin/async/handler/ThreadPoolAsyncRequestHandler.java#L38" "Task Builder Retrieval" _blank
    click BuildTask "../../src/com/amazon/santandergilgpparestplugin/async/handler/ThreadPoolAsyncRequestHandler.java#L42" "Task Building" _blank
    click ExecuteTask "../../src/com/amazon/santandergilgpparestplugin/async/handler/ThreadPoolAsyncRequestHandler.java#L46" "Task Execution" _blank
    click CreateHerdRequest "../../src/com/amazon/santandergilgpparestplugin/async/handler/HerdAsyncRequestHandler.java#L36" "Herd Request Creation" _blank
    click SubmitToHerd "../../src/com/amazon/santandergilgpparestplugin/async/handler/HerdAsyncRequestHandler.java#L36" "Herd Submission" _blank
```

## Detailed Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant ChargeController
    participant PSXTTransformer as PSXTProcessorRequestTransformer
    participant PurchaseReferenceDao
    participant IPEDao as InstallmentPaymentEntityDao
    participant LoanUtils
    participant AsyncHandler as AsyncRequestHandler
    participant ThreadPool as ThreadPoolExecutor

    Client->>ChargeController: POST /authorize (AuthorizeTask)
    activate ChargeController

    ChargeController->>ChargeController: Log incoming request
    ChargeController->>PSXTTransformer: new PSXTProcessorRequestTransformer
    ChargeController->>PSXTTransformer: transform(authorizeTask)
    activate PSXTTransformer

    PSXTTransformer->>PSXTTransformer: extractOrderId from payment line items
    PSXTTransformer->>PurchaseReferenceDao: getPurchaseDetailsForOrderId
    PurchaseReferenceDao-->>PSXTTransformer: PurchaseReference
    PSXTTransformer->>PSXTTransformer: createAuthorizeRequest
    PSXTTransformer->>PSXTTransformer: convertPaystationToLocalCurrency
    PSXTTransformer-->>ChargeController: AuthorizeRequest
    deactivate PSXTTransformer

    ChargeController->>ChargeController: Check if purchaseId is null
    alt Purchase ID is null
        ChargeController->>LoanUtils: createDummyLoanForDecline
        LoanUtils-->>ChargeController: dummy InstallmentPaymentEntity
    else Purchase ID exists
        ChargeController->>IPEDao: getLatestLoanForPurchaseId
        IPEDao-->>ChargeController: InstallmentPaymentEntity
    end

    ChargeController->>ChargeController: Set loan on AuthorizeRequest
    ChargeController->>AsyncHandler: handle(authorizeRequest)
    activate AsyncHandler

    AsyncHandler->>ThreadPool: execute(PaymentProcessingTask)
    AsyncHandler-->>ChargeController: void
    deactivate AsyncHandler

    ChargeController->>ChargeController: Log async invocation
    ChargeController-->>Client: HTTP 202 Accepted
    deactivate ChargeController
```

## Error Handling

### Exception Scenarios

```mermaid
flowchart TD
    ErrorStart[Error Scenarios] --> TransformError{Transformation Errors}
    ErrorStart --> DatabaseError{Database Errors}
    ErrorStart --> ValidationError{Validation Errors}

    TransformError -->|Currency Conversion| RedJackTokenError[RedJackTokenProcessingException]
    TransformError -->|Unsupported Task| UnsupportedOpError[UnsupportedOperationException]

    DatabaseError -->|Optimistic Locking| ConcurrencyError[OptimisticConcurrencyException]
    DatabaseError -->|Resource Not Found| NotFoundError[ResourceNotFoundException]
    DatabaseError -->|General DB Error| DependencyError[DependencyException]

    ValidationError -->|Null Purchase ID| CreateDummyFlow[Create Dummy Loan Flow]

    click RedJackTokenError "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L241" "RedJack Token Exception" _blank
    click UnsupportedOpError "../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java#L68" "Unsupported Operation Exception" _blank
    click ConcurrencyError "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L81" "Optimistic Concurrency Exception" _blank
    click NotFoundError "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L337" "Resource Not Found Exception" _blank
    click DependencyError "../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L330" "Dependency Exception" _blank
    click CreateDummyFlow "../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java#L84" "Dummy Loan Creation" _blank
```

### Error Response Handling
Since this is an asynchronous endpoint, errors during the initial request processing phase will result in standard HTTP error responses. However, errors during background processing are handled within the async task execution and do not affect the initial HTTP response.

## Data Models

### Key Data Structures

#### AuthorizeRequest
- **Location**: [`AuthorizeRequest.java`](../../src/com/amazon/santandergilgpparestplugin/async/requests/AuthorizeRequest.java)
- **Extends**: [`PaymentRequest`](../../src/com/amazon/santandergilgpparestplugin/async/requests/PaymentRequest.java)
- **Fields**:
  - `shouldDeclineAuth`: Boolean flag indicating if authorization should be declined
  - Inherited from PaymentRequest: requestId, paymentProcessingTaskId, originalTransactionId, requestedAmount, purchaseId, paymentContractId, orderId

#### InstallmentPaymentEntity
- **Location**: [`InstallmentPaymentEntity.java`](../../src/com/amazon/santandergilgpparestplugin/model/payment/InstallmentPaymentEntity.java)
- **Storage**: DynamoDB table with versioning support
- **Key Fields**: id, version, applicationId, purchaseId, paymentContractId, customerId

#### PurchaseReference
- **Location**: [`PurchaseReference.java`](../../src/com/amazon/santandergilgpparestplugin/model/purchase/PurchaseReference.java)
- **Storage**: DynamoDB table
- **Key Fields**: orderId, purchaseId, paymentContractId, customerId

## Configuration and Dependencies

### Dependency Injection
The [`ChargeController`](../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java#L43) uses constructor injection for:
- [`AsyncRequestHandler`](../../src/com/amazon/santandergilgpparestplugin/async/handler/AsyncRequestHandler.java) (interface - actual implementation injected)
- [`InstallmentPaymentEntityDao`](../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java)
- [`PurchaseReferenceDao`](../../src/com/amazon/santandergilgpparestplugin/accessor/dao/PurchaseReferenceDao.java)
- [`LoanUtils`](../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java)
- [`PluginCustomFieldUtils`](../../src/com/amazon/santandergilgpparestplugin/utils/PluginCustomFieldUtils.java)

### Service Constants
- **Service Name**: `SantanderGILGPPArestPlugin`
- **Authorization Path**: `/authorize`
- **Operation Name**: `paymentProcessing`

## Database Operations

### DynamoDB Tables Accessed
1. **InstallmentPaymentEntity Table**
   - Query by purchaseId using GSI (Global Secondary Index)
   - Retrieve latest version by sorting on version attribute

2. **PurchaseReference Table**
   - Load by orderId (primary key)
   - Contains payment contract and purchase mapping data

### Database Query Patterns
- **Purchase ID Lookup**: Uses [`PAYMENT_ENTITY_PURCHASE_ID_INDEX`](../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java#L270) GSI
- **Version Sorting**: Results sorted by version to get latest record
- **Consistent Read**: Disabled for performance (eventual consistency acceptable)

## Integration Points

### External Services
- **PayStation**: Source of AuthorizeTask requests
- **GIL (Global Installment Lending)**: Target system for payment processing
- **DynamoDB**: Data persistence layer
- **Herd/Orca**: Workflow orchestration system (when using HerdAsyncRequestHandler)

### Async Processing Options
1. **ThreadPoolAsyncRequestHandler**: Uses local thread pool executor
2. **HerdAsyncRequestHandler**: Submits work items to Herd/Orca workflow system

## Performance Considerations

### Asynchronous Design
- Immediate HTTP 202 response for better user experience
- Background processing prevents timeout issues
- Scalable through thread pool or workflow system

### Database Optimization
- Uses GSI for efficient purchase ID queries
- Version-based sorting for latest record retrieval
- Batch operations available for bulk processing

### Caching Strategy
[CONFIGURATION DEPENDENT] - No explicit caching implemented in the current flow

## Monitoring and Observability

### Logging Points
- Request receipt and transformation
- Database query operations
- Async handler invocation
- Error conditions and exceptions

### Metrics
[VERIFICATION NEEDED] - Specific metrics configuration depends on deployed environment

## Security

### Authentication
- **AAA (Amazon Authorization Architecture)** required
- Service name: `SantanderGILGPPArestPlugin`
- Operation name: `paymentProcessing`

### Authorization
[VERIFICATION NEEDED] - Specific authorization rules depend on AAA configuration

## Source Code References

### Controllers
- [`ChargeController.java`](../../src/com/amazon/santandergilgpparestplugin/controller/ChargeController.java) - Main controller handling authorize charge requests

### Transformers
- [`PSXTProcessorRequestTransformer.java`](../../src/com/amazon/santandergilgpparestplugin/transformer/PSXTProcessorRequestTransformer.java) - Request transformation logic

### Data Access Objects
- [`InstallmentPaymentEntityDao.java`](../../src/com/amazon/santandergilgpparestplugin/accessor/dao/InstallmentPaymentEntityDao.java) - Loan entity database operations
- [`PurchaseReferenceDao.java`](../../src/com/amazon/santandergilgpparestplugin/accessor/dao/PurchaseReferenceDao.java) - Purchase reference database operations

### Request Models
- [`AuthorizeRequest.java`](../../src/com/amazon/santandergilgpparestplugin/async/requests/AuthorizeRequest.java) - Authorization request model
- [`PaymentRequest.java`](../../src/com/amazon/santandergilgpparestplugin/async/requests/PaymentRequest.java) - Base payment request model
- [`Request.java`](../../src/com/amazon/santandergilgpparestplugin/async/requests/Request.java) - Base request model

### Async Handlers
- [`AsyncRequestHandler.java`](../../src/com/amazon/santandergilgpparestplugin/async/handler/AsyncRequestHandler.java) - Async handler interface
- [`ThreadPoolAsyncRequestHandler.java`](../../src/com/amazon/santandergilgpparestplugin/async/handler/ThreadPoolAsyncRequestHandler.java) - Thread pool implementation
- [`HerdAsyncRequestHandler.java`](../../src/com/amazon/santandergilgpparestplugin/async/handler/HerdAsyncRequestHandler.java) - Herd/Orca implementation

### Utilities
- [`LoanUtils.java`](../../src/com/amazon/santandergilgpparestplugin/loan/LoanUtils.java) - Loan management utilities
- [`PluginCustomFieldUtils.java`](../../src/com/amazon/santandergilgpparestplugin/utils/PluginCustomFieldUtils.java) - Custom field processing utilities

### Constants
- [`ServiceConstants.java`](../../src/com/amazon/santandergilgpparestplugin/constants/ServiceConstants.java) - Service configuration constants
