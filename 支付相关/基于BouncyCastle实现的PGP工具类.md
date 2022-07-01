# 基于BouncyCastle实现的PGP工具类

> 注意BC实现的标准是：[RFC 4880](https://datatracker.ietf.org/doc/html/rfc4880#page-17)
> 其github地址为：[bcgit/bc-java](https://github.com/bcgit/bc-java)

## gpg命令行工具的常用命令

> 注意GPG命令行工具2.3.0版本默认会使用还在草案中的AEAD加密，该加密方式引入了新的[AEAD Encrypted Data Packet (Tag 20)](https://tools.ietf.org/id/draft-ietf-openpgp-rfc4880bis-06.html#rfc.section.5.16)，用这种方式加密的数据，无法用1.6版本的bcpg解密。1.47有人提issue了，不知道啥时候有人提PR，并merge。想要使用原来的MDC加密方式，需要添加额外的选项参数`--force-mdc --rfc2440`，即：`gpg --force-mdc --rfc2440 --decrypt [file name]`

基本使用可以看阮一峰的[GPG入门教程](https://www.ruanyifeng.com/blog/2013/07/gpg.html)

常用操作 | 对应命令
--- | ---
列出密钥 | `gpg --list-keys --keyid-format long --fingerprint`
列出公钥 | `gpg --list-public-keys --keyid-format long --fingerprint`
列出私钥 | `gpg --list-secret-keys --keyid-format long --fingerprint`
删除公钥 | `gpg --delete-key [user id]`
删除公私钥 | `gpg --delete-secret-and-public-key [user id]`
输出公钥（--armor表示ASCII码显示） | `gpg --armor --output [output file name] --export [user id]`
输出私钥（--armor表示ASCII码显示） | `gpg --armor --output [output file name] --export-secret-keys [user id]`
引入密钥 | `gpg --import [file name]`
解密文件 | `gpg --decrypt [file name]`
加密文件 | `gpg --recipient [user id / key id] --output [output file name] --encrypt [file name]`
生成密钥 | `gpg --full-generate-key`
生成混合签名文件 | `gpg -u [user id / key id] --output [output file name] -s [file name]`
生产clearText签名文件 | `gpg -u [user id / key id] --output [output file name] -a --clearsign [file name]`
验证签名| `gpg --output [output file name] --verify [file name]`

## 所需maven依赖

```XML
<!-- PGP相关依赖 -->
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcpg-jdk16</artifactId>
    <version>1.46</version>
</dependency>
<!-- BC的Provider -->
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk16</artifactId>
    <version>1.46</version>
</dependency>
```

## 工具类

```Kotlin
import com.mib.bank.service.common.exception.BankException
import org.bouncycastle.bcpg.*
import org.bouncycastle.jce.provider.BouncyCastleProvider
import org.bouncycastle.openpgp.*
import java.io.ByteArrayOutputStream
import java.io.InputStream
import java.io.OutputStream
import java.math.BigInteger
import java.security.SecureRandom
import java.security.Security
import java.time.Instant
import java.util.*

/**
 * <p>
 *  PGPUtils
 * </p>
 * @author wfb
 * @version 1.0
 * @since 2022-04-08 10:52:19
 */
object PGPUtils {

    init {
        Security.addProvider(BouncyCastleProvider())
    }

    data class PublicKeyHolder(private val keyStream: InputStream, private val keyId: String): KeyHolder {
        private val publicKey = getPublicKey(keyStream, keyId)
        override fun obtainPublicKey(): PGPPublicKey = publicKey
        override fun obtainPrivateKey(): PGPPrivateKey = throw BankException("public key holder can not create the private key")
    }

    data class SecretKeyHolder(val keyStream: InputStream, val keyId: String, val pass: String): KeyHolder {
        private val secretKey = getSecretKey(keyStream, keyId)
        override fun obtainPublicKey(): PGPPublicKey = secretKey.publicKey
        override fun obtainPrivateKey(): PGPPrivateKey = secretKey.extractPrivateKey(pass.toCharArray(), "BC")
    }

    interface KeyHolder {
        fun obtainPublicKey(): PGPPublicKey
        fun obtainPrivateKey(): PGPPrivateKey
        
        /**
         * 将src流中的内容使用指定的公钥加密到dst流中
         * 可以配置是否对原始数据压缩，是否转化为asc的文本格式，是否添加完整性检查
         */
        fun encrypt(
            src: InputStream, dst: OutputStream,
            fileName: String= "", modifyTime: Long = Instant.now().toEpochMilli(),
            compressAlgorithmEnum: CompressAlgorithmEnum = CompressAlgorithmEnum.UNCOMPRESSED,
            symmetricKeyAlgorithmEnum: SymmetricKeyAlgorithmEnum = SymmetricKeyAlgorithmEnum.AES_128,
            armor: Boolean = false, withIntegrityCheck: Boolean = false
        ) = encrypt(src, dst, obtainPublicKey(), fileName, modifyTime, compressAlgorithmEnum, symmetricKeyAlgorithmEnum, armor, withIntegrityCheck)

        /**
         * 将src流使用指定的私钥解密到dst流中
         * 会自动判断使用的算法，是否需要解密，是否要进行完整性检查
         */
        fun decrypt(src: InputStream, dst: OutputStream) = decrypt(src, dst, obtainPrivateKey())

        /**
         * 将签名后的内容写入dst流
         */
        fun sign(
            src: InputStream, dst: OutputStream,
            fileName: String= "", modifyTime: Long = Instant.now().toEpochMilli(),
            compressAlgorithmEnum: CompressAlgorithmEnum = CompressAlgorithmEnum.UNCOMPRESSED,
            hashAlgorithmEnum: HashAlgorithmEnum = HashAlgorithmEnum.SHA256,
            armor: Boolean = false // 是否使用asc
        ) = sign(src, dst, obtainPrivateKey(), obtainPublicKey(), fileName, modifyTime, compressAlgorithmEnum, hashAlgorithmEnum, armor)

        /**
         * 对输入的签名流执行校验
         */
        fun verify(src: InputStream): ByteArray = verify(src, obtainPublicKey())
        
        /**
         * 对输入的签名流执行校验
         */
        fun forceVerify(src: InputStream, dst: OutputStream): Boolean = runCatching { dst.write(verify(src)) }.isSuccess

        /**
         * 生成cleartext格式的签名，然后写入dst流
         */
        fun clearTextSign(
            src: InputStream, dst: OutputStream,
            hashAlgorithmEnum: HashAlgorithmEnum = HashAlgorithmEnum.SHA256,
        ) = clearTextSign(src, dst, obtainPrivateKey(), obtainPublicKey(), hashAlgorithmEnum)
        
        /**
         * 对输入的文件流和签名流执行校验
         */
        fun clearTextVerify(src: InputStream): ByteArray = clearTextVerify(src, obtainPublicKey())

        /**
         * 对输入的签名流执行校验
         */
        fun forceClearTextVerify(src: InputStream, dst: OutputStream): Boolean = runCatching { dst.write(clearTextVerify(src)) }.isSuccess
    }
    
    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的密钥
     * 这里的keyId需要填写16进制的形式
     */
    private fun getSecretKey(keyStream: InputStream, keyId: String): PGPSecretKey {
        val keyIdL = BigInteger(keyId, 16).toLong()
        return getSecretKey(keyStream, keyIdL)
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的密钥
     * 注意keyId对应的16位16进制数最高位是1时，即为负数时，java没法单纯的用0x来将16进制数转为Long，这时需要自己做一下转换
     */
    private fun getSecretKey(keyStream: InputStream, keyId: Long): PGPSecretKey {
        PGPUtil.getDecoderStream(keyStream).use { stream ->
            val keyRingCollection = PGPSecretKeyRingCollection(stream)
            return keyRingCollection.getSecretKey(keyId)
                ?: throw PGPException("Don't find the keyId[$keyId] in this keyRingCollection")
        }
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的公钥
     * 这里的keyId需要填写16进制的形式
     */
    private fun getPublicKey(keyStream: InputStream, keyId: String): PGPPublicKey {
        val keyIdL = BigInteger(keyId, 16).toLong()
        return getPublicKey(keyStream, keyIdL)
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的公钥
     * 注意keyId对应的16位16进制数最高位是1时，即为负数时，java没法单纯的用0x来将16进制数转为Long，这时需要自己做一下转换
     */
    private fun getPublicKey(keyStream: InputStream, keyId: Long): PGPPublicKey {
        PGPUtil.getDecoderStream(keyStream).use { stream ->
            val keyRingCollection = PGPPublicKeyRingCollection(stream)
            return keyRingCollection.getPublicKey(keyId)
                ?: throw PGPException("Don't find the keyId[$keyId] in this keyRingCollection")
        }
    }

    /**
     * 将src流中的内容使用指定的公钥加密到dst流中
     * 可以配置是否对原始数据压缩，是否转化为asc的文本格式，是否添加完整性检查
     */
    private fun encrypt(
        src: InputStream, dst: OutputStream,
        publicKey: PGPPublicKey,
        fileName: String= "", modifyTime: Long = Instant.now().toEpochMilli(),
        compressAlgorithmEnum: CompressAlgorithmEnum = CompressAlgorithmEnum.UNCOMPRESSED,
        symmetricKeyAlgorithmEnum: SymmetricKeyAlgorithmEnum = SymmetricKeyAlgorithmEnum.AES_128,
        armor: Boolean = false, withIntegrityCheck: Boolean = false
    ) {
        val outStream = if (armor) ArmoredOutputStream(dst) else dst
        
        val bytes = src.readBytes() // 输入的数据
        // 对输入数据进行处理，并压缩
        val pgpBytes = ByteArrayOutputStream().use { baos ->
            PGPCompressedDataGenerator(getCompressionAlgorithmTags(compressAlgorithmEnum)).open(baos).use { compressStream ->
                PGPLiteralDataGenerator().open(compressStream, PGPLiteralData.BINARY, fileName, bytes.size.toLong(), Date(modifyTime)).use {
                    it.write(bytes)
                }
            }
            baos.toByteArray()
        }
        
        // 对压缩后的数据进行加密并写入
        val encryptedDataGenerator = PGPEncryptedDataGenerator(
            getSymmetricKeyAlgorithmTags(symmetricKeyAlgorithmEnum),
            withIntegrityCheck,
            SecureRandom(),
            "BC"
        )
        encryptedDataGenerator.addMethod(publicKey)
        // 将数据写入
        encryptedDataGenerator.open(outStream, pgpBytes.size.toLong()).use {
            it.write(pgpBytes)
        }
        
        if (armor) outStream.close()
    }

    /**
     * 将src流使用指定的私钥解密到dst流中
     * 会自动判断使用的算法，是否需要解密，是否要进行完整性检查
     */
    private fun decrypt(
        src: InputStream, dst: OutputStream,
        privateKey: PGPPrivateKey
    ) {
        // 获取拆分后的包
        val pgpObjectFactory = PGPObjectFactory(PGPUtil.getDecoderStream(src))
        val nextObject = pgpObjectFactory.nextObject()
        // 如果nextObject就是PGPEncryptedDataList则直接赋值，否则它是个PGP marker packet，直接取下一个数据就是了
        val encryptedDataList = if (nextObject is PGPEncryptedDataList) nextObject
        else pgpObjectFactory.nextObject() as PGPEncryptedDataList
        // 查找需要用该私钥解密的encryptedData，对其解密
        val ped = encryptedDataList.encryptedDataObjects.asSequence().map {
            it as PGPPublicKeyEncryptedData
        }.firstOrNull {
             it.keyID == privateKey.keyID
        } ?: throw PGPException("There isn't an encryptedData which need this privateKey[${privateKey.keyID}] to decrypt")
        // 创建解压缩工厂
        var message = PGPObjectFactory(ped.getDataStream(privateKey, "BC")).nextObject()
        if (message is PGPCompressedData) {
            message = PGPObjectFactory(message.dataStream).nextObject()
        }
        when (message) {
            is PGPLiteralData -> {
                message.inputStream.copyTo(dst)
            }
            is PGPOnePassSignatureList -> {
                throw PGPException("Encrypted message contains a signed message - not literal data.")
            }
            else -> {
                throw PGPException("Message is not a simple encrypted file - type unknown.")
            }
        }
        if (ped.isIntegrityProtected) {
            if (!ped.verify()) throw PGPException("Message failed integrity check")
        }
    }

    /**
     * 将签名后的内容写入dst流
     */
    private fun sign(
        src: InputStream, dst: OutputStream,
        privateKey: PGPPrivateKey, publicKey: PGPPublicKey,
        fileName: String= "", modifyTime: Long = Instant.now().toEpochMilli(),
        compressAlgorithmEnum: CompressAlgorithmEnum = CompressAlgorithmEnum.UNCOMPRESSED,
        hashAlgorithmEnum: HashAlgorithmEnum = HashAlgorithmEnum.SHA256,
        armor: Boolean = false // 是否使用asc
    ) {
        // 获取待签名的数据
        val bytes = src.readBytes()
        // 创建签名生成器，并初始化
        val sGen = PGPSignatureGenerator(publicKey.algorithm, getHashAlgorithmTags(hashAlgorithmEnum), "BC")
        sGen.initSign(PGPSignature.BINARY_DOCUMENT, privateKey)

        // 在签名生成器中增加hash标识子包，标识使用的签名公钥对的userID
        val userIDs = publicKey.userIDs
        if (userIDs.hasNext()) {
            sGen.setHashedSubpackets(PGPSignatureSubpacketGenerator().apply {
                this.setSignerUserID(false, userIDs.next() as String)
            }.generate())
        }
        
        // 生成写入流
        val armorOut = if (armor) ArmoredOutputStream(dst) else dst
        BCPGOutputStream(PGPCompressedDataGenerator(getCompressionAlgorithmTags(compressAlgorithmEnum)).open(armorOut)).use { out ->

            // 写入OnePassVersion标识
            sGen.generateOnePassVersion(false).encode(out)
            
            // 使用文字数据包装流写入原始内容
            PGPLiteralDataGenerator().open(out, PGPLiteralData.BINARY, fileName, bytes.size.toLong(), Date(modifyTime)).use {
                it.write(bytes)
            }

            // 使用签名生成器生成并写入签名流
            sGen.update(bytes)
            sGen.generate().encode(out)
        }
        if (armor) armorOut.close()
    }

    /**
     * 对输入的签名流执行校验
     */
    private fun verify(
        src: InputStream, publicKey: PGPPublicKey
    ): ByteArray {
        // 拆包获取数据
        val compressedData = PGPObjectFactory(PGPUtil.getDecoderStream(src)).nextObject()
        if (compressedData !is PGPCompressedData) throw BankException("the first object in sign file must be PGPCompressedData")
        val objectFactory = PGPObjectFactory(compressedData.dataStream)
        
        val onePassSignatureList = objectFactory.nextObject()
        if (onePassSignatureList !is PGPOnePassSignatureList) throw BankException("the first object in compressedData must be PGPOnePassSignatureList")
        if (onePassSignatureList.isEmpty) throw BankException("the onePassSignatureList must have at least one item")
        val onePassSignature = onePassSignatureList.get(0)
        if (onePassSignature !is PGPOnePassSignature) throw BankException("the item of the onePassSignatureList must be PGPOnePassSignature")
        
        val literalData = objectFactory.nextObject()
        if (literalData !is PGPLiteralData) throw BankException("the second object in compressedData must be PGPLiteralData")
        // 获取原始内容
        val bytes = literalData.inputStream.readBytes()
        
        val signatureList = objectFactory.nextObject()
        if (signatureList !is PGPSignatureList) throw BankException("the third object in compressedData must be PGPSignatureList")
        if (signatureList.isEmpty) throw BankException("the signatureList must have at least one item")
        val signature = signatureList.get(0)

        // 初始化验证
        onePassSignature.initVerify(publicKey, "BC")
        // update内容
        onePassSignature.update(bytes)
        // 判断签名是否可以验证
        if (!onePassSignature.verify(signature)) throw BankException("verify failed")
        // 返回原始内容
        return bytes
    }
    
    /**
     * 生成cleartext格式的签名，然后写入dst流
     */
    private fun clearTextSign(
        src: InputStream, dst: OutputStream,
        privateKey: PGPPrivateKey, publicKey: PGPPublicKey,
        hashAlgorithmEnum: HashAlgorithmEnum = HashAlgorithmEnum.SHA256,
    ) {
        // 获取待签名的数据
        val bytes0 = src.readBytes()
        val isCR = bytes0.contains(iCR)
        val isLF = bytes0.contains(iLF)
        val bytes = eatTheLastCRLF(bytes0)

        // 创建签名生成器，并初始化
        val sGen = PGPSignatureGenerator(publicKey.algorithm, getHashAlgorithmTags(hashAlgorithmEnum), "BC")
        sGen.initSign(PGPSignature.CANONICAL_TEXT_DOCUMENT, privateKey)

        // 在签名生成器中增加hash标识子包，标识使用的签名公钥对的userID
        val userIDs = publicKey.userIDs
        if (userIDs.hasNext()) {
            sGen.setHashedSubpackets(PGPSignatureSubpacketGenerator().apply {
                this.setSignerUserID(false, userIDs.next() as String)
            }.generate())
        }
        
        // 进行数据写入
        ArmoredOutputStream(dst).use { armorOut -> 
            // 写入BEGIN PGP SIGNED MESSAGE标头和Hash算法声明
            armorOut.beginClearText(getHashAlgorithmTags(hashAlgorithmEnum))
            
            // 写入原始内容
            armorOut.write(bytes)
            
            // 和endClearText之间要隔一行，但我不清楚要用回车还是用换行还是用回车换行，所以就看文件里面有啥用啥了，都没有的话就全用一下
            if (isCR || !isLF) armorOut.write(iCR.toInt())
            if (isLF || !isCR) armorOut.write(iLF.toInt())
            
            armorOut.endClearText()
            
            BCPGOutputStream(armorOut).use { out ->
                // 使用签名生成器生成并写入签名流
                sGen.update(bytes)
                sGen.generate().encode(out)
            }
        }
    }

    /**
     * 对输入的文件流和签名流执行校验
     */
    private fun clearTextVerify(
        src: InputStream, publicKey: PGPPublicKey
    ): ByteArray {
        val armorIn = ArmoredInputStream(src)
        
        // 首先读取clearText
        val bytes0 = ByteArrayOutputStream().use { baos ->
            if (!armorIn.isEndOfStream && armorIn.isClearText) {
                while (!armorIn.isEndOfStream) {
                    val byte = armorIn.read()
                    if (!armorIn.isClearText) break
                    baos.write(byte)
                }
            }
            baos.toByteArray()
        }
        
        // 吃掉最后的CRLF，用来进行签名验证
        val bytes = eatTheLastCRLF(bytes0)

        // 构建signature
        val factory = PGPObjectFactory(armorIn)
        val signatureList = factory.nextObject()
        if (signatureList !is PGPSignatureList) throw BankException("the verify file must have the signatureList")
        if (signatureList.isEmpty) throw BankException("the signatureList part of verify file must have at least one item")
        val signature = signatureList.get(0)
        // signature初始化并进行签名验证
        signature.initVerify(publicKey, "BC")
        signature.update(bytes)
        if (!signature.verify()) throw BankException("verify failed")
        return bytes0
    }
    
    private const val iCR = '\r'.code.toByte()
    private const val iLF = '\n'.code.toByte()
    
    private fun eatTheLastCRLF(bytes0: ByteArray): ByteArray {
        // 将最后的 回车 或/和 换行 吃掉，见：https://datatracker.ietf.org/doc/html/rfc4880#section-7.1
        return when {
            bytes0.isEmpty() -> bytes0
            bytes0.size == 1 -> when {
                bytes0.last() == iCR || bytes0.last() == iLF -> ByteArray(0)
                else -> bytes0
            }
            bytes0[bytes0.size - 2] == iCR && bytes0[bytes0.size - 1] == iLF -> bytes0.dropLast(2).toByteArray()
            bytes0.last() == iCR || bytes0.last() == iLF -> bytes0.dropLast(1).toByteArray()
            else -> bytes0
        }
    }
    
    enum class CompressAlgorithmEnum {
        UNCOMPRESSED,
        ZIP,
        ZLIB,
        BZIP2;
    }

    private fun getCompressionAlgorithmTags(enum: CompressAlgorithmEnum): Int {
        return when(enum) {
            CompressAlgorithmEnum.UNCOMPRESSED -> CompressionAlgorithmTags.ZIP
            CompressAlgorithmEnum.ZIP -> CompressionAlgorithmTags.ZIP
            CompressAlgorithmEnum.ZLIB -> CompressionAlgorithmTags.ZLIB
            CompressAlgorithmEnum.BZIP2 -> CompressionAlgorithmTags.BZIP2
        }
    }

    enum class HashAlgorithmEnum {
        MD5, // MD5
        SHA1, // SHA-1
        RIPEMD160, // RIPE-MD/160
        DOUBLE_SHA, // Reserved for double-width SHA (experimental)
        MD2, // MD2
        TIGER_192, // Reserved for TIGER/192
        HAVAL_5_160, // Reserved for HAVAL (5 pass, 160-bit)
        SHA256, // SHA-256
        SHA384, // SHA-384
        SHA512, // SHA-512
        SHA224 // SHA-224
    }
    
    private fun getHashAlgorithmTags(enum: HashAlgorithmEnum): Int {
        return when(enum) {
            HashAlgorithmEnum.MD5 -> HashAlgorithmTags.MD5
            HashAlgorithmEnum.SHA1 -> HashAlgorithmTags.SHA1
            HashAlgorithmEnum.RIPEMD160 -> HashAlgorithmTags.RIPEMD160
            HashAlgorithmEnum.DOUBLE_SHA -> HashAlgorithmTags.DOUBLE_SHA
            HashAlgorithmEnum.MD2 -> HashAlgorithmTags.MD2
            HashAlgorithmEnum.TIGER_192 -> HashAlgorithmTags.TIGER_192
            HashAlgorithmEnum.HAVAL_5_160 -> HashAlgorithmTags.HAVAL_5_160
            HashAlgorithmEnum.SHA256 -> HashAlgorithmTags.SHA256
            HashAlgorithmEnum.SHA384 -> HashAlgorithmTags.SHA384
            HashAlgorithmEnum.SHA512 -> HashAlgorithmTags.SHA512
            HashAlgorithmEnum.SHA224 -> HashAlgorithmTags.SHA224
        }
    }
    
    enum class SymmetricKeyAlgorithmEnum {
        NULL,
        IDEA,
        TRIPLE_DES,
        CAST5,
        BLOWFISH,
        SAFER,
        DES,
        AES_128,
        AES_192,
        AES_256,
        TWOFISH;
    }

    private fun getSymmetricKeyAlgorithmTags(enum: SymmetricKeyAlgorithmEnum): Int {
        return when(enum) {
            SymmetricKeyAlgorithmEnum.NULL -> SymmetricKeyAlgorithmTags.NULL
            SymmetricKeyAlgorithmEnum.IDEA -> SymmetricKeyAlgorithmTags.IDEA
            SymmetricKeyAlgorithmEnum.TRIPLE_DES -> SymmetricKeyAlgorithmTags.TRIPLE_DES
            SymmetricKeyAlgorithmEnum.CAST5 -> SymmetricKeyAlgorithmTags.CAST5
            SymmetricKeyAlgorithmEnum.BLOWFISH -> SymmetricKeyAlgorithmTags.BLOWFISH
            SymmetricKeyAlgorithmEnum.SAFER -> SymmetricKeyAlgorithmTags.SAFER
            SymmetricKeyAlgorithmEnum.DES -> SymmetricKeyAlgorithmTags.DES
            SymmetricKeyAlgorithmEnum.AES_128 -> SymmetricKeyAlgorithmTags.AES_128
            SymmetricKeyAlgorithmEnum.AES_192 -> SymmetricKeyAlgorithmTags.AES_192
            SymmetricKeyAlgorithmEnum.AES_256 -> SymmetricKeyAlgorithmTags.AES_256
            SymmetricKeyAlgorithmEnum.TWOFISH -> SymmetricKeyAlgorithmTags.TWOFISH
        }
    }
}
```

使用方式可以参考以下单元测试：

```kotlin
import org.bouncycastle.util.Arrays
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable
import java.nio.file.Paths
import kotlin.io.path.inputStream
import kotlin.io.path.outputStream

/**
 * <p>
 *  PGPUtilsTest
 * </p>
 * @author wfb
 * @version 1.0
 * @since 2022-06-29 18:27:19
 */
@EnabledIfEnvironmentVariable(named = "USER", matches = "wfb", disabledReason = "这些测试只能bin本地跑，因为强依赖于bin本地的公私钥文件和相关的目录和原始文件及终端安装的gpg命令")
@Suppress("SameParameterValue") // IDEA飘黄太烦人了，关闭警告吧！！！
class PGPUtilsTest {

    private val srcPath = "/Users/bin/MIB/测试使用/pgp/test.txt"
    
    private val keyId = "2D244B72A87B7DD2"
    private val secretKeyHolder = PGPUtils.SecretKeyHolder(Paths.get("/Users/bin/MIB/测试使用/pgp/private-key.txt").inputStream(), keyId, "123456")
    private val publicKeyHolder = PGPUtils.PublicKeyHolder(Paths.get("/Users/bin/MIB/测试使用/pgp/public-key.txt").inputStream(), keyId)
    private val encryptPath = "/Users/bin/MIB/测试使用/pgp/test-enc.asc"
    private val decryptPath = "/Users/bin/MIB/测试使用/pgp/test-dec.txt"
    
    private val keyId1 = "BF999AD71502F790"
    private val secretKeyHolder1 = PGPUtils.SecretKeyHolder(Paths.get("/Users/bin/MIB/测试使用/pgp/private-key.txt").inputStream(), keyId1, "123456")
    private val publicKeyHolder1 = PGPUtils.PublicKeyHolder(Paths.get("/Users/bin/MIB/测试使用/pgp/public-key.txt").inputStream(), keyId1)
    private val signPath = "/Users/bin/MIB/测试使用/pgp/test-sign.asc"
    private val verifyPath = "/Users/bin/MIB/测试使用/pgp/test-verify.txt"
    private val signPath2 = "/Users/bin/MIB/测试使用/pgp/test-sign2.asc"
    private val verifyPath2 = "/Users/bin/MIB/测试使用/pgp/test-verify2.txt"
    
    // ------------------------------------加密------------------------------------
    @Test
    fun testEncryptAndDecrypt() {
        encrypt(srcPath, encryptPath)
        decrypt(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }

    @Test
    fun testEncryptAndDecryptArmor() {
        encryptArmor(srcPath, encryptPath)
        decrypt(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }
    
    @Test
    fun testEncryptAndDecryptByTerminal() {
        encrypt(srcPath, encryptPath)
        decryptByTerminal(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }
    
    @Test 
    fun testEncryptByTerminalAndDecrypt() {
        encryptByTerminal(srcPath, encryptPath)
        decrypt(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }

    @Test
    fun testEncryptByTerminalAndDecryptArmor() {
        encryptByTerminalArmor(srcPath, encryptPath)
        decrypt(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }
    
    @Test
    fun testEncryptByTerminalAndDecryptByTerminal() {
        encryptByTerminal(srcPath, encryptPath)
        decryptByTerminal(encryptPath, decryptPath)
        assert(srcPath contentEquals decryptPath)
    }

    // ------------------------------------签名------------------------------------
    
    @Test
    fun testSignAndVerify() {
        sign(srcPath, signPath)
        verify(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }
    
    @Test
    fun testSignByTerminalAndVerify() {
        signByTerminal(srcPath, signPath)
        verify(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }

    @Test
    fun testSignAndVerifyByTerminal() {
        sign(srcPath, signPath)
        verifyByTerminal(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }

    @Test
    fun testSignAndVerifyArmor() {
        signArmor(srcPath, signPath)
        verify(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }

    @Test
    fun testSignByTerminalAndVerifyArmor() {
        signByTerminalArmor(srcPath, signPath)
        verify(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }

    @Test
    fun testSignAndVerifyByTerminalArmor() {
        signArmor(srcPath, signPath)
        verifyByTerminal(signPath, verifyPath)
        assert(srcPath contentEquals verifyPath)
    }

    // ------------------------------------clear text签名------------------------------------
    @Test
    fun testClearTextSignAndVerify() {
        clearTextSign(srcPath, signPath2)
        clearTextVerify(signPath2, verifyPath2)
        assert(srcPath contentEquals verifyPath2)
    }

    @Test
    fun testClearTextSignByTerminalAndVerify() {
        clearTextSignByTerminal(srcPath, signPath2)
        clearTextVerify(signPath2, verifyPath2)
        assert(srcPath contentEquals verifyPath2)
    }

    @Test
    fun testClearTextSignAndVerifyByTerminal() {
        clearTextSign(srcPath, signPath2)
        clearTextVerifyByTerminal(signPath2, verifyPath2)
        assert(srcPath contentEquals verifyPath2)
    }
    
    private infix fun String.contentEquals(dstPath: String): Boolean {
        Paths.get(this).inputStream().use { src1->
            Paths.get(dstPath).inputStream().use { src2->
                val bytes1 = src1.readBytes()
                val bytes2 = src2.readBytes()
                return Arrays.areEqual(bytes1, bytes2)
            }
        }
    }
    
    private fun encrypt(srcPath: String, encryptPath: String) {
        removeFile(encryptPath)
        Paths.get(srcPath).inputStream().use { src->
            Paths.get(encryptPath).outputStream().use { dst ->
                publicKeyHolder.encrypt(src, dst, "test.txt", withIntegrityCheck = true)
            }
        }
    }

    private fun encryptArmor(srcPath: String, encryptPath: String) {
        removeFile(encryptPath)
        Paths.get(srcPath).inputStream().use { src->
            Paths.get(encryptPath).outputStream().use { dst ->
                publicKeyHolder.encrypt(src, dst, "test.txt", armor = true, withIntegrityCheck = true)
            }
        }
    }
    
    private fun decrypt(encryptPath: String, decryptPath: String) {
        removeFile(decryptPath)
        Paths.get(encryptPath).inputStream().use { src->
            Paths.get(decryptPath).outputStream().use { dst->
                secretKeyHolder.decrypt(src, dst)
            }
        }
    }
    
    private fun encryptByTerminal(srcPath: String, encryptPath: String) {
        removeFile(encryptPath)
        // --force-mdc --rfc2440选项是为了禁用到AEAD，因为AEAD加密还在草案中，它会产生Packet Tag为20的包，这不属于RFC 4880定义的，BC的1。46版本没有实现它
        executeByTerminal("gpg -r $keyId --output $encryptPath --force-mdc --rfc2440 --encrypt $srcPath")
    }

    private fun encryptByTerminalArmor(srcPath: String, encryptPath: String) {
        removeFile(encryptPath)
        // --force-mdc --rfc2440选项是为了禁用到AEAD，因为AEAD加密还在草案中，它会产生Packet Tag为20的包，这不属于RFC 4880定义的，BC的1。46版本没有实现它
        executeByTerminal("gpg -r $keyId --armor --output $encryptPath --force-mdc --rfc2440 --encrypt $srcPath")
    }
    
    private fun decryptByTerminal(encryptPath: String, decryptPath: String) {
        removeFile(decryptPath)
        executeByTerminal("gpg --pinentry-mode loopback --passphrase 123456 --output $decryptPath --decrypt $encryptPath")
    }
    
    private fun sign(srcPath: String, signPath: String) {
        removeFile(signPath)
        Paths.get(srcPath).inputStream().use { src->
            Paths.get(signPath).outputStream().use { dst ->
                secretKeyHolder1.sign(src, dst, "test.txt")
            }
        }
    }
    
    private fun signArmor(srcPath: String, signPath: String) {
        removeFile(signPath)
        Paths.get(srcPath).inputStream().use { src->
            Paths.get(signPath).outputStream().use { dst ->
                secretKeyHolder1.sign(src, dst, "test.txt", armor = true)
            }
        }
    }
    
    private fun verify(signPath: String, verifyPath: String) {
        removeFile(verifyPath)
        Paths.get(signPath).inputStream().use { src->
            Paths.get(verifyPath).outputStream().use { dst ->
                assert(publicKeyHolder1.forceVerify(src, dst))
            }
        }
    }
    
    private fun clearTextSign(srcPath: String, signPath: String) {
        removeFile(signPath)
        Paths.get(srcPath).inputStream().use { src->
            Paths.get(signPath).outputStream().use { dst ->
                secretKeyHolder1.clearTextSign(src, dst)
            }
        }
    }
    
    private fun clearTextVerify(signPath: String, verifyPath: String) {
        removeFile(verifyPath)
        Paths.get(signPath).inputStream().use { src->
            Paths.get(verifyPath).outputStream().use { dst ->
                assert(publicKeyHolder1.forceClearTextVerify(src, dst))
            }
        }
    }

    private fun signByTerminal(srcPath: String, signPath: String) {
        removeFile(signPath)
        executeByTerminal("gpg --pinentry-mode loopback --passphrase 123456 -u $keyId1 --output $signPath -s $srcPath")
    }
    
    private fun signByTerminalArmor(srcPath: String, signPath: String) {
        removeFile(signPath)
        executeByTerminal("gpg --pinentry-mode loopback --passphrase 123456 -u $keyId1 --armor --output $signPath -s $srcPath")
    }
    
    private fun verifyByTerminal(signPath: String, verifyPath: String) {
        removeFile(verifyPath)
        executeByTerminal("gpg --output $verifyPath --verify $signPath")
    }
    
    private fun clearTextSignByTerminal(srcPath: String, signPath: String) {
        removeFile(signPath)
        executeByTerminal("gpg --pinentry-mode loopback --passphrase 123456 -u $keyId1 --output $signPath -a --clear-sign $srcPath")
    }
    
    private fun clearTextVerifyByTerminal(signPath: String, verifyPath: String) {
        removeFile(verifyPath)
        executeByTerminal("gpg --output $verifyPath --verify $signPath")
    }
    
    private fun executeByTerminal(command: String) {
        val dividingLine = "".padEnd(100, '-')
        val shortDividingLine = "".padEnd(30, '-')

        myColorPrintln(dividingLine, 94)

        myColorPrintln("$shortDividingLine execute command $shortDividingLine")
        println(command)
        
        val runtime = Runtime.getRuntime()
        val exec = runtime.exec(command)
        exec.waitFor()
        
        val text = exec.inputStream.bufferedReader().readText()
        myColorPrintln("$shortDividingLine message $shortDividingLine")
        println(text)
        
        val errorText = exec.errorStream.bufferedReader().readText()
        myColorPrintln("$shortDividingLine error message $shortDividingLine")
        println(errorText)

        myColorPrintln(dividingLine, 94)
    }
    
    private fun myColorPrintln(str: String, color: Int = 34) {
        println("\u001B[0;${color}m$str\u001B[0m") 
    }
    
    private fun removeFile(filePath: String) {
        FileUtils.forceDelete(Paths.get(filePath).toFile())
    }
}
```

PS：请将BankException这个异常类替换成自己的，或者jre自带的异常。
