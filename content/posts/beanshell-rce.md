---
title: "Validation Endpoint, to Production Credentials"
date: 2026-10-26
draft: false
slug: "beanshell-rce"
tags: ["rce", "beanshell", "hybris", "bug-bounty", "blind-rce", "ssrf"]
summary: "An unauthenticated BeanShell evaluation endpoint on a production SAP Commerce Cloud instance turned a one-bit boolean oracle into the database connection string, username, and password. CVSS 10.0."
---

# **TLDR**
Superdrug's SAP Commerce Cloud production server exposed an unauthenticated form-validation endpoint that accepted a user-chosen scripting language and an arbitrary expression to evaluate. Setting `languages=BEANSHELL` gave full Java evaluation. No authentication, no session, no CSRF token.

The response differed by one observable bit: a `true` expression returned `[]`, a `false` expression echoed the expression back. That's a boolean oracle. Using `Config.getParameter()` and a binary search over character ASCII values, I extracted the full production Azure SQL Server connection string, username, and 34-character password in about 1,400 requests.

The credentials were valid, meaning a direct connection passed Azure's credential check and was only rejected because of an IP firewall rule, which was only a temporary fix on their end. A companion endpoint (`/validate-dynamic`) was still leaking internal Spring bean class types and loading arbitrary classes from the classpath.

CVSS 10.0. Disclosed accordingly.

***

# **Boolean Oracle to Full Compromise**

Superdrug's SAP Commerce Cloud deployment exposed a form-validation endpoint that accepted a scripting language as a query parameter. The API specification at `/api/v2/api-docs` listed it in full: `/api/v2/sd/forms/external-validator/validate`, with three parameters: `targets` for a Java class name, `expressions` for the validation logic, and `languages` for the scripting engine.

A public endpoint where the caller selects the language and supplies the expression string is a code-execution primitive. The framing as "validation" is what obscures it. The intended design is presumably that an administrator configures rules like "this field must match this regex blabla bla" and the platform evaluates them server-side. That design assumes the person choosing the language is trusted... The endpoint itself enforces no such assumption.

A minimal test confirmed the primitive: an arithmetic comparison as the expression, with the language set to BEANSHELL. Here I handed it the expression `7*7==49`, which URL-encoded looks like this: `7*7%3D%3D49`

```http
POST /api/v2/sd/forms/external-validator/validate?targets=[redacted-target-web-service]&expressions=7*7%3D%3D49&languages=BEANSHELL HTTP/1.1
Host: app.superdrug.com
Content-Type: application/json
User-Agent: okhttp/4.9.3

{"uid":"beeeep, testing"}

HTTP/1.1 200 OK
Content-Type: application/json

[]
```
In this framework, an empty array like this means "no validation violations" — the expression evaluated true.
When I changed the expression from `49` to `50`:

```http
POST /api/v2/sd/forms/external-validator/validate?targets=de.hybris.platform.commercewebservicescommons.dto.user.UserWsDTO&expressions=7*7%3D%3D50&languages=BEANSHELL HTTP/1.1
Host: app.superdrug.com
Content-Type: application/json
User-Agent: okhttp/4.9.3

{"uid":"test"}

HTTP/1.1 200 OK
Content-Type: application/json

["7*7==50"]
```

This time the expression was echoed back, the failure case. The framework returns the text of the expression that failed validation. The response is therefore a clean boolean oracle: `[]` means the expression evaluated true, while `["..."]` means it evaluated false.

Removing the Authorization header entirely and repeating both requests gave identical results. The endpoint required no authentication of any kind. Not just a weak check that might be bypassed, but no check at all... That means any unauthenticated caller can ask the production application server a yes/no question about anything in its process memory.

That combination of caller-controlled Java expression, reliable boolean response, and no authentication is sufficient to exfiltrate anything the Java process can reach. So the remainder of the finding was a matter of establishing what it *could* reach.

***

BeanShell is a Java-like scripting language, and the relevant property is that it evaluates expressions with the standard library and the application classpath in scope.

```http
expressions=Integer.parseInt(%22123%22)%3D%3D123&languages=BEANSHELL  ->  []
```

The BeanShell context was the application process itself, not a sandboxed helper or a subprocess. It was the process that served requests.

The next step was reaching the application's own classes. SAP Commerce ships `de.hybris.platform.util.Config`, which exposes the platform's full property configuration through `getParameter()`. The configuration store was also readable through the same evaluation context, which meant the database password could be pulled one boolean at a time.

First I enumerated the length and format.

```http
de.hybris.platform.util.Config.getParameter("db.url").startsWith("[redacted]")       -> []
de.hybris.platform.util.Config.getParameter("db.password").length()==34              -> []
```
Once both were true, it took around 1,400 requests to produce a working database password, and only about 2,000 to extract the full set:

```txt
db.driver = com.microsoft.sqlserver.jdbc.SQLServerDriver
db.url    = jdbc:sqlserver://[redacted].database.windows.net:1433;database=[redacted];loginTimeout=4
db.username = [redacted] (12 characters)
db.password = [redacted] (34 characters)
```

These were live production credentials for an Azure SQL Server instance. A successful authentication against that server means read and write access to whatever the application database holds. For a retail pharmacy platform that's customer records, order history, and account data, not just configuration.

The only thing that stopped a direct connection was an Azure firewall rule restricting source IPs. That is worth being precise about: the credentials themselves were accepted. Azure validated the username and password and *then* rejected the session on the IP rule. A network allowlist is a filter sitting in front of a door that's already unlocked. Any allowed source, a future rule change, or any other path into the trusted range re-opens it. It bought time; it did not fix anything.

The bug was never BeanShell specifically. BeanShell was just the most convenient engine to reach. The root cause was an endpoint that let the caller choose the evaluation language *and* supply the expression. No scripting engine is safe under that design. Swapping BeanShell for another supported language changes the syntax, not the outcome.

Three things had to be true at once for this to reach CVSS 10.0, and each is its own fix:

**No authentication.** The endpoint answered anonymous callers. It should never have been reachable without a session.

**Caller-chosen language.** "Which scripting language should the server execute?" is not a user input. Validation rules belong server-side, fixed at deploy time.

**Full classpath in scope.** The evaluation context could see `Config` and the application's own classes. Expression evaluation, if it exists at all, should run with nothing reachable that isn't strictly needed.

And `/validate-dynamic` needs the same review. Leaking Spring bean types and loading arbitrary classpath classes is the same class of bug, one chain away from the same result.

The interesting part was never the trick. 
It was how ordinary "validation" framing hid a full code-execution primitive in plain sight on a production server, behind a name that sounds boring enough to skip in a review. One boolean. ~1,400 questions. The entire connection string.

