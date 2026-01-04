# GoTCC - Distributed Transaction TCC Framework

<p align="center">
<img src="https://github.com/xiaoxuxiansheng/gotcc/blob/main/img/sdk_frame.png" height="400px/"><br/><br/>
<b>GoTCC: A Lightweight TCC Distributed Transaction Framework Implemented in Pure Golang</b>
<br/><br/>
<a title="Go Report Card" target="_blank" href="https://goreportcard.com/report/github.com/xiaoxuxiansheng/gotcc"><img src="https://goreportcard.com/badge/github.com/xiaoxuxiansheng/gotcc?style=flat-square" /></a>
<a title="Codecov" target="_blank" href="https://codecov.io/gh/xiaoxuxiansheng/gotcc">
<img src="https://img.shields.io/codecov/c/github/xiaoxuxiansheng/gotcc?style=flat-square&logo=codecov"/>
</a>
<img src="https://img.shields.io/badge/Go-1.19+-00ADD8?style=flat-square&logo=go" />
<img src="https://img.shields.io/badge/license-MIT-green?style=flat-square" />
</p>

## 📖 Project Introduction

GoTCC is a distributed transaction solution framework based on the TCC (Try-Confirm-Cancel) pattern, implemented in pure Golang. This framework provides complete two-phase commit coordination capabilities, helping developers easily implement distributed transaction consistency guarantees across services and databases.

### ✨ Core Features

- 🎯 **Standard TCC Implementation**: Full support for Try-Confirm-Cancel two-phase commit protocol
- 🔄 **Transaction Coordinator**: Automatically manages the full lifecycle of distributed transactions
- 🛡️ **Exception Recovery Mechanism**: Built-in transaction monitoring and automatic compensation capabilities
- 🔌 **Flexible Extension**: Supports custom transaction storage and TCC components
- 📊 **Transaction Logging**: Complete transaction execution trace recording
- ⚡ **High Performance**: Lightweight design with low-latency transaction processing
- 🔒 **Distributed Lock**: Redis-based distributed lock ensuring concurrent safety

## 📚 Theoretical Foundation

Before using this framework, it is recommended to understand the theoretical knowledge of TCC distributed transactions to achieve unity of knowledge and action.

<img src="https://github.com/xiaoxuxiansheng/gotcc/blob/main/img/tcc_theory_frame.png" height="550px"/>

### TCC Pattern Description

TCC is a compensating distributed transaction solution consisting of three phases:

- **Try Phase**: Attempts to execute business logic, completes all business checks, and reserves necessary business resources
- **Confirm Phase**: Confirms business execution, actually executes business logic, and uses resources reserved in the Try phase
- **Cancel Phase**: Cancels business execution and releases resources reserved in the Try phase

### 📝 Related Technical Articles

- [TCC Technical Theory](https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247484585&idx=1&sn=b5ee56c2334e3cf4e9a1d8d9b54cd02c)
- [TCC Open Source Practice](https://mp.weixin.qq.com/s?__biz=MzkxMjQzMjA0OQ==&mid=2247484619&idx=1&sn=2415f0b9c1e043c22ae2fd6d75d6cbb3)

## 🚀 Quick Start

### Environment Requirements

- Go 1.19+
- MySQL (for transaction log storage)
- Redis (for distributed locks)

### Installation

```bash
go get github.com/xiaoxuxiansheng/gotcc
```

### Core Architecture

<img src="https://github.com/xiaoxuxiansheng/gotcc/blob/main/img/2pc.png" height="400px"/>

The framework mainly includes the following core modules:

1. **TXManager**: Transaction coordinator, responsible for orchestrating the Try-Confirm/Cancel two-phase process
2. **TXStore**: Transaction log storage module for persisting transaction state
3. **TCCComponent**: TCC component interface that business parties need to implement
4. **RegistryCenter**: Component registry center that manages all TCC components

## 📋 Integration Guide

### Step 1: Implement TXStore Interface

First, you need to implement the transaction log storage module for persisting transaction state:

```go
// Transaction log storage module
type TXStore interface {
    // Create a transaction detail record
    CreateTX(ctx context.Context, components ...component.TCCComponent) (txID string, err error)
    
    // Update transaction progress: actually updates each component's try request response result
    TXUpdate(ctx context.Context, txID string, componentID string, accept bool) error
    
    // Submit the final state of the transaction, indicating whether the transaction execution succeeded or failed
    TXSubmit(ctx context.Context, txID string, success bool) error
    
    // Get all incomplete transactions
    GetHangingTXs(ctx context.Context) ([]*Transaction, error)
    
    // Get a specific transaction
    GetTX(ctx context.Context, txID string) (*Transaction, error)
    
    // Lock the entire TXStore module (requires distributed lock)
    Lock(ctx context.Context, expireDuration time.Duration) error
    
    // Unlock the TXStore module
    Unlock(ctx context.Context) error
}
```

**Reference Implementation**: See the example implementation in `example/txstore.go`.

### Step 2: Implement TCCComponent Interface

Business parties need to implement the TCC component interface to define specific business logic:

```go
// TCC Component
type TCCComponent interface {
    // Return the unique ID of the component
    ID() string
    
    // Execute the try operation of the first phase
    Try(ctx context.Context, req *TCCReq) (*TCCResp, error)
    
    // Execute the confirm operation of the second phase
    Confirm(ctx context.Context, txID string) (*TCCResp, error)
    
    // Execute the cancel operation of the second phase
    Cancel(ctx context.Context, txID string) (*TCCResp, error)
}
```

**Reference Implementation**: See the example implementation in `example/tcccomponent.go`.

### Step 3: Register Components and Execute Transactions

```go
// Create transaction manager
txManager := gotcc.NewTXManager(
    txStore, 
    gotcc.WithMonitorTick(time.Second),  // Set monitoring period
    gotcc.WithTimeout(time.Second*30),    // Set transaction timeout
)
defer txManager.Stop()

// Register TCC component
if err := txManager.Register(componentA); err != nil {
    return err
}

// Execute distributed transaction
txID, success, err := txManager.Transaction(ctx, []*gotcc.RequestEntity{
    {
        ComponentID: "componentA",
        Request: map[string]interface{}{
            "biz_id": "business_001",
            "amount": 100,
        },
    },
    {
        ComponentID: "componentB",
        Request: map[string]interface{}{
            "biz_id": "business_002",
            "amount": 200,
        },
    },
}...)

if err != nil {
    log.Printf("transaction failed: %v", err)
    return
}

if success {
    log.Printf("transaction %s completed successfully", txID)
} else {
    log.Printf("transaction %s failed", txID)
}
```

## 💡 Complete Example

See the complete example code in the `example` directory:

```go
const (
    dsn      = "root:password@tcp(127.0.0.1:3306)/tcc_db?charset=utf8mb4&parseTime=True&loc=Local"
    network  = "tcp"
    address  = "127.0.0.1:6379"
    password = ""
)

func Test_TCC(t *testing.T) {
    // Initialize Redis client
    redisClient := pkg.NewRedisClient(network, address, password)
    
    // Initialize MySQL database
    mysqlDB, err := pkg.NewDB(dsn)
    if err != nil {
        t.Error(err)
        return
    }

    componentAID := "componentA"
    componentBID := "componentB"
    componentCID := "componentC"

    // Build TCC components
    componentA := NewMockComponent(componentAID, redisClient)
    componentB := NewMockComponent(componentBID, redisClient)
    componentC := NewMockComponent(componentCID, redisClient)

    // Build transaction log storage module
    txRecordDAO := dao.NewTXRecordDAO(mysqlDB)
    txStore := NewMockTXStore(txRecordDAO, redisClient)

    // Create transaction manager (check hanging transactions every second)
    txManager := gotcc.NewTXManager(txStore, gotcc.WithMonitorTick(time.Second))
    defer txManager.Stop()

    // Register each component
    if err := txManager.Register(componentA); err != nil {
        t.Error(err)
        return
    }

    if err := txManager.Register(componentB); err != nil {
        t.Error(err)
        return
    }

    if err := txManager.Register(componentC); err != nil {
        t.Error(err)
        return
    }

    // Execute distributed transaction
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*30)
    defer cancel()
    
    txID, success, err := txManager.Transaction(ctx, []*gotcc.RequestEntity{
        {
            ComponentID: componentAID,
            Request: map[string]interface{}{
                "biz_id": componentAID + "_biz",
            },
        },
        {
            ComponentID: componentBID,
            Request: map[string]interface{}{
                "biz_id": componentBID + "_biz",
            },
        },
        {
            ComponentID: componentCID,
            Request: map[string]interface{}{
                "biz_id": componentCID + "_biz",
            },
        },
    }...)
    
    if err != nil {
        t.Errorf("tx failed, err: %v", err)
        return
    }
    
    if !success {
        t.Error("tx failed")
        return
    }

    t.Logf("transaction %s success", txID)
}
```

## 🏗️ Project Structure

```
.
├── component.go         # TCC component interface definition
├── model.go            # Data model definition
├── option.go           # Configuration options
├── tccregister.go      # Component registry center
├── txmanager.go        # Transaction manager core implementation
├── txstore.go          # Transaction storage interface definition
├── example/            # Example code
│   ├── tcccomponent.go # TCC component example implementation
│   ├── txstore.go      # Transaction storage example implementation
│   ├── dao/            # Data access layer
│   │   ├── txrecord.go # Transaction record DAO
│   │   └── txrecord.sql # Database table structure
│   └── pkg/            # Utility package
│       ├── mysql.go    # MySQL client
│       └── redis.go    # Redis client
└── log/                # Log module
    └── log.go
```

## ⚙️ Configuration Options

GoTCC supports the following configuration options:

```go
type Options struct {
    Timeout     time.Duration  // Transaction timeout
    MonitorTick time.Duration  // Monitoring period (interval for checking hanging transactions)
}

// Usage example
txManager := gotcc.NewTXManager(
    txStore,
    gotcc.WithTimeout(time.Second*30),      // Set 30 second timeout
    gotcc.WithMonitorTick(time.Second*5),   // Check hanging transactions every 5 seconds
)
```

## 🔍 Core Workflow

### 1. Normal Flow

```
1. TXManager creates transaction record and generates globally unique transaction ID
2. Concurrently calls the Try method of all TCC components
3. If all Try calls succeed, concurrently calls the Confirm method of all components
4. If any Try fails, concurrently calls the Cancel method of all components
5. Updates final transaction state
```

### 2. Exception Recovery

The framework has built-in transaction monitoring mechanism that periodically scans hanging (incomplete) transactions:

```
1. Background goroutine periodically checks all hanging transactions
2. For timed-out transactions, attempts to re-execute Confirm or Cancel
3. Uses distributed lock to ensure only one instance processes hanging transactions at a time
4. Supports exponential backoff strategy to avoid frequent retries
```

## 🎯 Application Scenarios

GoTCC is suitable for the following distributed transaction scenarios:

- **Order System**: Order creation + inventory deduction + points addition
- **Payment System**: Account debit + merchant credit + transaction record
- **E-commerce System**: Place order + reduce inventory + redeem coupon
- **Financial System**: Transfer + deduct fees + notify third party

## ⚠️ Considerations

### TCC Component Implementation Points

1. **Try Phase**
   - Complete business checks
   - Reserve necessary resources (such as freezing inventory, freezing balance)
   - Idempotency guarantee required

2. **Confirm Phase**
   - Use resources reserved in the Try phase
   - Must succeed (eventual consistency guarantee)
   - Idempotency guarantee required

3. **Cancel Phase**
   - Release resources reserved in the Try phase
   - Must succeed (eventual consistency guarantee)
   - Idempotency guarantee required

### Idempotency Design

All TCC operations must guarantee idempotency. Recommended approach:

```go
// Use transaction ID + component ID as unique key
func (c *Component) Try(ctx context.Context, req *TCCReq) (*TCCResp, error) {
    // Check if already executed
    if c.isExecuted(req.TXID, req.ComponentID, "try") {
        return &TCCResp{ACK: true}, nil
    }
    
    // Execute business logic...
    
    // Record execution state
    c.recordExecution(req.TXID, req.ComponentID, "try")
    return &TCCResp{ACK: true}, nil
}
```

## 🧪 Testing

Run unit tests:

```bash
go test -v ./...
```

Run example tests:

```bash
# Need to start MySQL and Redis first
cd example
go test -v
```

## 📊 Performance Optimization Recommendations

1. **Concurrency Control**: Reasonably set the number of TCC components to avoid resource contention from excessive concurrency
2. **Timeout Setting**: Set reasonable timeout based on business scenarios
3. **Monitoring Period**: Adjust the check frequency of hanging transactions based on business importance
4. **Connection Pool**: Use connection pools to manage MySQL and Redis connections
5. **Log Level**: INFO level recommended for production environments

## 🤝 Contributing

Contributions, issues, and suggestions are welcome!

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Submit a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

Thanks to all the developers who have contributed to this project!

---

<p align="center">
If this project helps you, please give it a ⭐️ Star to support us!
</p>



