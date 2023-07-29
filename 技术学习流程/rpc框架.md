## Thrift
Thrift支持二进制，压缩格式，以及json格式数据的序列化和反序列化。开发人员可以更加灵活的选择协议的具体形式。协议是可自由扩展的，新版本的协议，完全兼容老的版本！
### 解析格式
thrfit属于半解析型，丢弃了部分信息，比如field名称，但引入了index(常常是id+type的方式)来对应具体属性和值
而jdk的serizable属于自解析型
### thrift序列化协议
可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本(text)和二进制(binary)传输协议。
1. TBinaryProtocol：二进制编码格式进行数据传输
2. TCompactProtocol：高效率的、密集的二进制编码格式进行数据传输
3. TJSONProtocol： 使用JSON文本的数据编码协议进行数据传输
4. TSimpleJSONProtocol：只提供JSON只写的协议，适用于通过脚本语言解析