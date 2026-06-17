# AGENT.md — JBoss → Tomcat Migration Playbook

> Instructions for an automated agent migrating Java web applications from
> **JBoss / WildFly 8** to **Apache Tomcat 10**, deployed as **WAR files on a
> standalone Tomcat**. Apps are **Maven** builds: a mix of **Spring / Spring Boot**
> and **plain Java EE**. Messaging is **IBM MQ over JMS**. All dependencies are
> pulled from the **internal Nexus registry**.

---

## 0. Prime directive & scope

For **each application** under the source tree:

1. **Do not edit the JBoss app in place.** Create a parallel copy under a new
   `tomcat/` directory that **mirrors the existing directory structure**.
   - Source: `<app>/...`
   - Target: `tomcat/<app>/...` (same relative layout, same module names).
2. Migrate the copy to run on Tomcat 10 as a deployable WAR.
3. Never invent dependency versions. Resolve **every** library from the internal
   Nexus registry (Section 3). If a version is not in Nexus, **stop and report it**
   rather than reaching out to Maven Central.
4. After each app, produce a `tomcat/<app>/MIGRATION_NOTES.md` listing what changed,
   what was mapped, and any TODOs that need a human.

**Hard rule — the namespace change is mandatory.** JBoss 8 targets Java EE 7
(`javax.*`). **Tomcat 10 implements Jakarta EE 9+ (`jakarta.*`).** A WAR using
`javax.servlet`, `javax.jms`, `javax.annotation`, etc. **will not run** on Tomcat 10.
Every migrated app must move to the `jakarta.*` namespace. See Section 2.

---

## 1. Per-app migration procedure (run in order)

1. **Copy** `<app>/` → `tomcat/<app>/` preserving structure.
2. **Inventory** what JBoss was providing for this app (Section 4 checklist).
   Grep the source for: `standalone.xml`, `jboss-web.xml`, `jboss-deployment-structure.xml`,
   `*-ds.xml`, `@Resource`, `lookup(`, `InitialContext`, `javax.jms`, `javax.transaction`,
   `persistence.xml`, `beans.xml`, `faces-config.xml`.
3. **Namespace migration** `javax.*` → `jakarta.*` (Section 2).
4. **Rewrite `pom.xml`** (Section 3): repackage as `war`, add the libs JBoss used to
   provide, point everything at Nexus, remove `provided` scopes that Tomcat won't supply.
5. **Map container services** JBoss gave for free → Tomcat equivalents (Section 4):
   datasource/JNDI, transactions, connection pooling, security.
6. **Map IBM MQ connectivity** (Section 5) — the biggest single change.
7. **Add Tomcat descriptors**: `META-INF/context.xml`, updated `web.xml`,
   replace `jboss-web.xml` (Section 6).
8. **Build** with the Nexus settings (Section 3.3) and **deploy** to a local Tomcat 10
   to smoke-test (Section 7).
9. Write `MIGRATION_NOTES.md`.

---

## 2. Namespace migration: `javax.*` → `jakarta.*`

Tomcat 10 = Jakarta EE 9 = `jakarta.*` packages. Apply across **all** `.java`,
`.xml`, `.properties`, JSP/JSF, and descriptor files.

| Old (`javax.*`)            | New (`jakarta.*`)            |
|----------------------------|------------------------------|
| `javax.servlet.*`          | `jakarta.servlet.*`          |
| `javax.servlet.jsp.*`      | `jakarta.servlet.jsp.*`      |
| `javax.servlet.jsp.jstl.*` | `jakarta.servlet.jsp.jstl.*` |
| `javax.jms.*`              | `jakarta.jms.*`              |
| `javax.annotation.*`       | `jakarta.annotation.*`       |
| `javax.transaction.*`      | `jakarta.transaction.*`      |
| `javax.persistence.*`      | `jakarta.persistence.*`      |
| `javax.validation.*`       | `jakarta.validation.*`       |
| `javax.ejb.*`              | `jakarta.ejb.*`              |
| `javax.enterprise.*` (CDI) | `jakarta.enterprise.*`       |
| `javax.inject.*`           | `jakarta.inject.*`           |
| `javax.faces.*` (JSF)      | `jakarta.faces.*`            |
| `javax.ws.rs.*` (JAX-RS)   | `jakarta.ws.rs.*`            |
| `javax.xml.bind.*` (JAXB)  | `jakarta.xml.bind.*`         |
| `javax.mail.*`             | `jakarta.mail.*`             |

**Do NOT rename** these — they are JDK/SE, not Jakarta EE, and stay `javax.*`:
`javax.sql.*`, `javax.naming.*`, `javax.net.*`, `javax.crypto.*`, `javax.security.auth.*`,
`javax.management.*`, `javax.xml.parsers/transform/xpath`.

**web.xml namespace bump** — change the schema to Servlet 5.0 (jakarta):

```xml
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
                             https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">
```
Do the same for `persistence.xml` (Jakarta Persistence 3.0), `beans.xml` (CDI 3.0),
`faces-config.xml` (Faces 3.0) namespaces.

**Spring apps:** the `jakarta.*` namespace requires **Spring 6 / Spring Boot 3**.
- Spring Boot apps → bump to Spring Boot 3.x. Many `spring.factories` /
  `javax.*` config references move; let the build errors drive the fixes.
- Plain Spring (non-Boot) → Spring Framework 6.x.
- These versions require **JDK 17+**. Confirm the Tomcat 10 host runs on JDK 17+.

> Tooling tip: the **Eclipse Transformer** or **`org.apache.tomcat:jakartaee-migration`**
> tool can mechanically rewrite `javax`→`jakarta` in bytecode/sources. Prefer source-level
> changes here so the result is reviewable, but the tool is acceptable for third-party
> jars that have no Jakarta release in Nexus.

---

## 3. Maven & Nexus

### 3.1 Repackage as WAR

In each `pom.xml`:
- `<packaging>war</packaging>`
- Add `maven-war-plugin`.
- Set `maven.compiler.release` to `17` (or the host JDK).
- **Remove reliance on JBoss BOMs / modules** (`jboss-javaee-*-spec`,
  `wildfly-*`, `jboss-deployment-structure.xml` module imports).

### 3.2 Scope rules — what Tomcat provides vs. what you must bundle

JBoss bundled most Java EE APIs *and implementations*. Tomcat 10 only provides the
**Servlet/JSP/EL/WebSocket** APIs. Everything else must ship **inside the WAR**.

| Capability        | On JBoss | On Tomcat 10 (WAR)                              | Maven scope |
|-------------------|----------|------------------------------------------------|-------------|
| Servlet/JSP/EL    | provided | provided by Tomcat                             | `provided`  |
| JSTL impl         | bundled  | **bundle it**                                  | `compile`   |
| JPA + Hibernate   | bundled  | **bundle Hibernate**                           | `compile`   |
| Bean Validation   | bundled  | **bundle Hibernate Validator**                 | `compile`   |
| CDI (Weld)        | bundled  | **bundle Weld servlet** (only if app uses CDI) | `compile`   |
| JSF (Mojarra)     | bundled  | **bundle Mojarra** (only if app uses JSF)      | `compile`   |
| JAX-RS            | bundled  | **bundle** (Jersey/RESTEasy) or use Spring MVC | `compile`   |
| JTA transactions  | built-in | **none** — see Section 4.2                     | `compile`   |
| JMS / IBM MQ      | RA       | **bundle MQ client** — see Section 5           | `compile`   |
| JDBC driver       | module   | **bundle or put in `$CATALINA_BASE/lib`**      | see 4.1     |

### 3.3 Point Maven at internal Nexus

Add (or confirm) `~/.m2/settings.xml` so **all** resolution goes through Nexus.
Replace `NEXUS_BASE` with the real internal URL — **ask the user if unknown; never
guess a public mirror.**

```xml
<settings>
  <mirrors>
    <mirror>
      <id>internal-nexus</id>
      <name>Internal Nexus</name>
      <url>https://NEXUS_BASE/repository/maven-public/</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>internal-nexus</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_TOKEN}</password>
    </server>
  </servers>
</settings>
```

### 3.4 Required libraries (resolve ALL from Nexus)

Pin versions to whatever the internal Nexus publishes — **do not hardcode the
examples below if Nexus carries a different approved version.** List of artifacts
the migrated apps will typically need:

**Jakarta APIs (compile unless noted):**
- `jakarta.servlet:jakarta.servlet-api` — `provided`
- `jakarta.servlet.jsp.jstl:jakarta.servlet.jsp.jstl-api`
- `org.glassfish.web:jakarta.servlet.jsp.jstl` (JSTL impl)
- `jakarta.jms:jakarta.jms-api`
- `jakarta.persistence:jakarta.persistence-api`
- `jakarta.transaction:jakarta.transaction-api`
- `jakarta.validation:jakarta.validation-api`
- `jakarta.annotation:jakarta.annotation-api`
- `jakarta.enterprise:jakarta.enterprise.cdi-api` (if CDI)
- `jakarta.faces:jakarta.faces-api` (if JSF)

**Implementations to bundle:**
- `org.hibernate.orm:hibernate-core` (Jakarta-compatible 6.x)
- `org.hibernate.validator:hibernate-validator` + `org.glassfish.expressly:expressly`
- `org.jboss.weld.servlet:weld-servlet-shaded` (if CDI)
- `org.glassfish:jakarta.faces` (Mojarra, if JSF)

**IBM MQ (Jakarta build — critical, see Section 5):**
- `com.ibm.mq:com.ibm.mq.jakarta.client`  ← **the `jakarta` artifact, NOT `com.ibm.mq.allclient`**

**Connection pooling / transactions (see Section 4):**
- `com.zaxxer:HikariCP`  *(or `org.apache.tomcat:tomcat-jdbc`)*
- `com.atomikos:transactions-jta` + `com.atomikos:transactions-jdbc` *(only if real JTA/XA is required)*

**JDBC driver** — match the database, e.g. `com.oracle.database.jdbc:ojdbc11`,
`org.postgresql:postgresql`, `com.microsoft.sqlserver:mssql-jdbc`.

**Spring (Boot 3 / Framework 6) — only for Spring apps:**
- `org.springframework.boot:spring-boot-starter-web` (3.x), `...-starter-tomcat` as `provided`
- `org.springframework:spring-jms` + `org.springframework.boot:spring-boot-starter-activemq`-style starter is NOT for IBM MQ; use `spring-jms` with the MQ client directly, or `com.ibm.mq:mq-jms-spring-boot-starter` (Jakarta version) if Nexus carries it.

---

## 4. Mapping JBoss container services → Tomcat

JBoss configured these centrally in `standalone.xml`; Tomcat configures them
per-app in `META-INF/context.xml` or globally in `$CATALINA_BASE/conf`.

### 4.1 Datasources / JNDI

JBoss `<datasource jndi-name="java:/jdbc/AppDS">` → Tomcat JNDI `Resource`.

`tomcat/<app>/src/main/webapp/META-INF/context.xml`:
```xml
<Context>
  <Resource name="jdbc/AppDS" auth="Container"
            type="javax.sql.DataSource"
            factory="org.apache.tomcat.jdbc.pool.DataSourceFactory"
            driverClassName="oracle.jdbc.OracleDriver"
            url="jdbc:oracle:thin:@//db-host:1521/SERVICE"
            username="${db.user}" password="${db.pass}"
            maxActive="50" maxIdle="10" minIdle="5"
            testOnBorrow="true" validationQuery="SELECT 1 FROM DUAL"/>
</Context>
```
- **JNDI name change:** JBoss `java:/jdbc/AppDS` → Tomcat lookup is
  `java:comp/env/jdbc/AppDS`. Update every `InitialContext` lookup and
  `@Resource(lookup=...)` / `persistence.xml` `<jta-data-source>` accordingly.
- **JDBC driver placement:** put the driver jar in `$CATALINA_BASE/lib` (shared) OR
  bundle in the WAR. If the `Resource` is defined in `context.xml`, the driver must
  be on the **container** classpath (`lib/`), not the WAR.
- Add `<resource-ref>` in `web.xml` for the same name.

### 4.2 Transactions (JTA)

JBoss had a built-in transaction manager. **Tomcat has none.** Options, in order of
preference:
- **Spring apps:** use Spring's `DataSourceTransactionManager` (single resource) — no
  JTA needed. This covers the vast majority of apps.
- **JPA without Spring:** use Hibernate's `RESOURCE_LOCAL` transactions; change
  `persistence.xml` `transaction-type="RESOURCE_LOCAL"` and use a
  `<non-jta-data-source>`.
- **Genuine XA / multi-resource (DB + MQ in one tx):** embed **Atomikos** (or Narayana
  standalone). Flag this loudly in `MIGRATION_NOTES.md` — it needs human review.

### 4.3 Security

- `jboss-web.xml` security-domain → Tomcat `Realm` (in `context.xml` or `server.xml`),
  or app-level Spring Security.
- Map declarative roles in `web.xml` `<security-constraint>` (these carry over) to a
  Tomcat Realm (`UserDatabaseRealm`, `DataSourceRealm`, `JNDIRealm`/LDAP).

### 4.4 Things to delete

- `jboss-deployment-structure.xml`, `jboss-web.xml`, `*-ds.xml`, `jboss-ejb3.xml`,
  any `standalone.xml` fragments. Capture their intent in the Tomcat equivalents first.

---

## 5. IBM MQ connectivity mapping (JMS)

This is the highest-risk part. On JBoss, IBM MQ was wired via the **WMQ resource
adapter (`wmq.jakarta.jmsra.rar` / `wmq.jmsra.rar`)** with the ConnectionFactory and
destinations defined as JNDI admin objects in `standalone.xml`. **Tomcat has no
resource-adapter container.** Replace the RA with the **IBM MQ JMS client library**
configured directly.

### 5.1 Use the Jakarta MQ client

Because Tomcat 10 is `jakarta.jms`, you **must** use the Jakarta build of the MQ
client. Using the old `javax` client (`com.ibm.mq.allclient`) will fail to bind.
- Pull `com.ibm.mq:com.ibm.mq.jakarta.client` from Nexus.
- ConnectionFactory class becomes `com.ibm.mq.jakarta.jms.MQConnectionFactory`
  (note the `jakarta` segment).

### 5.2 Map the JBoss RA config → Tomcat

Capture these from the JBoss `standalone.xml` MQ subsystem / RA config and carry
them onto the ConnectionFactory:

| JBoss RA property        | IBM MQ ConnectionFactory setter        |
|--------------------------|----------------------------------------|
| `hostName` / `connName`  | `setHostName` / `setConnectionNameList`|
| `port`                   | `setPort`                              |
| `queueManager`           | `setQueueManager`                      |
| `channel`                | `setChannel`                           |
| `transportType`          | `setTransportType` (`1` = CLIENT)      |
| `ccsid`                  | `setCCSID`                             |
| `userName` / `password`  | app-server credential → MQ CSP / app config |
| SSL `cipherSuite`        | `setSSLCipherSuite` + JVM truststore   |

### 5.3 Two wiring options

**Option A — Spring-managed (preferred for Spring/Spring Boot apps):**
```java
@Bean
public ConnectionFactory mqConnectionFactory() throws JMSException {
    MQConnectionFactory cf = new MQConnectionFactory(); // com.ibm.mq.jakarta.jms
    cf.setHostName(env("MQ_HOST"));
    cf.setPort(Integer.parseInt(env("MQ_PORT")));
    cf.setQueueManager(env("MQ_QMGR"));
    cf.setChannel(env("MQ_CHANNEL"));
    cf.setTransportType(WMQConstants.WMQ_CM_CLIENT);
    return cf;
}
```
Wrap with `CachingConnectionFactory`; drive listeners via Spring
`DefaultMessageListenerContainer` / `@JmsListener`. **MDBs (`@MessageDriven`) do not
exist on Tomcat** — convert each MDB into a Spring `@JmsListener` / listener container.

**Option B — Tomcat JNDI resource (for plain Java EE apps that do `lookup`):**
Define the CF as a JNDI `Resource` in `context.xml` so existing
`InitialContext().lookup("java:comp/env/jms/MyCF")` code keeps working:
```xml
<Resource name="jms/MyCF" auth="Container"
          factory="com.ibm.mq.jakarta.jms.MQConnectionFactoryFactory"
          type="com.ibm.mq.jakarta.jms.MQConnectionFactory"
          HOST="mq-host" PORT="1414" QMGR="QM1" CHAN="DEV.APP.SVRCONN"
          TRAN="CLIENT"/>
<Resource name="jms/MyQueue" auth="Container"
          factory="com.ibm.mq.jakarta.jms.MQQueueFactory"
          type="com.ibm.mq.jakarta.jms.MQQueue"
          QU="DEV.QUEUE.1"/>
```
Update the JNDI names from JBoss `java:/jms/...` to Tomcat `java:comp/env/jms/...`
and add matching `<resource-ref>` / `<resource-env-ref>` in `web.xml`.

### 5.4 MQ checklist per app
- [ ] All `javax.jms` → `jakarta.jms`.
- [ ] MQ client = the **jakarta** artifact from Nexus.
- [ ] Every MDB rewritten as a Spring listener (no MDB support on Tomcat).
- [ ] QM, channel, host, port, CCSID, SSL cipher carried from `standalone.xml`.
- [ ] JNDI lookup names re-prefixed to `java:comp/env/...`.
- [ ] XA? If MQ + DB were in one JTA tx on JBoss, see Section 4.2 (Atomikos) and flag.

---

## 6. Tomcat descriptors to add

Per app under `tomcat/<app>/src/main/webapp/`:
- `META-INF/context.xml` — datasource + JMS JNDI resources (Sections 4.1, 5.3B).
- `WEB-INF/web.xml` — bumped to Servlet 5.0 namespace; `<resource-ref>` /
  `<resource-env-ref>` entries; security constraints carried from `jboss-web.xml`.
- Externalize host/credentials via environment or `$CATALINA_BASE/conf` properties —
  **do not hardcode secrets** in `context.xml` checked into the repo.

---

## 7. Build, deploy, verify

```bash
# from tomcat/<app>
mvn -s ~/.m2/settings.xml clean package        # resolves from Nexus, builds WAR
# deploy
cp target/<app>.war $CATALINA_BASE/webapps/
$CATALINA_BASE/bin/catalina.sh run             # or .bat on Windows
```
Smoke test:
- App deploys with no `ClassNotFound` / `NoClassDefFound` (usually a missing bundled
  impl from Section 3.2) and no `javax/jakarta` linkage errors.
- Datasource JNDI lookup succeeds.
- MQ: send/receive a test message; confirm connection to the queue manager.

---

## 8. Per-app `MIGRATION_NOTES.md` template

```markdown
# <app> — JBoss → Tomcat migration notes
- Source: <path>   Target: tomcat/<app>
- Framework: Spring Boot 3 / plain Java EE
- Namespace: javax→jakarta applied (Y/N), files touched: N
- Datasource(s): JBoss jndi → Tomcat jndi (java:comp/env/...)
- Transactions: Spring tx / RESOURCE_LOCAL / Atomikos-XA
- IBM MQ: CF mapping, MDBs converted to listeners (count)
- Libraries added (all from Nexus): <list with versions>
- Open TODOs needing human review: <list>
```

---

## 9. Guardrails for the agent

- **Never** resolve dependencies from public Maven Central — Nexus only. If missing
  in Nexus, stop and report.
- **Never** hardcode credentials, MQ passwords, or DB passwords in committed files.
- **Never** edit the original JBoss sources — only the `tomcat/` copy.
- **Never** invent library versions; use what Nexus publishes (ask if unsure).
- If an app uses EJBs, JTA/XA, MDBs, or a JBoss-specific feature with no clean Tomcat
  equivalent, **flag it for human review** instead of guessing.
- Keep each app's change set reviewable; one app at a time.
