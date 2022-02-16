---
title: http_range
date: 2022-02-16 17:27:26
tag:
- http
- java
- spring
categories: 
- http

---

# http协议header中的range相关
客户端可以在http请求的header中指定请求资源的范围(range)，如果服务端支持的话，就可以只返回部分数据，节约网络资源。
如果服务端实现了这部分协议，客户端就可以进行断点续传、多线程下载、在线播放视频修改进度等功能了。
应用场景：
1。在线播放视频：调整进度；（seek offset）
2。下载文件：多线程下载、断点续传。

## 检查服务端是否支持
可以通过命令:
```shell script
curl -I https://cdn.com/xxx.png
```
来检查服务端是否支持。
如果支持,服务端的回复是bytes：
```
Accept-Ranges: bytes
```
如果不支持，服务端的回复是none:
```
Accept-Ranges: none
```
(`curl -I`只显示header; `curl -i`显示header和body)

## 实际应用中的回复
客户端发送:
```shell script
curl -i -H "Range: bytes=0-1023" https://cdn.com/xxx.png 
```
服务端正常回复206(单个区间的话):
```shell script
HTTP/1.1 206 Partial Content
Content-Type: image/png
Content-Length: 1024
Connection: keep-alive
ETag: "2F4AF992F06D75CA7BE353ED2A981C45"
Date: Tue, 01 Feb 2022 15:54:43 GMT
Last-Modified: Tue, 25 Jan 2022 15:54:42 GMT
Expires: Tue, 08 Feb 2022 15:54:43 GMT
Age: 579015
Cache-Control: max-age=604800
Content-Range: bytes 0-1023/9404
Accept-Ranges: bytes
```

如果range不合法的话，服务端回复:
```shell script
HTTP/1.1 416 Requested Range Not Satisfiable
```
如果服务端不支持range请求，会回复200，并且返回全部数据:
```shell script
HTTP/1.1 200 OK
```

其他Range相关的一些姿势可以参考：
https://www.greenbytes.de/tech/webdav/draft-ietf-httpbis-p5-range-latest.html#rule.ranges-specifier
实际还可以指定多区间，指定末尾100个字节（`Range: bytes=-100`），
还可以用`If-Range`避免文件发生变更的情况： 
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Range
只有Last-Modified或者ETag字段验证通过的时候，才处理`Range`请求，否则返回200（整个文件）。

# 实现：spring中
## 方案1，交给spring接管
如果是本地机器中的资源，可以封装成`FileSystemResource`；
如果要从别的地方获取byte数组，可以封装成`ByteArrayResource`;
状态直接写200(OK),spring会自动转成206。
示例代码：
```java
@RestController
@RequestMapping(value = "/v3/csc/center/range")
public class CscCenterRangeController {
    @RequestMapping("/load")
    @ResponseBody
    public ResponseEntity<ByteArrayResource> load(@RequestParam(value = "fileKey") String fileKey) {
        String res = "1234";
        final HttpHeaders responseHeaders = new HttpHeaders();
        // responseHeaders.add("Content-Type", "video/mp4");
        return new ResponseEntity<>(new ByteArrayResource(res.getBytes()), responseHeaders, HttpStatus.OK);
    }

    @RequestMapping("/load-file")
    @ResponseBody
    public ResponseEntity<FileSystemResource> loadFile(@RequestParam(value = "fileKey") String fileKey) {
        String filePathString = "/Users/fengmengqi/Documents/DH_scret.png";
        final HttpHeaders responseHeaders = new HttpHeaders();
        // responseHeaders.add("Content-Type", "video/mp4");
        return new ResponseEntity<>(new FileSystemResource(filePathString), responseHeaders, HttpStatus.OK);
    }
}
```
缺点是spring似乎没有实现Range协议中的`If-Range`头。

## 方案2，在spring提供的HttpRange类基础上实现
可以参考`org.springframework.web.servlet.resource.ResourceHttpRequestHandler#handleRequest`
在HttpRange类的基础上实现自定义的Range相关协议，这样可以增添`If-Range`的功能。


# 参考资料
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Range_requests
https://www.greenbytes.de/tech/webdav/draft-ietf-httpbis-p5-range-latest.html#rule.ranges-specifier


