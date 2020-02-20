## DLNA协议

##### DLNA定义：自己百度和google

##### 协议基于: [upnp协议](../pdf/UPnP-arch-DeviceArchitecture-v2.0-20150220.pdf)

---

### 设备发现---ssdp协议：

1. udp编程

2. 向 **239.255.255.250**  端口（一般是1900 yeelight是1982）广播

3. 广播信息是
```http
--------------------------------------------------
M-SEARCH * HTTP/1.1
ST: upnp:rootdevice
MAN: "ssdp:discover"
HOST: 239.255.255.250:1900
MX: 10

--------------------------------------------------
注:
1、分割线不是最后一行需要空一行
2、ST可以有很多选项也可以百度
```

4. 返回信息
```http
HTTP/1.1 200 OK
Location: http://192.168.5.234:9999/4b329fd5-0aa6-4aca-96c1-23ab1e3db458.xml
Ext:
USN: uuid:4b329fd5-0aa6-4aca-96c1-23ab1e3db458::upnp:rootdevice
Server: Linux/4.9.61 UPnP/1.0 GUPnP/1.0.2
Cache-Control: max-age=1800
ST: upnp:rootdevice
Date: Thu, 20 Feb 2020 07:49:16 GMT
Content-Length: 0

```

* USN 身份标识；
* LOCATION：这个是获取详细信息的关键；解析拿到ip和端口号 具体看upnp协议；、

----

### 请求LOCATION后    ====>（本文采用的是小米AI音响）解析xml

* friendlyName：设备名称
* 找到urn:schemas-upnp-org:service:AVTransport:1 下的 controlURL 这个就是你需要的path（/AVTransport/control）加上上文提到的ip（192.168.5.234）和port（9999）就可以请求播放链接 http://192.168.5.234:9999/AVTransport/control；
* SCPDURL下面的页面可以看到你可以对这个设备进行的操作，有开始，启动，停止，暂停等等；

---

拿到链接后 使用的是**http post**方法

* 需要header 
  * Content-Type 为 text/xml; charset="utf-8"
  * SOAPAction 为 "urn:schemas-upnp-org:service:AVTransport:1#SetAVTransportURI"-------->这个主要是由serviceType+固定参数在上面SCPDURL可以看到，基本上是固定几个方法 play，stop等；
* 保持下方的数据请求包的结构就行注释的那个 DIDL-Lite 可有可不有；

### http-post请求包

```http
POST /AVTransport/control HTTP/1.1
Content-Type: text/xml; charset="utf-8"
SOAPAction: "urn:schemas-upnp-org:service:AVTransport:1#SetAVTransportURI"
User-Agent: PostmanRuntime/7.22.0
Accept: */*
Cache-Control: no-cache
Postman-Token: fa5bd279-e79e-4078-aadd-f8a3269f38b5
Host: 192.168.5.234:9999
Accept-Encoding: gzip, deflate, br
Content-Length: 1145
Connection: keep-alive

<s:Envelope s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" xmlns:u="urn:schemas-upnp-org:service:AVTransport:1">
    <s:Body>
        <u:SetAVTransportURI>
            <InstanceID>0</InstanceID>
            <CurrentURI>http://192.168.199.114/123.mp3</CurrentURI>
            <CurrentURIMetaData>
               <!-- <DIDL-Lite xmlns="urn:schemas-upnp-org:metadata-1-0/DIDL-Lite/" xmlns:dc="http://purl.org/dc/elements/1.1/" 
xmlns:sec="http://www.sec.co.kr/" xmlns:upnp="urn:schemas-upnp-org:metadata-1-0/upnp/">
                    <item id="f-0" parentID="0" restricted="0">
                        <dc:title>Video</dc:title>
                        <dc:creator>Anonymous</dc:creator>
                        <upnp:class>object.item.videoItem</upnp:class>
                        <res protocolInfo="http-get:*:video/*:DLNA.ORG_OP=01;DLNA.ORG_CI=0;DLNA.ORG_FLAGS=01700000000000000000000000000000" sec:URIType="public">%@</res>
                    </item>
                </DIDL-Lite>-->
            </CurrentURIMetaData>
        </u:SetAVTransportURI>
    </s:Body>
</s:Envelope>
```
### http-post返回包

```http
HTTP/1.1 200 OK
Date: Thu, 20 Feb 2020 07:55:18 GMT
Content-Type: text/xml; charset="utf-8"
Ext:
Server: Linux/4.9.61 UPnP/1.0 GUPnP/1.0.2
Content-Length: 287

<?xml version="1.0"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
    <s:Body>
        <u:SetAVTransportURIResponse xmlns:u="urn:schemas-upnp-org:service:AVTransport:1"></u:SetAVTransportURIResponse>
    </s:Body>
</s:Envelope>
```

