# Nequi还款回调文档

Verificación de firmado usando Message Digests and Digital Signatures
使用消息摘要和数字签名进行签名验证

Este documento describe el proceso para la verificación de la autorización y firmado usando el método de Message Digests and Digital Signatures.
本文档描述了使用消息摘要和数字签名方法验证授权和签名的过程。

Nota:
注意：

Los métodos usados en los ejemplos de este manual son un pseudocódigo y no hacen parte de ningún lenguaje de programación, dependiendo del que se vaya a utilizar se deben usar los métodos apropiados para cumplir con el objetivo propuesto en este manual.
本手册示例中使用的方法是伪代码，不属于任何编程语言，根据要使用的语言，必须使用适当的方法来实现本手册中提出的目标。

Contenido

1. Request de ejemplo
2. Verificación del body
3. Transformación del header de firmado 
4. Verificación del Request Signature

内容简介

1. 请求示例
2. 校验消息体body
3. 从请求头中获取请求头签名规则
4. 校验请求头

Request de ejemplo:
请求示例

```json
{
    "headers:": {
        "Content-Type": "application/json",
        "Digest": "SHA-256=R2uaJxvz//7kwe6vNTcZ9KVDfM1N7MCpoXbf9rr3APk=",
        "Signature": "keyId=\"TestApp01\",algorithm=\"hmac-sha384\",headers=\"content-type digest\",signature=\"9WJc5wcu4sn1xDK5oyoZrF_V9VRHFIQkElphSYeqTKPiZTS1GzH6f3c Bt6gM1CR\""
    },
    "body": {
        "data": "test"
    }
}
```

Verificación del body:
校验消息体body：

Lo primero que se debe validar es que el hash del multidigest del body corresponda con el que se recibió en el header Digest. Para eso primero tomamos el json recibido en el body y lo parseamos a un string, luego se calcula el SHA256 y el digest en base64, así:
首先要验证的是正文的摘要是否和请求头中的摘要相同。为此，我们首先将正文中接收到的 json 解析为字符串，然后计算 SHA256 和 base64 中的摘要，如下所示：

```java
parsedBody = jsonToString(body) // 将请求体转为字符串
digester = hash(parsedBody, 'sha256') // 对其进行sha256散列
digest = "SHA-256=" + digester.digest('base64') // 对散列值进行base64编码
if(digest != headers['Digest']){ // 如果与请求头中的摘要不相符，则请求无效
    return 'Invalid Digest'
}
```

Transformación del header de firmado:
从请求头中获取请求头签名规则：

Se toma el header Signature, se separa por comas (",") y luego se debe crear un objeto con la información que está contenida en el Signature:
采用 Signature 标头，用逗号 (",") 分隔，然后必须使用 Signature 中包含的信息创建一个对象：

```java
// 将以逗号分隔存储在Signature请求头中的信息转化为程序的键值对。用于后续处理
parts = split(headers["Signature"], ",")
result = {}
for(part in parts){
    temp = split(part,"=")
    result[temp[0]] = slice(temp[1],1,-1)
}
```

Ejemplo de los valores de entrada y el resultado de salida de este paso:
此步骤的输入值和输出结果示例：

```java
// Signature recibido en los header
headers["Signature"] = "keyId=\"TestApp01\",algorithm=\"hmac-sha384\",headers=\"content-type digest\"signature=\"9WJc5wcu4sn1xDK5oyoZrF_V9VRHFIQkElphSYeqTKPiZTS1GzH6f3cTBt6gM1CR\""
// Objeto parseado
result = {
    "keyId": "TestApp01",
    "algorithm": "hmac-sha384",
    "headers": "content-type digest",
    "signature": "B_lqFDp8gR7fSmZlWT79iLxenJoiBqsJuyz4ukHYLlDEHwJsi3PUKb0hA9OtJaw-"
}
```

Verificación del Request Signature:
校验请求头：

Se crea un arreglo con las headers que fueron parseadas en el paso anterior, ademas se llevan a minúscula las llaves de las headers recibidas en el request, luego se crea el texto que debió ser firmado con base en las headers recibidas en el request, finalmente se calcula el hmac con hash SHA384, se convierte a base64 y se compara contra el valor recibido en el header del Signature.
使用在上一步中解析的标头创建一个排列，并将请求中接收到的标头的键设为小写，然后创建应该根据请求中收到的标头签名的文本，最后hmac 使用 SHA384 哈希计算，将其转换为 base64，并与签名标头中收到的值进行比较。

```java
signedHeaders = "content-type digest" // Valor parseado en el paso anterior // 上一步解析的值
appSecret = "ThisIsATest" // Este valor no debe estar expuesto y debe estar almacenado seguramente (No quemado en el código e ideal cifrado en BD) // 此值不得暴露，必须安全存储（未烧录在代码中，理想情况下在 BD 中加密）
headersName = split(signedHeaders, " ") // 使用空格分隔获取需要请求头签名需要的请求头字段
reqHeadersLowercase = keysToLowerCase(headers) // 将请求头的key都转化为小写
// 按照规则的规则排列请求头的键值对并生成源字符串
linesForSignature = []
for(headerName in headerNames){
    val = reqHeadersLowercase[headerName]
    push(linesForSignature, headerName + ": " + val) 
}
textForSignature = join(linesForSignature, "\n")
// 使用sha384对该源字符串进行签名，其签名结果的表示形式使用base64
digester = hmac(textForSignature, appSecret, "sha384")
base64hmac = digester.digest["base64"]
signature = base64url.fromBase64(base64hmac)
// 验证签名是否正确
if (signature != parsedSignature.signature) {
    return "Invalid Signature";
}
```

Ejemplos de los valores calculadores en los pasos intermedios:

```java
headerNames = ["content-type","digest"]
reqHeadersLowercase = {
    "content-type": "application/json",
    "digest": "SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY=",
    "signature": "keyId=\"TestApp01\",algorithm=\"hmac-sha384\",headers=\"content-type digest\"signature=\"B_lqFDp8gR7fSmZlWT79iLxenJoiBqsJushjnsnjd623HYLlDEHwJsi3PUKb0hA9OtJaw-\""
}
linesForSignature = ["content-type: application/json","digest: SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY="] textForSignature = "content-type: application/json\ndigest: SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY=" signature = "B_lqFDp8gR7fSmZlWT79iLxenJoiBqsJuyz4ukHYLlDEHwJsi3PUKb0hA9OtJaw-"
```
