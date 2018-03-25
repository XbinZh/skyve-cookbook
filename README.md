# Skyve Cookbook
Examples and code samples for using the [Skyve](http://skyve.org/) framework.

### Contents
* [Deproxy](#deproxy)
* [Ordering a collection in a data table](#ordering-a-collection-in-a-data-table)
* [Troubleshooting](#troubleshooting)
  * [Heap space errors](heap-space-errors)
* [Creating Rest Services](#creating-rest-services)
* [Understanding Skyve Rest](#understanding-skyve-rest)
* [Problems with utf8 - character sets for other languages - MySQL](#problems-with-utf8---character-sets-for-other-languages---mysql)
* [Customer Scoped Roles](#customer-scoped-roles)

### Deproxy 
When to use:
- performing an `instanceof` fails when operating over subclasses of a document with a persistence strategy of single
- Skyve logs an error performing a `BindUtil.get()`:
  
```
14:09:22,785 SEVERE [SKYVE] (default task-14) Could not BindUtil.get(modules.finance.Transaction.TransactionExtension@b46ca56a#97175c28-e505-4f39-911e-9c6e11d0d6bf, allocations[0].relatedTransaction.source.selectedDebitAllocation)!
14:09:22,785 SEVERE [SKYVE] (default task-14) The subsequent stack trace relates to obtaining bean property selectedDebitAllocation from modules.finance.domain.PaymentIn@de1d73c9#58db72b4-4ad1-451d-9526-29ed7112f1d9
```

The `Util.deproxy()` utility method lets you resolve this by loading the correct subclass.
```java
Occurrence occ = edc.getToBeOccurrence();
occ = Util.deproxy(occ);
if (occ instanceof DeviceOccurrence) {
}
```

**[⬆ back to top](#contents)**

### Ordering a collection in a data table
Collections can be ordered or unordered, where an ordered collection allows the user to define the sort of items in the list via drag and drop. Ordering on a collection can be enabled by adding the `ordered="true"` attribute to the collection declaration.

Collections can also have a default sorting by specifying the sort columns in the collection declaration in the document:
```xml
<ordering>
  <order by="binding1" sort="ascending" />
  <order by="binding2" sort="ascending" />
</ordering>
```

**[⬆ back to top](#contents)**

### Troubleshooting

#### Heap space errors
During startup of the server, if you encounter a `java.lang.OutOfMemoryError: Java heap space`, a possible cause could be an error in your json file which can't be parse correctly. A sample stacktrace of this is below:
```
14:12:51,910 ERROR [org.jboss.msc.service.fail] (ServerService Thread Pool -- 64) MSC000001: Failed to start service jboss.undertow.deployment.default-server.default-host./skyve: org.jboss.msc.service.StartException in service jboss.undertow.deployment.default-server.default-host./skyve: java.lang.OutOfMemoryError: Java heap space
    at org.wildfly.extension.undertow.deployment.UndertowDeploymentService$1.run(UndertowDeploymentService.java:85)
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
    at org.jboss.threads.JBossThread.run(JBossThread.java:320)
Caused by: java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3332)
    at java.lang.AbstractStringBuilder.expandCapacity(AbstractStringBuilder.java:137)
    at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:121)
    at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:622)
    at java.lang.StringBuffer.append(StringBuffer.java:383)
    at org.skyve.impl.util.json.JSONReader.add(JSONReader.java:421)
    at org.skyve.impl.util.json.JSONReader.add(JSONReader.java:426)
    at org.skyve.impl.util.json.JSONReader.string(JSONReader.java:410)
    at org.skyve.impl.util.json.JSONReader.read(JSONReader.java:165)
    at org.skyve.impl.util.json.JSONReader.object(JSONReader.java:293)
    at org.skyve.impl.util.json.JSONReader.read(JSONReader.java:103)
    at org.skyve.impl.util.json.JSONReader.read(JSONReader.java:79)
    at org.skyve.util.JSON.unmarshall(JSON.java:35)
    at org.skyve.impl.util.UtilImpl.readJSONConfig(UtilImpl.java:197)
    at org.skyve.impl.web.SkyveContextListener.contextInitialized(SkyveContextListener.java:60)
```

**[⬆ back to top](#contents)**

### Creating REST Services

The `BasicAuthFilter` provided in Skyve can be used to allow authentication using a HTTP basic auth header. So a user can make REST requests using their existing credentials, or create a specific API user in Skyve which has permission to the API.

The `web.xml` in the project will need to be updated to configure the filter, similar to the SkyveFacesFilter. Specify the 
init params for realm (optional and defaulted) and unsecured (optional).

You can either configure the filter in `web.xml` (see SkyveFacesFilter for something simliar) or by using a `@WebFilter` annotation on the class.

For example: 

```xml
@WebFilter(filterName = "BasicAuthFilter", urlPatterns = {"/api/*"})
```

The filter has to init parameters:

```java
private String realm = “Protected”;
private String[] unsecuredURLPrefixes;
```

The realm is used when an unauthorised response is sent, it is an arbitrary value.
The unsecured URL prefixes allows you to create exceptions that will not be secured by the filter.
Its similar in use to SkyveFacesFilter in web.xml (the URL prefixes are separated by a newline).

### Understanding Skyve Rest 
Skyve provides a number of Rest endpoints to support whatever application you build.

However to use these you will require a knowledge of Rest and web filters. An integration test is available at [BasicAuthIT.java](https://github.com/skyvers/skyve/blob/master/skyve-ee/src/test/org/skyve/impl/web/filter/rest/BasicAuthIT.java).

You can  trial the endpoints in your browser to gain an understanding of how they work, before you begin coding interactions. To trial the endpoints, you'll need to edit the web.xml which is located in your project at:
`/src/main/webapp/WEB-INF/web.xml`

#### Configuring The Session Filter
Insert this filter and mapping after the other filters:
```xml
<filter>
  <display-name>RestFilter</display-name>
  <filter-name>RestFilter</filter-name>
  <filter-class>org.skyve.impl.web.filter.rest.SessionFilter</filter-class>
</filter>
<filter-mapping>
  <filter-name>RestFilter</filter-name>
  <url-pattern>/rest/*</url-pattern>
</filter-mapping>
```    

Then redploy your app (or restart your app server).

#### Testing end points
The SessionFilter will allow you to interact with the end points while you have a valid Session, and you'll be redirected to a login page at the first interaction to authenticate. Once you've logged in, you can then exercise the endpoints using your browser. The SessionFilter allows you to make REST calls after initial login that will propagate the remote user onto the REST call’s execution context (the logged in user).

For example:

```
http://localhost:8080/<projectName>/rest/json/admin/Contact
```

This end point will return an array of json strings for all user contacts. Note that the address includes
- _rest_ - the filtered service 
- _json_ - the specific implemented result type
- _admin_ - the Skyve module
- _Contact_ - the Skyve document

Your result will be similar to the following:
```json
[
  {
    "bizModule":"admin",
    "bizDocument":"Contact",
    "name":"admin",
    "contactType":"Person",
    "email1":"admin@skyve.org",
    "mobile":"",
    "image":"",
    "bizId":"e4d033b8-7012-47f1-9344-f1676fea6be5",
    "bizCustomer":"bizHub",
    "bizDataGroupId":null,
    "bizUserId":"01f82b45-9374-434f-a058-a94c5fa98329"
  }
]
```

In the example above, there is only one Contact in the database, called admin. The fields returned are the fields which would be returned if you used the menu item Contacts in admin module, that is, the default Skyve query for Contacts.

To retrieve a specific instance, use the bizId of the instance in the address, for example:

```
http://localhost:8080/<projectName>/rest/json/admin/Contact/e4d033b8-7012-47f1-9344-f1676fea6be5
```

Note that the id matches the bizId in the prior example, and that the entire Contact object is now retrieved - not only the fields in the default query. This includes the result of conditions declared in the Contact document.

```json
{
  "created":true,
  "notPersisted":false,
  "persisted":true,
  "changed":false,
  "notCreated":false,
  "notChanged":true,
  "bizId":"e4d033b8-7012-47f1-9344-f1676fea6be5",
  "bizVersion":0,
  "bizLock":"20180305223718156admin",
  "bizFlagComment":null,
  "bizDataGroupId":null,
  "bizUserId":"01f82b45-9374-434f-a058-a94c5fa98329",
  "name":"admin",
  "contactType":"person",
  "email1":"admin@skyve.org",
  "mobile":null,
  "image":null
}
```

To retrieve data from a specific query, provide the query name, for example:

```
http://localhost:8080/<projectName>/rest/json/query/admin/qContacts/
```

To retrieve a paged result set, provide the start and end rows and the specific Skyve query name, for example to retrieve results 0 to 9:

```
http://localhost:8080/siteStorage/rest/json/query/admin/qContacts?start=0&end=9
```

There are also endpoints provide for insert, update and delete. 

```
http://localhost:8080/<projectName>/rest/json/insert/
http://localhost:8080/<projectName>/rest/json/update/
http://localhost:8080/<projectName>/rest/json/delete/
```

These require a complete json representation of the object to be manipulated. For example, the update the email address of the contact `admin` in the prior example from `admin@skyve.org` to `test@skyve.org`, supply the modified json as follows:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"created":true,"notPersisted":false,"persisted":true,"changed":false,"notCreated":false,"notChanged":true,"bizId":"e4d033b8-7012-47f1-9344-f1676fea6be5","bizVersion":0,"bizLock":"20180305223718156admin","bizFlagComment":null,"bizDataGroupId":null,"bizUserId":"01f82b45-9374-434f-a058-a94c5fa98329","name":"admin","contactType":"person","email1":"test@skyve.org","mobile":null,"image":null}' http://localhost:8080/<projectName>/rest/json/update/
```
_Note: on Windows, you will need to use double quotes instead of single quotes around the JSON, and escape the double quotes with a backslash, e.g. "{\"created\":true. This can also be performed as a GET request if the JSON is properly escaped._

#### Using the BasicAuthFilter
When you're ready to start coding comment out the following in your `web.xml`:

```xml
<!-- Secure rest services -->
<url-pattern>/rest/*</url-pattern>
```

This will stop these url patterns redirecting to the login page, and change the filter to the `BasicAuthFilter.java` as follows:

```xml
<filter-class>org.skyve.impl.web.filter.rest.SessionFilter</filter-class>
```

to

```xml
<filter-class>org.skyve.impl.web.filter.rest.BasicAuthFilter</filter-class>
```

The BasicAuthFilter allows authentication using a HTTP basic auth header. You can use the class by just changing the package to your package - there are no extra dependencies introduced by its use.

You can either configure the filter in web.xml (see SkyveFacesFilter for something similar) or by using a `@WebFilter` annotation on the class.

For example:

```
@WebFilter(filterName = "BasicAuthFilter", urlPatterns = {"/api/*"})
```

The filter has two init parameters:

```
private String realm = “Protected”;
private String[] unsecuredURLPrefixes;
```

The realm is used when an unauthorised response is sent (it is an arbitrary value). The unsecured URL prefixes allows you to create exceptions that will not be secured by the filter. For comparison, review the SkyveFacesFilter in web.xml (the URL prefixes are separated by a newline).

#### Example (CURL)
```bash
curl --user name:password http://192.168.43.91:8080/<projectName>/rest/json/query/admin/qContacts
```

#### Example (React)
```javascript
const base64 = require('base-64');
var headers = new Headers();
headers.append("Authorization", "Basic " + base64.encode("admin:password01"));
console.log(headers);
fetch('http://192.168.43.91:8080/<projectName>/rest/json/query/admin/qContacts', {
headers: headers
})
.then((response) => {
console.log('response:',response);
})
.catch((error) => {
console.error("err" + error);
}
```

#### Other Resources
https://github.com/skyvers/skyve/tree/master/skyve-web/src/main/java/org/skyve/impl/web/service/rest
https://github.com/skyvers/skyve/blob/master/skyve-ee/src/test/org/skyve/impl/web/filter/rest/BasicAuthIT.java

**[⬆ back to top](#contents)**

### Problems with utf8 - character sets for other languages - MySQL
If your Skyve application is not storing utf8 chars correctly, and you're using MySQL, check that MySQL is configured for utf8. Check the charset of the DB and tables, e.g. the default  is 'latin1'.

In the my.cnf file (for MySQL), check that you have the following:
```
[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

[mysqld]
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
```

To keep an existing database once this change has been made, export the schema from MySQL workbench, use text edit change latin1 to utf8, then drop your schema and import the edited one.

If you don't need to keep existing data, then after the my.cnf changes above, drop your schema, create a new one, then use Skyve bootstrap (in the json settings file) to log in and let Skyve create the new schema for you.

#### Other Resources
https://stackoverflow.com/questions/3513773/change-mysql-default-character-set-to-utf-8-in-my-cnf

**[⬆ back to top](#contents)**

### Customer Scoped Roles

While Groups and Roles allow user permissions to be managed at the _module_ level, some applications have permissions which span multiple modules and the permissions need a way to be linked to function correctly (e.g. a user may have incomplete access to application features if module permissions were not correctly applied to their account). To address this, a customer scoped role can be defined in the `customer.xml` which allows roles to be aggregated together into something coherent and cross module.

```xml
<roles allowModuleRoles="true">
    <!-- External Access Roles -->
    <role name="External Basic">
        <description>Basic role for all external users</description>
        <roles>
            <role module="admin" name="BasicUser"/>
            <role module="admin" name="AppUser"/>
            <role module="module1" name="ExternalBasic"/>
            <role module="module2" name="ExternalBasic"/>
        </roles>
    </role>
</roles>
```

The `allowModuleRoles` property controls whether the module roles show up in the admin User Interface or not.

Customer roles aggregate only module roles, they cannot reference other customer roles. Groups can be used to aggregate either module rules or customer roles (if enabled by `allowModuleRoles`).

**[⬆ back to top](#contents)**
