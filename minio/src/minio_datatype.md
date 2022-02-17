---
title: minio 数据结构
date: 2018-10-21 14:34:10
category: database, storage, minio, src
---

minio 数据结构

ObjectLayer 对象处理封装接口层 

```go
//全局object 处理
func newObjectLayerFn() (layer ObjectLayer) {
	globalObjLayerMutex.RLock()
	layer = globalObjectAPI
	globalObjLayerMutex.RUnlock()
	return
}

//ObjectLayer
//minio/cmd/object-api-interface.go
//ObjectLayer 规定了object 处理api 应该处理的功能接口
type ObjectLayer interface {
	// Storage operations.
	Shutdown(context.Context) error
	StorageInfo(context.Context) StorageInfo

	// Bucket operations.
	MakeBucketWithLocation(ctx context.Context, bucket string, location string) error
	GetBucketInfo(ctx context.Context, bucket string) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string) error
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)

	// Object operations.
	GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, lockType LockType, opts ObjectOptions) (reader *GetObjectReader, err error)
	GetObject(ctx context.Context, bucket, object string, startOffset int64, length int64, writer io.Writer, etag string, opts ObjectOptions) (err error)
	GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *hash.Reader, metadata map[string]string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	CopyObject(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string) error

	// Multipart operations.
	ListMultipartUploads(ctx context.Context, bucket, prefix, keyMarker, uploadIDMarker, delimiter string, maxUploads int) (result ListMultipartsInfo, err error)
	NewMultipartUpload(ctx context.Context, bucket, object string, metadata map[string]string, opts ObjectOptions) (uploadID string, err error)
	CopyObjectPart(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, uploadID string, partID int,
		startOffset int64, length int64, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (info PartInfo, err error)
	PutObjectPart(ctx context.Context, bucket, object, uploadID string, partID int, data *hash.Reader, opts ObjectOptions) (info PartInfo, err error)
	ListObjectParts(ctx context.Context, bucket, object, uploadID string, partNumberMarker int, maxParts int) (result ListPartsInfo, err error)
	AbortMultipartUpload(ctx context.Context, bucket, object, uploadID string) error
	CompleteMultipartUpload(ctx context.Context, bucket, object, uploadID string, uploadedParts []CompletePart) (objInfo ObjectInfo, err error)

	// Healing operations.
	ReloadFormat(ctx context.Context, dryRun bool) error
	HealFormat(ctx context.Context, dryRun bool) (madmin.HealResultItem, error)
	HealBucket(ctx context.Context, bucket string, dryRun bool) ([]madmin.HealResultItem, error)
	HealObject(ctx context.Context, bucket, object string, dryRun bool) (madmin.HealResultItem, error)
	ListBucketsHeal(ctx context.Context) (buckets []BucketInfo, err error)
	ListObjectsHeal(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (ListObjectsInfo, error)

	// Policy operations
	SetBucketPolicy(context.Context, string, *policy.Policy) error
	GetBucketPolicy(context.Context, string) (*policy.Policy, error)
	DeleteBucketPolicy(context.Context, string) error

	// Supported operations check
	IsNotificationSupported() bool
	IsEncryptionSupported() bool

	// Compression support check.
	IsCompressionSupported() bool
}
```

CacheObjectLayer

```go
//minio/cmd/disk-cache.go
// CacheObjectLayer implements primitives for cache object API layer.
//缓存对象处理接口定义层
type CacheObjectLayer interface {
	// Bucket operations.
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)
	GetBucketInfo(ctx context.Context, bucket string) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string) error
	// Object operations.
	GetObject(ctx context.Context, bucket, object string, startOffset int64, length int64, writer io.Writer, etag string) (err error)
	GetObjectInfo(ctx context.Context, bucket, object string) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *hash.Reader, metadata map[string]string) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string) error

	// Multipart operations.
	NewMultipartUpload(ctx context.Context, bucket, object string, metadata map[string]string) (uploadID string, err error)
	PutObjectPart(ctx context.Context, bucket, object, uploadID string, partID int, data *hash.Reader) (info PartInfo, err error)
	AbortMultipartUpload(ctx context.Context, bucket, object, uploadID string) error
	CompleteMultipartUpload(ctx context.Context, bucket, object, uploadID string, uploadedParts []CompletePart) (objInfo ObjectInfo, err error)

	// Storage operations.
	StorageInfo(ctx context.Context) CacheStorageInfo
}
```