# 通过 Apache CXF 生成 SOAP 服务客户端代码

## 环境准备

### Apache CXF

#### 下载`Apache CXF`

地址: http://cxf.apache.org/download.html

#### 配置环境变量

- 新建：CXF_HOME：D:\DevEnv\apache-cxf
- 追加：Path：%CXF_HOME%\bin;

### Java 配置

`Apache CXF`需要 JAVA_HOME 这个环境变量，所以需要以这种方式配置 Java。

> 注：Java 路径不能太长，可以临时在根目录放一个。

## 代码生成

首先明确 WSDL 文件的位置，例如：https://www.dovepay.com/BIAServer/wsdl/BIAWebService.wsdl

命令：wsdl2java

参数：

```bash
-p ：指定存放包名
-d  :指定存放本地目录
-encoding utf-8 ：指定字符集
-client :生成客户端代码 (未写参数默认生成的就是客户端的代码)
-uri参数指定了wsdl文件的路径
```

### 示例

```bash
wsdl2java -encoding utf-8 -d D:/ABC/abc https://www.dovepay.com/BIAServer/wsdl/BIAWebService.wsdl
```

生成的代码结构如下：

```bash
.
├── com
│   └── acca
│       └── bia
│           └── webservice
│               ├── AdultSingleTicket.java
│               ├── AdultSingleTicketResponse.java
│               ├── BIAWebService.java
│               ├── BIAWebServiceService.java
│               ├── CancelPayment.java
│               ├── CancelPaymentResponse.java
│               ├── DeleteReservationRecord.java
│               ├── DeleteReservationRecordResponse.java
│               ├── DeleteSpecifiedGroup.java
│               ├── DeleteSpecifiedGroupResponse.java
│               ├── DetrDirectiveService.java
│               ├── DetrDirectiveServiceResponse.java
│               ├── ExtractCAPnr.java
│               ├── ExtractCAPnrResponse.java
│               ├── FfService.java
│               ├── FfServiceResponse.java
│               ├── FindPrice.java
│               ├── FindPriceResponse.java
│               ├── FreightPriceInquiry.java
│               ├── FreightPriceInquiryResponse.java
│               ├── FreightPriceRecord.java
│               ├── FreightPriceRecordResponse.java
│               ├── GetAvailability.java
│               ├── GetAvailabilityResponse.java
│               ├── GetSchedule.java
│               ├── GetScheduleResponse.java
│               ├── GroupPassengerTktInfo.java
│               ├── GroupPassengerTktInfoResponse.java
│               ├── InsurRefund.java
│               ├── InsurRefundResponse.java
│               ├── IpayBuyInsur.java
│               ├── IpayBuyInsurResponse.java
│               ├── IssueTkt.java
│               ├── IssueTktResponse.java
│               ├── ObjectFactory.java
│               ├── PayOutTkt.java
│               ├── PayOutTktResponse.java
│               ├── QueryAccountBalanceInfo.java
│               ├── QueryAccountBalanceInfoResponse.java
│               ├── QueryIssueTktInfo.java
│               ├── QueryIssueTktInfoResponse.java
│               ├── RtDirectiveService.java
│               ├── RtDirectiveServiceResponse.java
│               ├── RtNmService.java
│               ├── RtNmServiceResponse.java
│               ├── RtktService.java
│               ├── RtktServiceResponse.java
│               ├── ThirdPartBuyInsur.java
│               ├── ThirdPartBuyInsurResponse.java
│               ├── VtTkt.java
│               ├── VtTktResponse.java
│               └── package-info.java
└── tree.txt

4 directories, 53 files

```

取自己需要的代码使用。

这些代码是可以自己写的，不过考虑到既然可以自动生成，说明规律性很强，对这方面不太了解的话担心出错，所以选择自动生成。

### 其他问题

生成的代码，如果所有接口都使用，建议把所有文件都放到工程里，如果只调用某一个接口，需要注意把`package-info.java`也放到工程里，否则调用时会报错`javax.xml.ws.webserviceexception class do not have a property of the name`。

### 另

用 IDEA 也可以自动生成代码和测试,并且 IDEA 版本不同功能会略有不同，老版本的 IDEA 功能会更多一些(指生成的方法)
