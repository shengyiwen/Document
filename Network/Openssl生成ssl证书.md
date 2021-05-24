## 1.什么是x509证书链

- x509证书一般会用到三类文件 key, csr, crt

- key:是私用密钥，openssl格式，通常rsa算法

- csr:证书请求文件
  
用于申请证书，在制作csr的时候，必须使用自己的私钥来签署申请，还可以设定一个密钥

- crt:ca认证后的证书文件，签署人用自己的key给你签署的凭证

## 2.openssl中后缀文件名解释

- .key:私钥
- .csr:证书请求文件，包含公钥信息
- .crt:证书文件
- .crl:证书吊销列表
- .pem:用于导出，导入证书时候的证书格式


## 3.证书生成过程

    1.生成CA证书密钥
    
    openssl genrsa -out ca-key.pem 1024
    
    2.创建CA证书请求
    
    openssl req -new -out ca-req.csr -key ca-key.pem
    
    3.自签署CA证书
    
    openssl x509 -req -in ca-req.csr -out ca-cert.pem -signkey ca-key.pem -days 365
    
    4.生成服务端的密钥
    
    openssl genrsa -out server-key.pem 1024
    
    5.创建服务端证书请求
    
    openssl req -new -out server-req.csr -key server-key.pem
    
    6.自签署服务端证书
    
    openssl x509 -req -in server-req.csr -out server-cert.pem -signkey server-key.pem -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -days 365
    
    7. 创建客户端私钥
    
    openssl genrsa -out client-key.pem 1024
    
    8. 创建客户端证书签名
    
    openssl req -new -out client-req.csr -key client-key.pem
    
    9. 自签署客户端证书
    
    openssl x509 -req -in client-req.csr -out client-cert.pem -signkey client-key.pem -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -days 365


到此CA证书、server证书、client证书则全部完成。其中server证书和client证书都是使用ca证书生成做匹配

Note:证书的制作也可以通过语言实现，如go、java等！！！