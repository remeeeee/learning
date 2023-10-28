[jspxcms历史漏洞分析与复现-JAVA - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/10891#toc-7)

# 网站简单信息

## 前台

<img src=".\图片\Snipaste_2023-05-22_20-37-46.png" alt="Snipaste_2023-05-22_20-37-46" style="zoom:80%;" />



## 后台

<img src=".\图片\Snipaste_2023-05-22_20-38-21.png" alt="Snipaste_2023-05-22_20-38-21" style="zoom:80%;" />

# 第三方组件

对于审计者来说，在审计第三方组件的漏洞时，首先需要翻阅 pom.xml文件，该文件中 记录着这个项目使用的第三方组件及其版本号

```
表8-1 Jspxcms 使用的第三方组件及其版本号
commons-lang3 3.4
commons-net 3.4
commons-io 2.4
ehcache-core 2.6.11
shiro 1.3.2
lucene 3.6.2
htmlparser 2.1
quartz 2.2.2
poi 3.13
ant 1.9.6
prettytime 4.0.1.Final
owasp-java-html-sanitizer 20160924.1
UserAgentUtils 1.20
weixin4j-mp 1.7.4
imgscalr 4.2
jcaptcha 2.0.0
jodconverter-core 1.0.5
jacob 1.14.3
im4java 1.4.0
aliyun-sdk-mns 1.1.8
IKAnalyzer 2012_u6
qq-connect-Sdk4J 2.0.0
weibo4j-oauth2 beta3.1.1
jdbc.driver. 5.1.41
jsp-api 2.2.1
jaxb-api 2.3.0
powermock.version 1.6.6
```

# SQL审计

## 1．关键字全局搜索

根据 pom.xml 文件可以得知，这套CMS使用了 Hibernate 作为数据库持久化框架，某些未正确使用 Hibernate 框架的情况下会产生SQL注入漏洞。用户可以通过全局搜索关键字 【query】快速寻找可能存在的漏洞点

<img src=".\图片\Snipaste_2023-05-22_20-52-57.png" alt="Snipaste_2023-05-22_20-52-57" style="zoom:80%;" />

如下代码使用了占位符的方式构造了SQL语句，这种方式是不会产生SQL注入的

<img src=".\图片\Snipaste_2023-05-22_20-54-37.png" alt="Snipaste_2023-05-22_20-54-37" style="zoom:80%;" />

## 2．功能定点审计

### 用户会员中心

我们可以从程序的具体功能上进行定点的漏洞挖掘，与**数据库交互的位置**就有可能出现SQL注 入，比如用户会员中心页面：

<img src=".\图片\Snipaste_2023-05-23_10-46-16.png" alt="Snipaste_2023-05-23_10-46-16" style="zoom:80%;" />

根据路由信息 `info/35`，可以定位到程序代码在控制器 `core.web.fore.InfoController#info` 中

```java
@RequestMapping("/info/{id:[0-9]+}")
	public String info(@PathVariable Integer id, HttpServletRequest request, HttpServletResponse response,
			org.springframework.ui.Model modelMap) {
		return info(null, id, 1, request, response, modelMap);
	}
```

具体的功能逻辑代码实现在 info() 方法中，但此时已经可以判断此处不存在注入。因为 Java是强类型语言，id 需要是数字，不能是字符串，所以此处不存在SQL注入

### 用户名检查

在注册账户时，常有检验用户名的功能，而将用户名带入数据库查询的过程中可能存在SQL注入的问题`core/web/back/UserController#checkUsername` 中有一段检查用户名是否存在的代码，如下所示：

```java
/**
	 * 检查用户名是否存在
	 */
	@RequestMapping("check_username.do")
	public void checkUsername(String username, String original, HttpServletResponse response) {
		if (StringUtils.isBlank(username)) {
			Servlets.writeHtml(response, "false");
			return;
		}
		if (StringUtils.equals(username, original)) {
			Servlets.writeHtml(response, "true");
			return;
		}
		// 检查数据库是否重名
		boolean exist = service.usernameExist(username);
		if (!exist) {
			Servlets.writeHtml(response, "true");
		} else {
			Servlets.writeHtml(response, "false");
		}
	}
```

接下来一路跟踪 service.usernameExist() 方法，从controler到serviceImpl再到dao里，寻找到sql语句，用的占位符 `?`  ，所以不存在注入

<img src=".\图片\Snipaste_2023-05-23_10-57-04.png" alt="Snipaste_2023-05-23_10-57-04" style="zoom:80%;" />

对本套CMS的几个功能点进行审查时，在数据库交互的过程中采用了安全的编码方式，未发现 SQL注入漏洞。在挖掘SQL注入的过程中，用**全局搜索关键字**可以快速发现可能存在的漏洞点，常需要回溯找到上一级调用点，理清变量的传递过程，从而确定漏洞是否真实存在。而**定点功能的审计**大多从功能点的入口开始，逐步递进到SQL语句执行的部分，这是一个正向推理的过程。

总结：就是**功能点和关键函数**，一个正向走一个逆向推的过程

# SSRF审计

可能出现SSRF漏洞点的站点功能：

- 内容管理中的文档属性。
- 文件管理中的上传文件。
- 模块组件功能中的采集管理。
- 插件功能中的广告管理

猜测代码中可能出现SSRF的地方，http请求和远程加载之类的：

- 在com.jspxcms.common ip中，其主要代码的逻辑是通过IP地址查询实际地址。因此可能对传入的IP地址进行 URL 反查，并存在内部http请求，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.common —— web中，其主要代码是Spring MVC等Web相关类，其中可能存在自定义的与 http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.core —— domain中，其主要代码逻辑是实体类代码，其中可能存在自定义的与 http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.core —— service中，其主要代码逻辑是服务层代码，其中可能存在自定义的与 http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.core —— Web 中，其主要代码逻辑是Controller层代码，其中可能存在自定义的与 http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.ext 中，其主要代码逻辑是扩展模块的相关功能，其中可能存在自定义的与http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在com.jspxcms.plug中，其主要代码逻辑是插件模块的相关功能，其中可能存在自定义的与http 请求相关的函数，因此猜测其可能出现 SSRF漏洞。
- 在内容管理的文档属性功能点中，存在上传图片的功能，图片上传可能从远程加载或获取，对于远程加载或获取的功能点，可能存在 SSRF 漏洞。
- 在文件管理的上传文件功能点中，存在上传图片的功能，图片上传可能从远程加载或获取，对于远程加载或获取的功能点，可能存在 SSRF 漏洞。
- 在模块组件功能的采集管理功能点中，存在采集其他网站新闻的功能，该功能可能从远程加载或获取，对于远程加载或获取的功能点，可能存在 SSRF 漏洞。
- 在插件功能的广告管理功能点中，存在上传图片的功能，图片上传可能从远程加载或获取，对于远程加载或获取的功能点，可能存在 SSRF 漏洞。

审计 SSRF 时需要注意的敏感函数：

```java
URL.openConnection()
URL.openStream()
HttpClient.execute()
HttpClient.executeMethod()
HttpURLConnection.connect()
HttpURLConnection.getInputStream()
HttpServletRequest()
BasicHttpEntityEnclosingRequest()
DefaultBHttpClientConnection()
BasicHttpRequest()
```

## 搜索关键字

1、全局搜索 `HttpURLConnection`

<img src=".\图片\Snipaste_2023-05-23_11-32-47.png" alt="Snipaste_2023-05-23_11-32-47" style="zoom:80%;" />

在 `com/jspxcms/core/web/back/UploadControllerAbstract.java` 类的 `ueditorCatchImage()` 方法里找到了 `HttpURLConnection conn = (HttpURLConnection) new URL(src).openConnection();` 进行了 `openConnection()` 连接，关键代码如下：

```java
protected void ueditorCatchImage(Site site, HttpServletRequest request,
                                     HttpServletResponse response) throws IOException {
        GlobalUpload gu = site.getGlobal().getUpload();
        PublishPoint point = site.getUploadsPublishPoint();
        FileHandler fileHandler = point.getFileHandler(pathResolver);
        String urlPrefix = point.getUrlPrefix();

        StringBuilder result = new StringBuilder("{\"state\": \"SUCCESS\", list: [");
        List<String> urls = new ArrayList<String>();
        List<String> srcs = new ArrayList<String>();

        String[] source = request.getParameterValues("source[]"); //可控参数 source[]
        if (source == null) {
            source = new String[0];
        }
        for (int i = 0; i < source.length; i++) {
            String src = source[i];
            String extension = FilenameUtils.getExtension(src);
            // 格式验证
            if (!gu.isExtensionValid(extension, Uploader.IMAGE)) {
                // state = "Extension Invalid";
                continue;
            }
            HttpURLConnection.setFollowRedirects(false);
            HttpURLConnection conn = (HttpURLConnection) new URL(src).openConnection();//关键函数
            if (conn.getContentType().indexOf("image") == -1) {
                // state = "ContentType Invalid";
                continue;
            }
            if (conn.getResponseCode() != 200) {
                // state = "Request Error";
                continue;
            }
            String pathname = site.getSiteBase(Uploader.getQuickPathname(Uploader.IMAGE, extension));
            InputStream is = null;
            try {
                is = conn.getInputStream();
                fileHandler.storeFile(is, pathname);
            } finally {
                IOUtils.closeQuietly(is);
            }
            String url = urlPrefix + pathname;
            urls.add(url);
            srcs.add(src);
            result.append("{\"state\": \"SUCCESS\",");
            result.append("\"url\":\"").append(url).append("\",");
            result.append("\"source\":\"").append(src).append("\"},");
        }
        if (result.charAt(result.length() - 1) == ',') {
            result.setLength(result.length() - 1);
        }
        result.append("]}");
        logger.debug(result.toString());
        response.getWriter().print(result.toString());
    }
```

经过仔细阅读可知，该函数的功能是获取并下载远程URL 图片。首先传入 `source[]` 变量，然 后利用 getExtension 判断传入的URL文件扩展名，若是图片类型的文件，则修改文件名并保存到指 定路径，最终反馈到页面上。

找到了终点函数，接下来就该找调用关系，看看是谁调用了 `ueditorCatchImage()`  函数，此时很可能存在连环调用的套娃关系，最后得找到web路由并且由我们来触发。

在 `com/jspxcms/core/web/fore/UploadController.java` 里找到了调用  `ueditorCatchImage()` 函数：

<img src=".\图片\Snipaste_2023-05-23_11-46-26.png" alt="Snipaste_2023-05-23_11-46-26" style="zoom:67%;" />

最后找到路由，找到对应路径，传入所利用的参数。若请求端口开放就返回 `success`,不开放则服务器暴出 `500`，如下所示：

success：

<img src=".\图片\Snipaste_2023-05-23_11-51-20.png" alt="Snipaste_2023-05-23_11-51-20" style="zoom:50%;" />

500：

<img src=".\图片\Snipaste_2023-05-23_11-53-48.png" alt="Snipaste_2023-05-23_11-53-48" style="zoom: 50%;" />

另外后续利用：

<img src=".\图片\Snipaste_2023-05-23_11-56-03.png" alt="Snipaste_2023-05-23_11-56-03"  />

## 功能点审计

模块组件功能 —— 采集管理审计

对于功能点的审计和功能目录的审计略有不同，首先是要确定该功能点在站点的位置，了解其具体功能，比如：

<img src=".\图片\Snipaste_2023-05-23_12-10-57.png" alt="Snipaste_2023-05-23_12-10-57" style="zoom:67%;" />

可以看到该功能是获取远程 URL地址的html 源码页面，并且将源码输出到页面上。我们可以尝试修改采集的 URL 地址：

访问3306端口成功：

<img src=".\图片\Snipaste_2023-05-23_12-09-53.png" alt="Snipaste_2023-05-23_12-09-53" style="zoom:50%;" />

访问3307端口失败：

<img src=".\图片\Snipaste_2023-05-23_12-12-37.png" alt="Snipaste_2023-05-23_12-12-37" style="zoom:67%;" />

# RCE

在审计 RCE 时需要先查看项目使用了哪些依赖包

[jspxcms历史漏洞分析与复现-JAVA - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/10891#toc-6)

[Java代码审计（入门篇）.pdf](file:///E:/java安全/java审计书/Java代码审计（入门篇）.pdf)

# 文件上传

在之前暗月的项目里做了复现，参考之前的就行


