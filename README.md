# tcc-toy

## 使用方法

#### APP/TM(事务管理器)

``` go
func main() {
    err := transaction_manager.TCCCall(func(tx *transaction_manager.Transaction) transaction_manager.Operation {
        _, err := tx.CallTry("localhost:9210", &pb.TryRequest{Param: "tx1"})
        if err != nil {
            t.Errorf("tx1 try failed: %v", err)
            return tx.Cancel()
        }
        _, err = tx.CallTry("localhost:9211", &pb.TryRequest{Param: "tx2"})
        if err != nil {
            t.Errorf("tx2 try failed: %v", err)
            return tx.Cancel()
        }
        t.Logf("tx1 and tx2 try success")
        return tx.Commit()
    })
    if err != nil {
        panic(err)
    }
}
```

#### RM(资源管理器/需要执行事务的下游服务)

``` go
type MockResourceManager struct {
    pb.UnimplementedResourceManagerServer
}

func (manager *MockResourceManager) Try(_ context.Context, req *pb.TryRequest) (*pb.TryReply, error) {
    fmt.Printf("try -- xid: %s, param: %s\n", req.Xid, req.Param)
    return &pb.TryReply{}, nil
}

func (manager *MockResourceManager) Commit(_ context.Context, req *pb.CommitRequest) (*pb.CommitReply, error) {
    fmt.Printf("commit -- xid: %s\n", req.Xid)
    return &pb.CommitReply{}, nil
}

func (manager *MockResourceManager) Cancel(_ context.Context, req *pb.CancelRequest) (*pb.CancelReply, error) {
    fmt.Printf("cancel -- xid: %s\n", req.Xid)
    return &pb.CancelReply{}, nil
}

func main() {
    mock := &MockResourceManager{}
    redisCli := redis.NewClient(&redis.Options{Addr: "localhost:6379", Password: "123456"})
    rm := resource_manager.New(mock, redisCli, resource_manager.WithPort(9210))
    if err := rm.Run(); err != nil {
        panic(err)
    }
}
```