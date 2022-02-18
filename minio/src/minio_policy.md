<!-- ---
title: minio policy
date: 2018-11-15 14:00:47
category: src, minio, src
--- -->

minio policy

1. admin web 控制policy 
2. 前端访问时进行校验
3. policy 文件地址 .minio.sys/buckets/{bucket}/policy.json


```go
minio/cmd/server-main.go
serverMain

// Create new policy system.
globalPolicySys = NewPolicySys()

// Initialize policy system.
if err := globalPolicySys.Init(newObject); err != nil {
    logger.Fatal(err, "Unable to initialize policy system")
}


// PolicySys - policy subsystem.
type PolicySys struct {
    sync.RWMutex
    bucketPolicyMap map[string]policy.Policy
}

// Policy - bucket policy.
type Policy struct {
    ID         ID `json:"ID,omitempty"`
    Version    string
    Statements []Statement `json:"Statement"`
}




// NewPolicySys - creates new policy system.
func NewPolicySys() *PolicySys {
    return &PolicySys{
        bucketPolicyMap: make(map[string]policy.Policy),
    }
}

// Init - initializes policy system from policy.json of all buckets.
func (sys *PolicySys) Init(objAPI ObjectLayer) error {
    if objAPI == nil {
        return errInvalidArgument
    }

    // Load PolicySys once during boot.
    if err := sys.refresh(objAPI); err != nil {
        return err
    }

    // Refresh PolicySys in background.
    go func() {
        ticker := time.NewTicker(globalRefreshBucketPolicyInterval)
        defer ticker.Stop()
        for {
            select {
            case <-globalServiceDoneCh:
                return
            case <-ticker.C:
                sys.refresh(objAPI)
            }
        }
    }()
    return nil
}
```


访问策略语言 (policy) 概述 https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/access-policy-language-overview.html
访问控制列表 (ACL) 概述 https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/acl-overview.html
creat post https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/API/sigv4-HTTPPOSTConstructPolicy.html
