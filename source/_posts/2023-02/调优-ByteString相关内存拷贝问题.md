---
title: 调优-ByteString相关内存拷贝问题
date: 2023-02-03 11:01:39
tags: 
- java
- jvm
- ByteString
- HttpRange

categories:
- java
- 性能


---

# 背景
媒体中心api内存消耗较大，检查内存分配情况:
{% img /images/2023-02/bytestring-background.png 800 1200 background %}

对应的伪代码:
```java
1. byte[] bytes = BlobStore.loadFile(); 
2. ByteString pb = ByteString.copyFrom(bytes); 
3. ByteString afterDecrypt = callRpc(pb); // 2次拷贝 
4. byte[] decryptBytes = afterDecryp.toByteArray(); 
5. byte[] resp = XXXUtils.downloadRange(decryptBytes);
```

涉及到的堆内内存申请：（堆外暂且不管）
1.业务线程: 从blobstore读取数据；
2.业务线程: 解密前拷贝给pb；
3.grpc线程: 接收rpc结果;
4.grpc线程: 从结果拷贝到resp;
5.业务线程: 从resp拷贝到byte[]; 
6.业务线程: range下载,byte[]到byte[]。

# 解决方案
## ByteString.copyFrom优化
### 常规写法
```java
byte[] videoBytes = doLoadFile(videoKey);
return ByteString.copyFrom(videoBytes);
```
会多一份内存申请的内存消耗和拷贝的性能消耗。

> ByteString 实例通常使用 ByteString.CopyFrom(byte[] data) 创建。 此方法会分配新的 ByteString 和新的 byte[]。 数据会复制到新的字节数组中。
  
  通过使用 UnsafeByteOperations.UnsafeWrap(ReadOnlyMemory<byte> bytes) 创建 ByteString 实例，可以避免其他分配和复制操作。
### 优化写法
```java
// 要求fileBytes是immutable的，数据不能再修改
ByteString data = UnsafeByteOperations.unsafeWrap(fileBytes);
```

### 注意事项：
1。 UnsafeByteOperations.UnsafeWrap 要求使用 Google.Protobuf 版本 3.15.0 或更高版本：
参考:
https://learn.microsoft.com/zh-cn/aspnet/core/grpc/performance?view=aspnetcore-7.0
2。 如果修改了数据可能会导致抛各种异常:
参考: https://cloud.google.com/java/docs/reference/protobuf/latest/com.google.protobuf.UnsafeByteOperations

# ByteString.toByteArray优化
ByteString有多种实现，不一定内部有byte数组,所以要根据实际情况选择`inputStream`或者`byte`
```java
// 1. 方法1:
public abstract InputStream newInput();
// 2. 方法2:
public abstract ByteBuffer asReadOnlyByteBuffer()  
```

## RangeDownload优化
spring5的`org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor`默认实现对于大部分Resouce类型的Http Range协议支持，尽量不要自己实现Range协议，因为里面的规范、边界还是很多很繁琐的；
目前XXXUtils实现的版本就会多一次byte数组的拷贝问题，也没有实现全部规范。
spring的处理源码:
```java
// org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor#writeWithMessageConverters(T, org.springframework.core.MethodParameter, org.springframework.http.server.ServletServerHttpRequest, org.springframework.http.server.ServletServerHttpResponse)
if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}
```

### 示例使用
底层是byte数组时:
```java
@RequestMapping("/load")
    @ResponseBody
    public ResponseEntity<ByteArrayResource> load(@RequestParam(value = "fileKey") String fileKey, HttpServletRequest request) {
        byte[] fileBytes = doLoadFile(fileKey, false);
        MediaType mediaType = parseMediaType(fileKey);
        final HttpHeaders headers = new HttpHeaders();
        headers.setContentType(mediaType);
        headers.set("Content-Disposition", "attachment; filename=" + fileKey);
        headers.set("Access-Control-Allow-Headers", "Range,Content-Length");
        headers.set("Access-Control-Allow-Methods", "GET,POST,OPTIONS");
        headers.set("Access-Control-Allow-Origin", "*");
        return ResponseEntity.ok()
                .headers(headers)
                .body(new ByteArrayResource(fileBytes));
    }
```
底层是ByteString时:
```java
@RequestMapping("/video")
@ResponseBody
public ResponseEntity<ByteStringResource> loadVideo(HttpServletRequest request, @RequestParam(value = "fileKey") String fileKey){
  ByteString byteString = doLoadFileV2(fileKey);
            return ResponseEntity.ok()
                    .contentType(VIDEO_TYPE)
                    .cacheControl(maxAge(DEFAULT_CACHE_AGE, DAYS).cachePublic())
                    .eTag(eTag)
                    .body(new ByteStringResource(byteString));
}
```

### 注意事项
并不是所有`Resource`类型spring都支持了`Http Range`，可以看spring源码特别单独排除了`InputStreamResource`类型。
相关的拦截部分源码:
```java
/**
	 * Return whether the returned value or the declared return type extends {@link Resource}.
	 */
	protected boolean isResourceType(@Nullable Object value, MethodParameter returnType) {
		Class<?> clazz = getReturnValueType(value, returnType);
		return clazz != InputStreamResource.class && Resource.class.isAssignableFrom(clazz);
	}

/**
	 * Turn a {@code Resource} into a {@link ResourceRegion} using the range
	 * information contained in the current {@code HttpRange}.
	 * @param resource the {@code Resource} to select the region from
	 * @return the selected region of the given {@code Resource}
	 * @since 4.3
	 */
	public ResourceRegion toResourceRegion(Resource resource) {
		// Don't try to determine contentLength on InputStreamResource - cannot be read afterwards...
		// Note: custom InputStreamResource subclasses could provide a pre-calculated content length!
		Assert.isTrue(resource.getClass() != InputStreamResource.class,
				"Cannot convert an InputStreamResource to a ResourceRegion");
		long contentLength = getLengthFor(resource);
		long start = getRangeStart(contentLength);
		long end = getRangeEnd(contentLength);
		return new ResourceRegion(resource, start, end - start + 1);
	}
```
原因是Range协议中需要获取contentLength，
而InputStreamResource的数据大小获取的默认实现是将inputStream先遍历一遍，这样显然是不符合实际使用场景的。(只能读1次数据)
所以如果我们拿到的是inputStream+数据大小时，我们需要将contentLength自行实现一个版本（不遍历的）。
这里可以参考ByteStringResource:
```java
/**
 * @author fengmengqi <fengmengqi@xxx.com>
 * Created on 2023-02-02
 */
public class ByteStringResource extends AbstractResource {
    private final ByteString byteString;

    private final String description;

    /**
     * Create a new {@code ByteStringResource}.
     *
     * @param byteString the byteString to wrap
     */
    public ByteStringResource(ByteString byteString) {
        this(byteString, "resource loaded from byteString");
    }

    /**
     * Create a new {@code ByteStringResource} with a description.
     *
     * @param byteString the byteString to wrap
     * @param description where the byteString comes from
     */
    public ByteStringResource(ByteString byteString, @Nullable String description) {
        Assert.notNull(byteString, "ByteString must not be null");
        this.byteString = byteString;
        this.description = (description != null ? description : "");
    }


    /**
     * Return the underlying byteString.
     */
    public final ByteString getByteString() {
        return this.byteString;
    }

    /**
     * This implementation always returns {@code true}.
     */
    @Override
    public boolean exists() {
        return true;
    }

    /**
     * This implementation returns the length of the underlying byte array.
     */
    @Override
    public long contentLength() {
        return this.byteString.size();
    }

    /**
     * This implementation returns a ByteArrayInputStream for the
     * underlying byte array.
     *
     * @see java.io.ByteArrayInputStream
     */
    @Override
    public InputStream getInputStream() throws IOException {
        return byteString.newInput();
    }

    /**
     * This implementation returns a description that includes the passed-in
     * {@code description}, if any.
     */
    @Override
    public String getDescription() {
        return "ByteString resource [" + this.description + "]";
    }


    /**
     * This implementation compares the underlying byte array.
     *
     * @see java.util.Arrays#equals(byte[], byte[])
     */
    @Override
    public boolean equals(Object other) {
        return (this == other || (other instanceof ByteStringResource
                && ((ByteStringResource) other).byteString.equals(this.byteString)));
    }

    /**
     * This implementation returns the hash code based on the
     * underlying byte array.
     */
    @Override
    public int hashCode() {
        return (byte[].class.hashCode() * 29 * this.byteString.size());
    }

}

```


## 展望
理论上这个场景下（假如不修改业务逻辑），最小内存申请次数应该是2次而不是3次。（解密前、解密后）
目前多出来的这一次，是背景一节中，grpc代码对于一次数据的响应会进行两次堆内内存的申请，相关源码参考：

```java
// com.google.protobuf.CodedInputStream.StreamDecoder#readBytesSlowPath
/**
     * Like readBytes, but caller must have already checked the fast path: (size <= (bufferSize -
     * pos) && size > 0 || size == 0)
     */
    private ByteString readBytesSlowPath(final int size) throws IOException {
      final byte[] result = readRawBytesSlowPathOneChunk(size);
      if (result != null) {
        // We must copy as the byte array was handed off to the InputStream and a malicious
        // implementation could retain a reference.
        return ByteString.copyFrom(result);
      }

      final int originalBufferPos = pos;
      final int bufferedBytes = bufferSize - pos;

      // Mark the current buffer consumed.
      totalBytesRetired += bufferSize;
      pos = 0;
      bufferSize = 0;

      // Determine the number of bytes we need to read from the input stream.
      int sizeLeft = size - bufferedBytes;

      // The size is very large. For security reasons we read them in small
      // chunks.
      List<byte[]> chunks = readRawBytesSlowPathRemainingChunks(sizeLeft);

      // OK, got everything.  Now concatenate it all into one buffer.
      final byte[] bytes = new byte[size];

      // Start by copying the leftover bytes from this.buffer.
      System.arraycopy(buffer, originalBufferPos, bytes, 0, bufferedBytes);

      // And now all the chunks.
      int tempPos = bufferedBytes;
      for (final byte[] chunk : chunks) {
        System.arraycopy(chunk, 0, bytes, tempPos, chunk.length);
        tempPos += chunk.length;
      }
      
      return ByteString.wrap(bytes);
    }
```

主要是`readRawBytesSlowPathOneChunk`和`copyFrom`这两次。
2020年有人发现了类似情况，相关issue参考：
https://github.com/protocolbuffers/protobuf/issues/7899
目前还比较遗憾出于安全角度（回复是要保持`immutable`）被拒绝无法优化。

实际上用户观看一个视频的过程中chrome首先会发一个bytes=0-的请求（最早可能还有一个http到https的307），
然后如果服务端支持range，chrome会分段range请求，所以服务端会收到同一个文件的多个http请求。
本篇优化了一半内存分配，最坏gc时间压平到2s；
再加上临时磁盘缓存，同一个文件多次http请求只会分配3次内存，再调优一下gc配置（固定新生代大小），最坏gc时间压低到了130ms，平时大概是90ms。

```shell script
#!/bin/bash
# 删除60min未访问视频
find /data/tmp/ -amin +60 -name '*.mp4' -type f -delete
find /data/tmp/ -amin +60 -name '*.MOV' -type f -delete
```