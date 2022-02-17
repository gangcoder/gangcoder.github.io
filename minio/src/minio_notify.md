---
title: minio notify
date: 2018-11-16 12:51:28
category: src, minio, src
---

minio notify

1. server-main 中初始化notify
2. 加载bucket 的notify target
3. 触发event 时调用Send 将event 发往target


```go
minio/cmd/server-main.go
serverMain

// Create new notification system.
globalNotificationSys = NewNotificationSys(globalServerConfig, globalEndpoints)

// Initialize notification system.
globalNotificationSys.Init(newObject)


// NewNotificationSys - creates new notification system object.
func NewNotificationSys(config *serverConfig, endpoints EndpointList) *NotificationSys {
    targetList := getNotificationTargets(config)
    peerRPCClientMap := makeRemoteRPCClients(endpoints)

    // bucketRulesMap/bucketRemoteTargetRulesMap are initialized by NotificationSys.Init()
    return &NotificationSys{
        targetList:                 targetList,
        bucketRulesMap:             make(map[string]event.RulesMap),
        bucketRemoteTargetRulesMap: make(map[string]map[event.TargetID]event.RulesMap),
        peerRPCClientMap:           peerRPCClientMap,
    }
}


// Init - initializes notification system from notification.xml and listener.json of all buckets.
func (sys *NotificationSys) Init(objAPI ObjectLayer) error {
    if objAPI == nil {
        return errInvalidArgument
    }

    buckets, err := objAPI.ListBuckets(context.Background())
    if err != nil {
        return err
    }

    for _, bucket := range buckets {
        ctx := logger.SetReqInfo(context.Background(), &logger.ReqInfo{BucketName: bucket.Name})
        config, err := readNotificationConfig(ctx, objAPI, bucket.Name)
        if err != nil {
            if !IsErrIgnored(err, errDiskNotFound, errNoSuchNotifications) {
                return err
            }
        } else {
            sys.AddRulesMap(bucket.Name, config.ToRulesMap())
        }

        if err = sys.initListeners(ctx, objAPI, bucket.Name); err != nil {
            return err
        }
    }

    return nil
}


minio/pkg/event/targetlist.go
target list


发送notify

sendEvent(eventArgs{
    EventName:  event.ObjectRemovedDelete,
    BucketName: bucket,
    Object: ObjectInfo{
        Name: dobj.ObjectName,
    },
    ReqParams: extractReqParams(r),
    UserAgent: r.UserAgent(),
    Host:      host,
    Port:      port,
})


minio/cmd/notification.go
sendEvent

Send

send

minio/pkg/event/targetlist.go
Send

minio/pkg/event/targetlist.go
// Target - event target interface
type Target interface {
    ID() TargetID
    Send(Event) error
    Close() error
}
```


事件消息结构
https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/notification-content-structure.html

