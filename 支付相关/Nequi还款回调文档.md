# Nequi还款回调文档

## Verificación de firmado usando Message Digests and Digital Signatures

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

```js
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

```js
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

```js
signedHeaders = "content-type digest" // Valor parseado en el paso anterior // 上一步解析的值
appSecret = "ThisIsATest" // Este valor no debe estar expuesto y debe estar almacenado seguramente (No quemado en el código e ideal cifrado en BD) // 此值不得暴露，必须安全存储（未烧录在代码中，理想情况下在 BD 中加密）
headersNames = split(signedHeaders, " ") // 使用空格分隔获取需要请求头签名需要的请求头字段
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

```json
headerNames = ["content-type","digest"]
reqHeadersLowercase = {
    "content-type": "application/json",
    "digest": "SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY=",
    "signature": "keyId=\"TestApp01\",algorithm=\"hmac-sha384\",headers=\"content-type digest\"signature=\"B_lqFDp8gR7fSmZlWT79iLxenJoiBqsJushjnsnjd623HYLlDEHwJsi3PUKb0hA9OtJaw-\""
}
linesForSignature = ["content-type: application/json","digest: SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY="] 
textForSignature = "content-type: application/json\ndigest: SHA-256=MQyB7LscfTetjRZpW5TU63hq15m/b55MKoDIThyHXuY="
signature = "B_lqFDp8gR7fSmZlWT79iLxenJoiBqsJuyz4ukHYLlDEHwJsi3PUKb0hA9OtJaw-"
```

## Especificación de Webhook para reportar resultado de Pago por APIs

通过 API 报告支付结果的 Webhook 规范

En esta guía se encuentra la especificación para reportar el estado de los pagos hechos por APIs y asi evitar el mecanismo de sonda, la seguridad que debe tener este servicio es definida por Nequi y necesitará hacer uso del App ClientId entregado con las credenciales para consumir las APIs de Nequi y definir un secreto que compartirá con Nequi para asegurar las peticiones. Para ver más información sobre la seguridad que debe tener el servicio, revisar documentación "Verificación de firmado usando Message Digests and Digital Signatures".

在本指南中，你会发现通过API报告支付状态的规范，以避免使用轮询机制，这项服务必须具备的安全性由Nequi定义，你需要利用随证书交付的App ClientId来消费Nequi API，并定义一个你将与Nequi共享的密钥，以确保请求的安全性。关于服务必须具备的安全性的更多信息，请参见文件 "使用消息摘要和数字签名的签名验证"。

Entendiendo ya como se verifican las peticiones enviadas por Nequi, las especificaciones del servicio que debe ser implementado para recibir el resultado de un pago son las siguientes:

现在您已经知道了Nequi发送的请求是如何被验证的了，为接收支付结果而必须实现的服务规格如下：

- Debe ser un servicio REST usando HTTPS y TLS1.2 implementando el método POST.
- Cuando el proceso de verificación falle se debe retornar un error con código HTTP 401.
- Su servicio debe recibir la notificación y hacer el procesamiento hacia sus sistemas de forma asíncrona, para no dejar abierta la conexión de nuestro cliente HTTP hacia sus servidores durante mucho tiempo.
- Nuestro sistema esperará máximo 10 segundos por la respuesta de su servicio, y reintentara durante 6 veces cada 5 minutos notificar el resultado en caso de timeout o error HTTP 500, por esto es importante que garantice la disponibilidad y escalabilidad del servicio o webhook que debe implementar. Es opcional configurar el reverso de la transacción cuando no se puede confirmar un pago por indisponibilidad en su servicio.
- El uso del servicio para consultar el estado de un pago sigue disponible y solo debe ser usado en caso que nunca reciba la notificación o que haya tenido indisponibilidad en su webhook por más de 30 minutos.

- 它必须是一个使用HTTPS和TLS1.2协议使用post方法实现的RESTful服务
- 当验证失败，服务应该返回一个401的状态码
- 你的系统必须在接收到通知后异步的处理请求，从而避免链接持续太长时间
- Nequi系统最多等待10s，如果没有响应就超时了。如果超时或者响应码是500，系统会发起重试。每5分钟会重试一次，最多重试6次。
- 接入webhook后，订单状态的主动查询服务仍是可用的。不过，一般只需要在未收到回调，或者webhook已经超过30分钟无效时使用。

En el caso del webhook para notificar el resultado de un pago, recibirá en su servicio usando el método POST el siguiente payload:

在使用通知支付结果的webhook后，你将在你的服务中使用POST方法收到以下有效载荷：

```json
{
    "commerceCode": //Código del comercio interno de nequi：Nequi 内部贸易代码
    "code"://Campo code enviado en el pago, normalmente representa el identificador de la subsidiaria：支付中发送的code字段，一般代表子公司的标识
    "value": //Valor del pago：支付的金额
    "phoneNumber": //Número de celular del pagador：付款人手机号码
    "messageId": //Identificador único de la transacción：交易的唯一标识符
    "transactionId": //Identificador del pago：付款标识符
    "region": //Region del pago P001 (Panama) o C001 (Colombia)：支付地区 P001（巴拿马）或 C001（哥伦比亚）
    "receivedAt": // Fecha en formato JSON de cuando se realizo el pago：付款时的 JSON 格式日期
    "paymentStatus": // Estado del pago, SUCCESS (cuando se realiza el pago de forma exitosa) o CANCELED/REFUSED (si el pago fue cancelado o rechazado)}：付款状态，SUCCESS（付款成功时）或 CANCELED/REFUSED（付款被取消或拒绝）}
```

Este payload trae toda la información de la transacción con que puede verificar el correcto pago por parte del usuario.

这个有效载荷带来了所有的交易信息，你可以用它来验证用户的正确支付。

Para poner estos webhooks en funcionamiento es necesario nos envie los endpoints de su servicio construidos bajo esta especificación para un ambiente de pruebas y producción, y enviarnos el secreto (appSecret) que definan para su servicio según el ambiente. Para el keyId requerido para el calculo de la firma use el App ClientId que le entregamos para consumir nuestras APIs de Pagos según el ambiente.

为了让这些webhooks启动和运行，你需要把你在此规范下构建的服务的端点发送给我们，用于测试和生产环境，并把你根据环境为你的服务定义的秘密（appSecret）发送给我们。对于签名计算所需的keyId，使用我们给你的App ClientId，以根据环境消费我们的支付API。
