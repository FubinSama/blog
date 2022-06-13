# 基于BouncyCastle实现的PGP工具类

## gpg命令行工具的常用命令

> `https://www.ruanyifeng.com/blog/2013/07/gpg.html`

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
import org.bouncycastle.bcpg.ArmoredOutputStream
import org.bouncycastle.bcpg.CompressionAlgorithmTags
import org.bouncycastle.bcpg.SymmetricKeyAlgorithmTags
import org.bouncycastle.jce.provider.BouncyCastleProvider
import org.bouncycastle.openpgp.*
import java.io.ByteArrayOutputStream
import java.io.InputStream
import java.io.OutputStream
import java.math.BigInteger
import java.nio.file.Paths
import java.security.SecureRandom
import java.time.Instant
import java.util.*
import kotlin.io.path.inputStream
import kotlin.io.path.outputStream

/**
 * <p>
 *  PGPUtils
 * </p>
 * @author wfb
 * @version 1.0
 * @since 2022-04-08 10:52:19
 */
object PGPUtils {

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的密钥，并通过密码解析出私钥
     * 这里的keyId需要填写16进制的形式
     */
    fun getPrivateKey(keyStream: InputStream, keyId: String, pass: String): PGPPrivateKey {
        val keyIdL = BigInteger(keyId, 16).toLong()
        return getPrivateKey(keyStream, keyIdL, pass.toCharArray())
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的密钥，并通过密码解析出私钥
     * 注意keyId对应的16位16进制数最高位是1时，即为负数时，java没法单纯的用0x来将16进制数转为Long，这时需要自己做一下转换
     */
    fun getPrivateKey(keyStream: InputStream, keyId: Long, pass: CharArray): PGPPrivateKey {
        val provider = BouncyCastleProvider()
        PGPUtil.getDecoderStream(keyStream).use { stream ->
            val keyRingCollection = PGPSecretKeyRingCollection(stream)
            return keyRingCollection.getSecretKey(keyId)?.extractPrivateKey(pass, provider)
                ?: throw PGPException("Don't find the keyId[$keyId] in this keyRingCollection")
        }
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的公钥
     * 这里的keyId需要填写16进制的形式
     */
    fun getPublicKey(keyStream: InputStream, keyId: String): PGPPublicKey {
        val keyIdL = BigInteger(keyId, 16).toLong()
        return getPublicKey(keyStream, keyIdL)
    }

    /**
     * 通过读取私钥keyRing文件流来获取指定的keyId对应的公钥
     * 注意keyId对应的16位16进制数最高位是1时，即为负数时，java没法单纯的用0x来将16进制数转为Long，这时需要自己做一下转换
     */
    fun getPublicKey(keyStream: InputStream, keyId: Long): PGPPublicKey {
        PGPUtil.getDecoderStream(keyStream).use { stream ->
            val keyRingCollection = PGPPublicKeyRingCollection(stream)
            return keyRingCollection.getPublicKey(keyId)
                ?: throw PGPException("Don't find the keyId[$keyId] in this keyRingCollection")
        }
    }

    /**
     * 将src流使用指定的公钥加密到dst流中
     * 可以配置是否对原始数据压缩，是否转化为asc的文本格式，是否添加完整性检查
     */
    fun encrypt(
        src: InputStream, dst: OutputStream,
        publicKey: PGPPublicKey,
        fileName: String, modifyTime: Long,
        compressAlgorithmEnum: CompressAlgorithmEnum = CompressAlgorithmEnum.UNCOMPRESSED,
        symmetricKeyAlgorithmEnum: SymmetricKeyAlgorithmEnum = SymmetricKeyAlgorithmEnum.AES_128,
        armor: Boolean = false, withIntegrityCheck: Boolean = false
    ) {
        val provider = BouncyCastleProvider()
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
        // 对拆包压缩后的数据进行加密并写入
        val encryptedDataGenerator = PGPEncryptedDataGenerator(
            getSymmetricKeyAlgorithmTags(symmetricKeyAlgorithmEnum),
            withIntegrityCheck,
            SecureRandom(),
            provider
        )
        encryptedDataGenerator.addMethod(publicKey)
        // 将数据写入
        encryptedDataGenerator.open(outStream, pgpBytes.size.toLong()).use {
            it.write(pgpBytes)
        }
    }

    /**
     * 将src流使用指定的私钥解密到dst流中
     * 会自动判断使用的算法，是否需要解密，是否要进行完整性检查
     */
    fun decrypt(
        src: InputStream, dst: OutputStream,
        privateKey: PGPPrivateKey
    ) {
        val provider = BouncyCastleProvider()
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
        var message = PGPObjectFactory(ped.getDataStream(privateKey, provider)).nextObject()
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

    @JvmStatic
    fun main(args: Array<String>) {
        val keyId = "BF999AD71502F790"
        val privateKey = getPrivateKey(Paths.get("/Users/bin/MIB/测试使用/pgp/private-key.txt").inputStream(), keyId, "123456")
        val publicKey = getPublicKey(Paths.get("/Users/bin/MIB/测试使用/pgp/public-key.txt").inputStream(), keyId)
        val srcPath = Paths.get("/Users/bin/MIB/测试使用/pgp/test.txt")
        val encryptPath = Paths.get("/Users/bin/MIB/测试使用/pgp/test_encrypt.txt")
        val decryptPath = Paths.get("/Users/bin/MIB/测试使用/pgp/test_decrypt.txt")
        encrypt(srcPath.inputStream(), encryptPath.outputStream(), publicKey, "test.txt", Instant.now().toEpochMilli())
        decrypt(encryptPath.inputStream(), decryptPath.outputStream(), privateKey)
    }

}
```
