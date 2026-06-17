Got it. The key constraint: **JBoss/WildFly 8 (javax, Java EE 7) → Tomcat 10 (jakarta, Jakarta EE 9+)** means a mandatory `javax.*` → `jakarta.*` namespace migration, and Tomcat won't provide the container services JBoss did (JTA, JNDI datasources, the IBM MQ resource adapter, bundled Hibernate/Weld/JSF). I'll write the playbook around those facts.

I'll create `AGENT.md` at the root. There's a Nexus URL placeholder you'll fill in.

Created [AGENT.md](AGENT.md). Let me note this migration initiative to memory for future sessions.

Done. I created [AGENT.md](AGENT.md) — a per-app migration playbook tuned to your exact stack (JBoss 8 → Tomcat 10, Maven, Spring Boot + plain Java EE, IBM MQ, WAR on standalone Tomcat).

**What it covers:**
- **Section 1** — ordered per-app procedure: copy each app into a parallel `tomcat/<app>/` tree mirroring the source structure (never edits JBoss in place).
- **Section 2** — the mandatory `javax.*` → `jakarta.*` namespace migration (Tomcat 10 is Jakarta EE 9), with a full rename table, the "do NOT rename" JDK packages, and the Spring 6 / Boot 3 + JDK 17 requirement.
- **Section 3** — Maven/Nexus: `settings.xml` mirror config pointing all resolution at internal Nexus, and a full list of required libraries (Jakarta APIs, Hibernate, Weld, Mojarra, pooling, JDBC, the Jakarta IBM MQ client) with a hard rule to never pull from Maven Central.
- **Section 4** — mapping JBoss container services Tomcat doesn't provide: JNDI datasources, transactions (no JTA on Tomcat), pooling, security realms.
- **Section 5** — the IBM MQ mapping: replacing the JBoss WMQ resource adapter with the **jakarta** MQ client (`com.ibm.mq.jakarta.client`), a `standalone.xml`-property → `MQConnectionFactory`-setter table, Spring `@JmsListener` vs. Tomcat-JNDI wiring, and converting MDBs (no MDB support on Tomcat).
- **Sections 6–9** — Tomcat descriptors, build/deploy/verify steps, a `MIGRATION_NOTES.md` template, and guardrails.

**Two things to fill in / decide:**
1. **Nexus URL** — `AGENT.md` uses a `NEXUS_BASE` placeholder; drop in your real internal URL.
2. Note: this working dir is your observability app, not the JBoss apps — point the agent at the actual app repo when you run it.

Want me to also generate the concrete `settings.xml` and a sample migrated `pom.xml` / `context.xml` once you give me one real app's `standalone.xml` and current `pom.xml`?
