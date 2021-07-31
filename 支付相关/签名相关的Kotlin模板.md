# 签名相关的Kotlin模板

## 获取签名原串模板

```Kotlin
/**
 * @className: OriginalStringUtils
 * @description: 根据请求的json字段，生成对应的签名原串
 * @author: wfb
 * @date: 2021-05-11 17:03:10
 * @version: 1.0
 */
object OriginalStringUtils {

    private val opmCreateOrderSignFieldsOrder = arrayOf(
        "beneficiaryName",
        "beneficiaryUid",
        "beneficiaryBank",
        "beneficiaryAccount",
        "beneficiaryAccountType",
        "payerAccount",
        "numericalReference",
        "paymentDay",
        "paymentType",
        "concept",
        "amount"
    )

    private val opmLoanWebhookSignFieldsOrder = arrayOf(
        "id",
        "status",
        "detail"
    )

    private val opmRepayWebhookSignFieldsOrder = arrayOf(
        "beneficiaryName",
        "beneficiaryUid",
        "beneficiaryAccount",
        "beneficiaryBank",
        "beneficiaryAccountType",
        "payerName",
        "payerUid",
        "payerAccount",
        "payerBank",
        "payerAccountType",
        "amount",
        "concept",
        "trackingKey",
        "numericalReference"
    )

    enum class OpmSignEnum(val fieldsOrder: Array<String>) {
        // 枚举值的数组中写明了要求的签名字段和其顺序
        CREATE_ORDER(opmCreateOrderSignFieldsOrder),
        LOAN_WEBHOOK(opmLoanWebhookSignFieldsOrder),
        REPAY_WEBHOOK(opmRepayWebhookSignFieldsOrder)
    }

    /**
     * opm 放款生成sign
     */
    fun getOpmSignString(req: JSONObject, opmSignEnum: OpmSignEnum = OpmSignEnum.CREATE_ORDER): String {
        return opmSignEnum.fieldsOrder.map {
            when { // 在when中写第三方文档中要求的签名原串的转换逻辑，如金额要求两位小数
                req[it] == null -> ""
                it == "amount" -> DecimalFormat("0.00").format(req.getBigDecimal(it))
                it == "paymentDay" -> ZonedDateTime.ofInstant(Instant.ofEpochMilli(req.getLong(it)), ZoneId.of("Z"))
                    .format(DateTimeFormatter.ISO_LOCAL_DATE)
                else -> req[it]
            }
        }.joinToString(prefix = "||", postfix = "||", separator = "|") // 生成原串时各个字段的拼接方式
    }
}
```

调用方法：

```Kotlin
val originalStr = OriginalStringUtils.getOpmSignString(data, OriginalStringUtils.OpmSignEnum.LOAN_WEBHOOK)
// originalStr即为要生成的签名原串
// 下一步使用要求的签名方法对其进行签名
```

## RSA签名模板-使用[Signature]实现

使用`Signature`进行签名。`Signature`方法是一个上层方法，只允许使用密钥对原串进行先摘要后签名的处理（这区别于先调用摘要算法，再调用签名算法，因为内部多了一个对于摘要后的数据的封装）。

```Kotlin
/**
 * @className: FileRSAUtils
 * @description: 使用某个文件夹下的PKCS8的密钥进行RSA类的签名
 * @author: wfb
 * @date: 2021-05-11 17:03:10
 * @version: 1.0
 */
object FileRSAUtils {

    private val log = LoggerFactory.getLogger(this.javaClass)

    private val configDir = EnvConstants.userDir + "/config" // EnvConstants.userDir对应于getProperty("user.dir") ?: getenv("user.dir")

    // 获取存储该支付商对应的该马甲包在该环境下使用的私钥的文件全路径名
    private fun getPrivateKeyFileName(keyDir: String, appId: String, env: String): String {
        /*
        不同的支付商使用不同的密钥，由keyDir表示支付商目录
        不同的马甲包可能对应着不同的商户，由appId表示对应马甲包名
        测试环境和生常环境使用不同的密钥,由env表示对应环境
         */
        return "$configDir/$keyDir/$appId-$env.pkcs8"
    }

    // 将密钥从文件读入到内存中
    private fun getPrivateKeyStr(file: File): String {
        val lines = file.readLines()
        val sb = StringBuilder()
        for (i in 1 until lines.size - 1) sb.append(lines[i])
        return sb.toString()
    }

    private val cache = HashMap<String, String>()

    private fun getPrivateKeyStr(fileName: String): String {
        return cache.getOrPut(fileName) { getPrivateKeyStr(File(fileName)) }
    }

    private val keyFactory = KeyFactory.getInstance("RSA")

    private fun encryptByPrivateKey(keyDir: String, bytes: ByteArray, appId: String, algorithm: String): ByteArray {
        val fileName = getPrivateKeyFileName(keyDir, appId, EnvConstants.env)
        val privateKeyStr = getPrivateKeyStr(fileName)
        val privateKey = keyFactory.generatePrivate(PKCS8EncodedKeySpec(Base64.getDecoder().decode(privateKeyStr)))
        val signature = Signature.getInstance(algorithm)
        signature.initSign(privateKey)
        signature.update(bytes)
        return signature.sign()
    }

    /**
     * 该方法主要用在对特殊的消息摘要(如MD2、MD5、SHA256)进行签名，内部使用封装的[Signature]类
     * @param keyDir ：支付商密钥所在的目录名
     * @param originalString ：需要进行签名的字符串原串
     * @param appId ：马甲包名
     * @param algorithm ：采用的加密算法(`<digest>with<encryption>`),详见[https://docs.oracle.com/javase/8/docs/technotes/guides/security/StandardNames.html#Signature]
     * @param preHandle ：将字符串原串转化为字节数组使用的方法，默认为`str.toByteArray()`
     * @param postHandle ：将签名后的字节数组转化为字符串的方法，默认为`String()的构造方法`
     */
    fun sign(keyDir: String, originalString: String, appId: String, algorithm: String="SHA256withRSA",
             preHandle: (str: String)->ByteArray = {str: String -> str.toByteArray()},
             postHandle: (bytes: ByteArray)->String = ::String
    ): String {
        log.info("oriString=$originalString")
        try {
            val inputString = preHandle(originalString)
            val rsaBytes = encryptByPrivateKey(keyDir, inputString, appId, algorithm)
            val retString = postHandle(rsaBytes)
            log.info("retString=$retString")
            return retString
        } catch (e: Exception) {
            log.info("FileRSAUtils [$keyDir][$appId] sign exception: {}", e)
            throw BusinessException("$keyDir sign error")
        }
    }
}
```

将`PKCS1`密钥转化为`PKCS8`密钥的常用命令：

```shell
openssl pkcs8 -topk8 -inform PEM -in <appId>-<env>.pkcs1 -outform pem -nocrypt -out <appId>-<env>.pkcs8
```

拷贝某文件在当前目录下生成多个`<appId>-<env>.pkcs1`文件：

```shell
source_file="private_key"
app_id_list=(
  "appId1"
  "appId2"
  "appId3"
)
env="prod"
for app_id in ${app_id_list[@]}; do
  cp $source_file "$app_id-$env.pkcs1";
done
```

为当前文件夹下的所有`.pkcs1`后缀结尾的文件生成对应的`.pkcs8`后缀文件：

```shell
for file in $(ls | grep '^.*.pkcs1$'); do
  file_name=$(echo $file | cut -d . -f1);
  openssl pkcs8 -topk8 -inform PEM -in ${file_name}.pkcs1 -outform pem -nocrypt -out ${file_name}.pkcs8
done
```

使用方式：

```Kotlin
// 使用`SHA256withRSA`对原串加密，并使用`Base64`将字节数组转化为数组
val str = sign("opm", body, appId, postHandle = Base64.getEncoder()::encodeToString)
```
