---
name: "v5-sso"
description: "Develop a complete V5 SSO (Single Sign-On) integration plugin with Controller-based PC/mobile differentiation, OkHttpUtil HTTP client, and TokenUtil for H5 token retrieval. Use when the user needs to implement a full SSO plugin with portalToken handling, third-party callback, user lookup by employee code, and mobile H5 redirect."
---

# V5 SSO Integration Plugin (Complete with Controller + OkHttpUtil + TokenUtil)

This skill provides a complete template for developing a V5 single sign-on plugin that supports:
- **PC端**: SSO handshake via `SSOLoginHandshakeAbstract` → `/seeyon/login/sso?from=xxx&ticket=xxx`
- **移动端**: H5 token retrieval via `TokenUtil` → `/seeyon/H5/collaboration/index.html?token=xxx`
- **Controller层**: `type` parameter to differentiate PC/mobile, `ticket` as third-party ticket, `tourl` for redirect
- **OkHttpUtil**: OkHttp-based HTTP client with SSL bypass (replaces Hutool/HttpRequest)
- **TokenUtil**: H5 token retrieval via REST API with authentication

---

## Part 1: Architecture Overview

### 1.1 Complete SSO Flow

```
Third-Party Portal
    ↓ redirects to /kkportalSSO.do?type=pc|mobile&ticket=xxx&tourl=xxx
{PluginName}Controller.index()
    ├── type=pc → ssoTicket = portalToken_timestamp
    │   └── redirect → /seeyon/login/sso?from=xxx&ticket=ssoTicket
    │       └── SSOLoginContext → {PluginName}SSOLoginImpl.handshake(ssoTicket)
    │           ├── extract portalToken from ticket
    │           ├── call SSM callback API → get employeeCode
    │           ├── query OA user by employeeCode
    │           └── return loginName → OA auto-login
    │
    └── type=mobile → call SSM callback API → get employeeCode
        └── query OA user → TokenUtil.getToken(member)
            └── redirect → /seeyon/H5/collaboration/index.html?token=h5Token&loginName=&html=tourl
```

### 1.2 Components

| Component | Purpose |
|-----------|---------|
| `{PluginName}Controller` | Entry point: receives `type`, `ticket`, `tourl`; routes to PC or mobile flow |
| `{PluginName}SSOLoginImpl` | SSO handshake: validates ticket, looks up user, returns login name |
| `KKOrgDao` / `KKOrgDaoImpl` | DAO: queries OA user by employee code |
| `TokenUtil` | H5 token retrieval via REST API (`/seeyon/rest/token/...`) |
| `OkHttpUtil` | HTTP client utility (OkHttp) with SSL bypass, used by both Controller and TokenUtil |
| `pluginProperties.xml` | Configurable properties (OA URL, SSM callback, REST credentials) |
| `pluginCfg.xml` | Plugin metadata (id, name, category) |
| `spring-kkportalsssm-sso.xml` | Spring: SSOLoginContext bean with handshake impl |
| `spring-kkportalsssm-api.xml` | Spring: Controller bean + DAO bean |
| `needless_check_login_*.xml` | Login whitelist for Controller URL |

---

## Part 2: Project Structure

### 2.1 Complete Directory Layout

```
apps-customize/
├── pom.xml
└── src/
    └── main/
        ├── java/
        │   └── com/seeyon/apps/{pluginName}/
        │       ├── controller/
        │       │   └── {PluginName}Controller.java      # Controller: PC/mobile routing
        │       ├── dao/
        │       │   ├── KKOrgDao.java                    # DAO interface
        │       │   └── KKOrgDaoImpl.java                # DAO implementation
        │       ├── sso/
        │       │   └── {PluginName}SSOLoginImpl.java        # SSO handshake implementation
        │       └── util/
        │           ├── OkHttpUtil.java                  # OkHttp HTTP client (MUST generate)
        │           └── TokenUtil.java                   # H5 token retrieval (MUST generate)
        ├── resources/
        │   ├── needless_check_login_{pluginName}.xml             # Login whitelist
        │   └── needless_check_login_recheck_{pluginName}.xml     # Recheck whitelist
        └── webapp/
            └── WEB-INF/
                └── cfgHome/
                    └── plugin/
                        └── {pluginName}/
                            ├── pluginCfg.xml
                            ├── pluginProperties.xml
                            └── spring/
                                ├── spring-{pluginName}-sso.xml   # SSO bean
                                └── spring-{pluginName}-api.xml   # Controller + DAO beans
```

---

## Part 3: OkHttpUtil — HTTP Client Utility (REQUIRED)

**MUST generate this class** in `{pluginName}/util/OkHttpUtil.java`. It replaces Hutool's `HttpUtil` and `HttpRequest` with OkHttp.

### 3.1 Complete Implementation

```java
package com.seeyon.apps.{pluginName}.util;

import com.seeyon.ctp.common.log.CtpLogFactory;
import okhttp3.*;
import org.apache.commons.collections.MapUtils;
import org.apache.commons.logging.Log;

import javax.net.ssl.*;
import java.io.IOException;
import java.security.KeyStore;
import java.util.Arrays;
import java.util.Map;

public class OkHttpUtil {

    private static final Log log = CtpLogFactory.getLog(OkHttpUtil.class);

    public static String get(String url) throws IOException {
        OkHttpClient client = getUnsafeOkHttpClient();
        Request request = new Request.Builder()
                .url(url)
                .get()
                .addHeader("content-type", "application/json")
                .build();
        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                log.error("OkHttpUtil GET失败：" + response.message());
            }
            return response.body().string();
        }
    }

    public static String post(String url, String json) throws IOException {
        OkHttpClient client = getUnsafeOkHttpClient();
        MediaType mediaType = MediaType.parse("application/json; charset=utf-8");
        RequestBody body = RequestBody.create(mediaType, json);
        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .addHeader("content-type", "application/json")
                .build();
        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                log.error("OkHttpUtil POST失败：" + response.message());
            }
            return response.body().string();
        }
    }

    public static String post(String url, String json, Map<String, String> headers) throws IOException {
        OkHttpClient client = getUnsafeOkHttpClient();
        MediaType mediaType = MediaType.parse("application/json; charset=utf-8");
        RequestBody body = RequestBody.create(mediaType, json);
        Request.Builder requestBuilder = new Request.Builder()
                .url(url)
                .post(body)
                .addHeader("content-type", "application/json");
        if (MapUtils.isNotEmpty(headers)) {
            for (Map.Entry<String, String> entry : headers.entrySet()) {
                requestBuilder.addHeader(entry.getKey(), entry.getValue());
            }
        }
        Request request = requestBuilder.build();
        try (Response response = client.newCall(request).execute()) {
            if (!response.isSuccessful()) {
                log.error("OkHttpUtil POST失败：" + response.message());
            }
            return response.body().string();
        }
    }

    public static OkHttpClient getUnsafeOkHttpClient() {
        try {
            final TrustManager[] trustAllCerts = new TrustManager[]{
                    new X509TrustManager() {
                        @Override
                        public void checkClientTrusted(java.security.cert.X509Certificate[] chain, String authType) {
                        }

                        @Override
                        public void checkServerTrusted(java.security.cert.X509Certificate[] chain, String authType) {
                        }

                        @Override
                        public java.security.cert.X509Certificate[] getAcceptedIssuers() {
                            return new java.security.cert.X509Certificate[]{};
                        }
                    }
            };
            final SSLContext sslContext = SSLContext.getInstance("SSL");
            sslContext.init(null, trustAllCerts, new java.security.SecureRandom());
            final SSLSocketFactory sslSocketFactory = sslContext.getSocketFactory();
            OkHttpClient.Builder builder = new OkHttpClient.Builder();
            builder.sslSocketFactory(sslSocketFactory, getX509TrustManager());
            builder.hostnameVerifier(new HostnameVerifier() {
                @Override
                public boolean verify(String hostname, SSLSession session) {
                    return true;
                }
            });
            return builder.build();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public static X509TrustManager getX509TrustManager() {
        X509TrustManager trustManager = null;
        try {
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init((KeyStore) null);
            TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
            if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
                throw new IllegalStateException("Unexpected default trust managers:" + Arrays.toString(trustManagers));
            }
            trustManager = (X509TrustManager) trustManagers[0];
        } catch (Exception e) {
            e.printStackTrace();
        }
        return trustManager;
    }
}
```

**Key features**:
- `get(String url)` — HTTP GET request
- `post(String url, String json)` — HTTP POST with JSON body
- `post(String url, String json, Map<String, String> headers)` — HTTP POST with custom headers
- SSL certificate bypass for internal/self-signed APIs
- Uses `OkHttpClient` (OkHttp3) with `CtpLogFactory` logging

**Maven dependency** (add to pom.xml):
```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.9.0</version>
</dependency>
```

---

## Part 4: TokenUtil — H5 Token Retrieval (REQUIRED)

**MUST generate this class** in `{pluginName}/util/TokenUtil.java`. It retrieves an H5 token from the OA REST API for mobile SSO redirect.

### 4.1 Complete Implementation

```java
package com.seeyon.apps.{pluginName}.util;

import com.alibaba.fastjson.JSONObject;
import com.seeyon.ctp.common.AppContext;
import com.seeyon.ctp.common.log.CtpLogFactory;
import org.apache.commons.logging.Log;

public class TokenUtil {

    private static final Log log = CtpLogFactory.getLog(TokenUtil.class);

    private static String OAURL = AppContext.getSystemProperty("{pluginName}.oaUrl");
    private static String RESTNAME = AppContext.getSystemProperty("{pluginName}.restname");
    private static String RESTPWD = AppContext.getSystemProperty("{pluginName}.restpwd");

    public static String getToken(String loginName) {
        try {
            String url = OAURL + "/seeyon/rest/token";
            JSONObject param = new JSONObject();
            param.put("userName", RESTNAME);
            param.put("password", RESTPWD);
            param.put("loginName", loginName);
            String requestBody = param.toString();
            String result = OkHttpUtil.post(url, requestBody);
            log.info("【客开Log】获取H5 Token返回结果：" + result);
            JSONObject jsonObject = JSONObject.parseObject(result);
            if (jsonObject.containsKey("id")) {
                return jsonObject.getString("id");
            }
            return null;
        } catch (Exception e) {
            log.error("【客开Log】获取H5 Token失败：" + e.getMessage(), e);
            return null;
        }
    }
}
```

**Configuration properties** (in pluginProperties.xml):

```xml
<oaUrl mark="{VE}" desc="OA系统访问地址">http://127.0.0.1</oaUrl>
<restname mark="{VE}" desc="REST接口认证用户名">restAdmin</restname>
<restpwd mark="{VE}" desc="REST接口认证密码">123456</restpwd>
```

**REST API format**:
```
POST {oaUrl}/seeyon/rest/token
Content-Type: application/json
Body: {"userName":"{restname}","password":"{restpwd}","loginName":"{loginName}"}
```

**Return value**: String — the H5 token ID, or `null` if retrieval fails.

---

## Part 5: Controller — Entry Point with PC/Mobile Differentiation

### 5.1 Complete Implementation

```java
package com.seeyon.apps.{pluginName}.controller;

import com.alibaba.fastjson.JSONObject;
import com.seeyon.apps.{pluginName}.dao.KKOrgDao;
import com.seeyon.apps.{pluginName}.util.OkHttpUtil;
import com.seeyon.apps.{pluginName}.util.TokenUtil;
import com.seeyon.ctp.common.AppContext;
import com.seeyon.ctp.common.controller.BaseController;
import com.seeyon.ctp.common.exceptions.BusinessException;
import com.seeyon.ctp.common.log.CtpLogFactory;
import com.seeyon.ctp.organization.bo.V3xOrgMember;
import com.seeyon.ctp.organization.manager.OrgManager;
import com.seeyon.ctp.organization.po.OrgMember;
import com.seeyon.ctp.util.Strings;
import org.apache.commons.logging.Log;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.net.URLEncoder;

public class {PluginName}Controller extends BaseController {

    private static final Log log = CtpLogFactory.getLog({PluginName}Controller.class);

    private OrgManager orgManager;

    private KKOrgDao kkOrgDao;

    private String oaUrl = AppContext.getSystemProperty("{pluginName}.oaUrl");
    private String ssmCallbackUrl = AppContext.getSystemProperty("{pluginName}.ssmCallbackUrl");

    public void setOrgManager(OrgManager orgManager) {
        this.orgManager = orgManager;
    }

    public void setKkOrgDao(KKOrgDao kkOrgDao) {
        this.kkOrgDao = kkOrgDao;
    }

    public ModelAndView index(HttpServletRequest request, HttpServletResponse response) {
        try {
            String type = request.getParameter("type");
            String portalToken = request.getParameter("ticket");
            String tourl = request.getParameter("tourl");

            log.info("单点登录开始，type=" + type + "，ticket=" + portalToken);

            Long timeStamp = System.currentTimeMillis();
            String ssoTicket = portalToken + "_" + timeStamp;
            log.info("生成ssoTicket：" + ssoTicket);

            if ("mobile".equals(type)) {
                JSONObject userInfo = getSSMUserInfo(portalToken);
                if (userInfo == null) {
                    log.error("移动端单点登录失败：获取用户信息失败");
                    return null;
                }
                String employeeCode = userInfo.getString("employeeCode");
                if (Strings.isBlank(employeeCode)) {
                    log.error("移动端单点登录失败：员工编号为空");
                    return null;
                }
                V3xOrgMember member = findMemberByEmployeeCode(employeeCode);
                if (member == null) {
                    log.error("移动端单点登录失败：OA中不存在该用户");
                    return null;
                }

                String loginName = member.getLoginName();
                String h5Token = TokenUtil.getToken(loginName);
                if (Strings.isNotBlank(h5Token)) {
                    log.info("移动端H5获取TOKEN：" + h5Token);
                    String url = oaUrl + "/seeyon/H5/collaboration/index.html?token=" + h5Token + "&loginName=&html=";
                    if (Strings.isNotBlank(tourl)) {
                        url += URLEncoder.encode(tourl, "UTF-8");
                    } else {
                        url += "/seeyon/m3/apps/v5/portal/html/portalIndex.html";
                    }
                    log.info("移动端单点登录地址：" + url);
                    response.sendRedirect(url);
                } else {
                    log.error("移动端单点登录失败：获取H5 Token失败");
                }
                return null;
            }

            String url = oaUrl + "/seeyon/login/sso?from={pluginName}&ticket=" + URLEncoder.encode(ssoTicket, "UTF-8");
            if (Strings.isNotBlank(tourl)) {
                url += "&tourl=" + URLEncoder.encode(tourl, "UTF-8");
            }
            log.info("PC端单点登录地址：" + url);
            response.sendRedirect(url);
        } catch (Exception e) {
            log.error("单点登录异常：" + e.getMessage(), e);
        }
        return null;
    }

    private JSONObject getSSMUserInfo(String portalToken) {
        try {
            JSONObject params = new JSONObject();
            params.put("portalToken", portalToken);
            log.info("调用SSM解析用户信息接口，参数：" + params.toJSONString());
            String result = OkHttpUtil.post(ssmCallbackUrl, params.toJSONString());
            log.info("SSM返回结果：" + result);
            if (Strings.isNotBlank(result)) {
                JSONObject json = JSONObject.parseObject(result);
                if (json.getBooleanValue("success")) {
                    return json.getJSONObject("data");
                }
            }
        } catch (Exception e) {
            log.error("调用SSM接口失败：" + e.getMessage(), e);
        }
        return null;
    }

    private V3xOrgMember findMemberByEmployeeCode(String employeeCode) {
        try {
            OrgMember member = kkOrgDao.getMemberByCode(null, "code", employeeCode);
            if (member != null) {
                return orgManager.getMemberById(member.getId());
            }
        } catch (BusinessException e) {
            log.error("根据员工编号查询用户失败：" + e.getMessage(), e);
        }
        return null;
    }
}
```

### 5.2 Controller Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | `pc` or `mobile` — determines redirect strategy |
| `ticket` | Yes | Third-party SSO ticket (UUID) |
| `tourl` | No | Target URL after login; defaults to portal home |

### 5.3 PC Flow (type=pc or default)

```
portalToken → ssoTicket = portalToken_timestamp
redirect → /seeyon/login/sso?from={pluginName}&ticket=ssoTicket[&tourl=xxx]
```

### 5.4 Mobile Flow (type=mobile)

```
portalToken → call SSM API → get employeeCode
→ query OA user → TokenUtil.getToken(loginName)
→ H5 token → redirect to /seeyon/H5/collaboration/index.html?token=h5Token&loginName=&html=tourl
```

**Mobile H5 URL format**:
```
{oaUrl}/seeyon/H5/collaboration/index.html?token={h5Token}&loginName=&html={tourl}
```

- `token` — H5 token from REST API
- `loginName` — always empty string (authentication is via token)
- `html` — target path; if `tourl` is null, defaults to `/seeyon/m3/apps/v5/portal/html/portalIndex.html`

---

## Part 6: SSO Handshake Implementation

### 6.1 Complete Implementation

```java
package com.seeyon.apps.{pluginName}.sso;

import com.alibaba.fastjson.JSONObject;
import com.seeyon.apps.{pluginName}.dao.KKOrgDao;
import com.seeyon.apps.{pluginName}.util.OkHttpUtil;
import com.seeyon.ctp.common.AppContext;
import com.seeyon.ctp.common.exceptions.BusinessException;
import com.seeyon.ctp.common.log.CtpLogFactory;
import com.seeyon.ctp.organization.bo.V3xOrgMember;
import com.seeyon.ctp.organization.manager.OrgManager;
import com.seeyon.ctp.organization.po.OrgMember;
import com.seeyon.ctp.portal.sso.SSOLoginHandshakeAbstract;
import com.seeyon.ctp.util.Strings;
import org.apache.commons.logging.Log;

public class {PluginName}SSOLoginImpl extends SSOLoginHandshakeAbstract {

    private static final Log log = CtpLogFactory.getLog({PluginName}SSOLoginImpl.class);

    private String ssmCallbackUrl = AppContext.getSystemProperty("{pluginName}.ssmCallbackUrl");

    private OrgManager orgManager;

    private KKOrgDao kkOrgDao;

    public void setOrgManager(OrgManager orgManager) {
        this.orgManager = orgManager;
    }

    public void setKkOrgDao(KKOrgDao kkOrgDao) {
        this.kkOrgDao = kkOrgDao;
    }

    @Override
    public String handshake(String ticket) {
        log.info("握手类开始，ticket=" + ticket);

        if (Strings.isBlank(ticket)) {
            log.error("ticket为空");
            return null;
        }

        String portalToken = extractPortalToken(ticket);
        log.info("提取portalToken：" + portalToken);

        JSONObject userInfo = getSSMUserInfo(portalToken);
        if (userInfo == null) {
            log.error("调用SSM获取用户信息失败");
            return null;
        }

        String employeeCode = userInfo.getString("employeeCode");
        if (Strings.isBlank(employeeCode)) {
            log.error("SSM返回的员工编号为空");
            return null;
        }

        log.info("SSM返回员工编号：" + employeeCode);

        V3xOrgMember member = findMemberByEmployeeCode(employeeCode);
        if (member == null) {
            log.error("OA中不存在员工编号为 " + employeeCode + " 的用户");
            return null;
        }

        String loginName = member.getLoginName();
        log.info("握手类返回登录名：" + loginName);
        return loginName;
    }

    @Override
    public void logoutNotify(String ticket) {
        log.info("退出通知，ticket=" + ticket);
    }

    private String extractPortalToken(String ticket) {
        if (ticket.contains("_")) {
            return ticket.substring(0, ticket.lastIndexOf("_"));
        }
        return ticket;
    }

    private JSONObject getSSMUserInfo(String portalToken) {
        try {
            JSONObject params = new JSONObject();
            params.put("portalToken", portalToken);
            log.info("握手类调用SSM解析用户信息接口，参数：" + params.toJSONString());
            String result = OkHttpUtil.post(ssmCallbackUrl, params.toJSONString());
            log.info("握手类SSM返回结果：" + result);
            if (Strings.isNotBlank(result)) {
                JSONObject json = JSONObject.parseObject(result);
                if (json.getBooleanValue("success")) {
                    return json.getJSONObject("data");
                }
            }
        } catch (Exception e) {
            log.error("握手类调用SSM接口失败：" + e.getMessage(), e);
        }
        return null;
    }

    private V3xOrgMember findMemberByEmployeeCode(String employeeCode) {
        try {
            OrgMember member = kkOrgDao.getMemberByCode(null, "code", employeeCode);
            if (member != null) {
                return orgManager.getMemberById(member.getId());
            }
        } catch (BusinessException e) {
            log.error("握手类根据员工编号查询用户失败：" + e.getMessage(), e);
        }
        return null;
    }
}
```

### 6.2 Token Extraction

The controller appends `_{timestamp}` to the portalToken, forming a ticket:
```
ticket = portalToken + "_" + timestamp
```

The handshake extracts the original portalToken:
```java
String portalToken = ticket.substring(0, ticket.lastIndexOf("_"));
```

### 6.3 SSM Callback API

The SSM (third-party authentication) callback API is called via POST:
```
POST {ssmCallbackUrl}
Body: {"portalToken": "xxx"}
Response: {"success": true, "data": {"employeeCode": "xxx"}}
```

---

## Part 7: DAO Layer — User Lookup

### 7.1 DAO Interface

```java
package com.seeyon.apps.{pluginName}.dao;

import com.seeyon.ctp.organization.po.OrgMember;

public interface KKOrgDao {
    OrgMember getMemberByCode(Long accountId, String property, Object fieldValue);
}
```

### 7.2 DAO Implementation

```java
package com.seeyon.apps.{pluginName}.dao;

import com.seeyon.ctp.organization.po.OrgMember;
import com.seeyon.ctp.util.DBAgent;
import com.seeyon.ctp.util.Strings;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class KKOrgDaoImpl implements KKOrgDao {
    @Override
    public OrgMember getMemberByCode(Long accountId, String property, Object fieldValue) {
        StringBuilder hql = new StringBuilder();
        Map<String, Object> params = new HashMap<String, Object>();
        hql.append("SELECT m ");
        hql.append(" FROM OrgMember as m");
        hql.append(" WHERE ");
        hql.append(" m.deleted=false AND m.admin='0' AND m.virtual='0' AND m.assigned='1'");
        if (accountId != null) {
            hql.append(" AND m.orgAccountId=:accountId ");
            params.put("accountId", accountId);
        }
        if (Strings.isNotBlank(property) && !"null".equals(property)) {
            hql.append(" AND m.").append(property).append("=:feildvalue");
            params.put("feildvalue", fieldValue);
        }
        hql.append(" ORDER BY m.sortId ASC ,m.createTime desc");
        List list = DBAgent.find(hql.toString(), params);
        if (!list.isEmpty() && list.get(0) != null) {
            return (OrgMember) list.get(0);
        } else {
            return null;
        }
    }
}
```

**Key DAO patterns**:
- Uses `DBAgent.find()` — the platform's HQL query utility
- Filters: `deleted=false`, `admin='0'`, `virtual='0'`, `assigned='1'`
- Query by `code` property for employee code lookup
- Returns first match ordered by `sortId ASC, createTime DESC`

---

## Part 8: Configuration Files

### 8.1 pluginCfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plugin>
    <id>{pluginName}</id>
    <name>{pluginName}单点登录插件</name>
    <category>{category}</category>
</plugin>
```

### 8.2 pluginProperties.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<ctpConfig>
    <{pluginName}>
        <ssmCallbackUrl mark="{VE}" desc="SSM解析用户信息回调接口地址">http://127.0.0.1:8080/api/sso/restore</ssmCallbackUrl>
        <oaUrl mark="{VE}" desc="OA系统访问地址">http://127.0.0.1</oaUrl>
        <m3Url mark="{VE}" desc="M3移动端访问地址">http://127.0.0.1</m3Url>
        <restname mark="{VE}" desc="REST接口认证用户名">restAdmin</restname>
        <restpwd mark="{VE}" desc="REST接口认证密码">123456</restpwd>
    </{pluginName}>
</ctpConfig>
```

**Properties reference**:

| Property | Key | Usage |
|----------|-----|-------|
| SSM callback URL | `{pluginName}.ssmCallbackUrl` | Third-party SSM user info API |
| OA URL | `{pluginName}.oaUrl` | OA system base URL |
| M3 URL | `{pluginName}.m3Url` | Mobile M3 base URL |
| REST username | `{pluginName}.restname` | REST API authentication username |
| REST password | `{pluginName}.restpwd` | REST API authentication password |

### 8.3 Spring Config — SSO

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">

    <bean id="{pluginName}SSO" class="com.seeyon.ctp.portal.sso.SSOLoginContext">
        <property name="name" value="{pluginName}"/>
        <property name="ticketName" value="ticket"/>
        <property name="forward" value="true"/>
        <property name="handshake">
            <bean class="com.seeyon.apps.{pluginName}.sso.{PluginName}SSOLoginImpl"/>
        </property>
    </bean>
</beans>
```

**SSOLoginContext properties**:

| Property | Description |
|----------|-------------|
| `name` | SSO scheme name, used in URL: `/seeyon/{pluginName}.do` |
| `ticketName` | URL parameter name for the ticket (default: `ticket`) |
| `forward` | Whether to forward to OA homepage after login |
| `handshake` | The `SSOLoginHandshakeAbstract` implementation bean |

### 8.4 Spring Config — Controller + DAO

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans default-autowire="byName">
    <bean id="kkOrgDao" class="com.seeyon.apps.{pluginName}.dao.KKOrgDaoImpl"/>
    <bean name="/kkportalSSO.do" class="com.seeyon.apps.{pluginName}.controller.{PluginName}Controller"/>
</beans>
```

### 8.5 Login Whitelist — needless_check_login_{pluginName}.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean>
        <id>/kkportalSSO.do</id>
        <methods>
            <method>index</method>
        </methods>
    </bean>
</beans>
```

### 8.6 Login Whitelist — needless_check_login_recheck_{pluginName}.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean>
        <id>/kkportalSSO.do</id>
        <name>com.seeyon.apps.{pluginName}.controller.{PluginName}Controller</name>
        <methods>
            <method>index</method>
        </methods>
    </bean>
</beans>
```

---

## Part 9: Complete Development Checklist

### Step-by-step

1. **Create plugin folder**: `WEB-INF/cfgHome/plugin/{pluginName}/`
2. **Create pluginCfg.xml**: Define plugin ID, name, category
3. **Create pluginProperties.xml**: Define SSM callback, OA URL, REST credentials
4. **Create OkHttpUtil.java** (REQUIRED): `{pluginName}/util/OkHttpUtil.java`
5. **Create TokenUtil.java** (REQUIRED): `{pluginName}/util/TokenUtil.java`
6. **Create DAO interface + impl**: `KKOrgDao` / `KKOrgDaoImpl`
7. **Create SSO handshake**: `{PluginName}SSOLoginImpl` extending `SSOLoginHandshakeAbstract`
8. **Create Controller**: `{PluginName}Controller` extending `BaseController`
9. **Create Spring config for SSO**: `spring-{pluginName}-sso.xml`
10. **Create Spring config for Controller + DAO**: `spring-{pluginName}-api.xml`
11. **Create login whitelist XMLs**: `needless_check_login_*.xml`
12. **Add OkHttp Maven dependency**: `com.squareup.okhttp3:okhttp:4.9.0`

### File Generation Order

| Order | File | Required |
|-------|------|----------|
| 1 | `util/OkHttpUtil.java` | **YES** |
| 2 | `util/TokenUtil.java` | **YES** |
| 3 | `dao/KKOrgDao.java` | Yes |
| 4 | `dao/KKOrgDaoImpl.java` | Yes |
| 5 | `sso/{PluginName}SSOLoginImpl.java` | Yes |
| 6 | `controller/{PluginName}Controller.java` | Yes |
| 7 | `pluginCfg.xml` | Yes |
| 8 | `pluginProperties.xml` | Yes |
| 9 | `spring/spring-{pluginName}-sso.xml` | Yes |
| 10 | `spring/spring-{pluginName}-api.xml` | Yes |
| 11 | `needless_check_login_{pluginName}.xml` | Yes |
| 12 | `needless_check_login_recheck_{pluginName}.xml` | Yes |

---

## Part 10: Key Rules Summary

| # | Rule |
|---|------|
| 1 | **Always generate OkHttpUtil** in `util/` directory — replaces Hutool/HttpRequest |
| 2 | **TokenUtil.getToken(String loginName) returns String**: Input is loginName, returns token string directly or null |
| 3 | **Controller extends BaseController** — entry point with `type` parameter for PC/mobile |
| 4 | **Handshake extends SSOLoginHandshakeAbstract** — `handshake()` returns login name or null |
| 5 | **Ticket format**: `portalToken_timestamp` — controller appends, handshake extracts |
| 6 | **Use SSOLoginContext bean**: Wrap handshake with `name`, `ticketName`, `forward` |
| 7 | **Whitelist Controller URL**: `needless_check_login_*.xml` for `/kkportalSSO.do` |
| 8 | **Mobile H5 URL**: `loginName=` is empty — authentication via token, not loginName |
| 9 | **Use DBAgent for HQL**: Platform-standard query utility |
| 10 | **Filter OrgMember**: `deleted=false`, `admin='0'`, `virtual='0'`, `assigned='1'` |
| 11 | **Use CtpLogFactory**: Platform-standard logging |
| 12 | **Use AppContext.getSystemProperty()**: All config from `pluginProperties.xml` |
| 13 | **OkHttp replaces all HTTP**: Controller, handshake, and TokenUtil all use `OkHttpUtil` |
| 14 | **REST API auth**: TokenUtil posts `userName`、`password`、`loginName` JSON to `/seeyon/rest/token` |

---

## Part 11: URL Patterns Reference

### Controller Entry
```
GET /kkportalSSO.do?type=pc&ticket={token}&tourl={url}
GET /kkportalSSO.do?type=mobile&ticket={token}&tourl={url}
```

### PC Redirect
```
{oaUrl}/seeyon/login/sso?from={pluginName}&ticket={portalToken}_{timestamp}[&tourl={url}]
```

### Mobile H5 Redirect
```
{oaUrl}/seeyon/H5/collaboration/index.html?token={h5Token}&loginName=&html={tourl}
```

### REST Token API (called by TokenUtil)
```
POST {oaUrl}/seeyon/rest/token
Content-Type: application/json
Body: {"userName":"{restname}","password":"{restpwd}","loginName":"{loginName}"}
```

### SSM Callback API (called by Controller and Handshake)
```
POST {ssmCallbackUrl}
Content-Type: application/json
Body: {"portalToken": "xxx"}
Response: {"success": true, "data": {"employeeCode": "xxx"}}
```