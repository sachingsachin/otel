# otel
Otel stands for [Open Telemetry](https://opentelemetry.io/) which is a way to collect telemetry info in a standard way from different kinds of applications.  
Telemetry info means metrics, logging and traces.

# Brief Architecture
OTEL first helps in capturing the telemetry data and then it has tools to export that data to other tools like Jaeger and Prometheus.

Main idea is to become a one-stop place for all telemetry collection tools and libraries. So they not only have standardized APIs but also actively integrating with both sides - [sources](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md) and telemetry consumers.

For collection, OTEL has 2 modes:  
1. Zero Code Instrumentation
2. Spring Boot Starter (With Code Instrumentation)
   
## 1. Zero code instrumentation
[Zero code instrumentation](https://opentelemetry.io/docs/zero-code/java/) is useful when you cannot or do not want to change application code for emitting metrics.  
In such a case, the `opentelemetry-javaagent.jar` is used with the Java command line option `-javaagent`
   
A Java Agent is a special kind of JAR file that can intercept, modify, and instrument classes at runtime before the JVM loads them.  
It uses the Java Instrumentation API (java.lang.instrument) under the hood.  
Agents are often used for: Profiling (e.g., YourKit, JProfiler), Monitoring (e.g., New Relic, Datadog, AppDynamics) and Code coverage (e.g., JaCoCo, Cobertura) etc.

When the JVM starts, it looks for a class inside the agent JAR with a special method `premain` which is called before main and can be used to register class transformers, Modify bytecode of classes before they are loaded and add monitoring hooks, loggers, metrics exporters etc.

```java
public static void premain(String agentArgs, Instrumentation inst)
```
   
Note that this kind of telemetry collection is not in-depth app-specific instrumentation. Rather it captures telemetry data only at the “edges” of an app or service, such as inbound requests, outbound HTTP calls, database calls, and so on.

For a maven project, you can edit pom.xml file to specify javaagent as:
   ```xml
         <commandlineArgs>
          -javaagent:${project.basedir}/opentelemetry-javaagent.jar
         </commandlineArgs>
   ```
Or use MAVEN_OPTS env var:  
   ```
   export MAVEN_OPTS="-javaagent:/full/path/to/opentelemetry-javaagent.jar"
   ```
   
You can also use [annotations](https://opentelemetry.io/docs/zero-code/java/agent/annotations/) `@WithSpan` and `@SpanAttribute` in your code (while using the above agent) to further customise the telemetry collection.

## 2. Spring Boot Starter (With Code Instrumentation)  
[The with-code option](https://opentelemetry.io/docs/zero-code/java/spring-boot-starter/) has to be used when the java agent option cannot be used. That is when the java agent is found to be consuming too many resources or the underlying app is already using an agent.  
Once you have the proper maven dependencies, then you can get OTL instances to create your own tracers, spans and monitor objects.


# Collector
[Collector](https://opentelemetry.io/docs/collector/) is a stand-alone tool that can receive telemetry data, process it and then feed to other tools like Jaeger, Prometheus, Fluent Bit, etc.
While a collector process is not strictly needed, and small dev projects can do without a collector, its strongly recommended to have one for production and critical uses because collector
improves the scalability, performance and stability of receiving and exporting data. It also supports retries, encryption, batching and sensitive data filtering.

The structure of any Collector configuration file consists of four classes of pipeline components that access telemetry data:
1. Receivers 
2. Processors 
3. Exporters 
4. Connectors : These join two pipelines, acting as both exporter and receiver. A connector consumes data as a receiver at the end of one pipeline and emits data as an exporter at the beginning of another pipeline

   

# Some other related concepts

## MDC

MDC stands for Mapped Diagnostic Context used in logging frameworks like Log4j, SLF4J, or Logback in Java.  
MDC lets you attach contextual information (like userId, requestId, sessionId) to the current thread of execution.  
That info is then automatically included in all log messages from that thread, without having to pass the values explicitly everywhere.

Example:

```java
import org.slf4j.MDC;

public class Example {
    public void processRequest(String requestId) {
        MDC.put("requestId", requestId); // Add context
        try {
            log.info("Starting processing"); 
            // Logs will now include requestId
        } finally {
            MDC.clear(); // Clean up after thread finishes
        }
    }
}
```


Log output (with `%X{requestId}` in the log pattern specified in log4j2.xml):

```
2025-09-04 12:00:00 INFO  [requestId=abc123] Starting processing
```

🚦 Why it’s useful

Adds traceability across logs, especially in multi-threaded or distributed systems.

Helps correlate logs for a single request in microservices.

Works well with correlation IDs in observability setups.

👉 Sometimes MDC is paired with NDC (Nested Diagnostic Context), which tracks nested contexts like call stacks.


## NDC (Nested Diagnostic Context)

While MDC is a map of key–value pairs, NDC is essentially a stack of contextual messages.  
You push context information when entering a certain scope (e.g., method, request, transaction).  
You pop it when leaving.  

Each log message will include the entire context stack, so logs show nested scopes.

```java
import org.apache.log4j.NDC;

public class Example {
    public void processOrder(String orderId) {
        NDC.push("orderId=" + orderId); // Enter context
        try {
            log.info("Start processing");
            validate(orderId);
            ship(orderId);
        } finally {
            NDC.pop(); // Leave context
            NDC.remove(); // Clean up
        }
    }
}
```

If the pattern layout is:
%d %-5p %c %x - %m%n

Example log output:
```
2025-09-04 12:00:00 INFO  com.example.OrderService [orderId=123] - Start processing
2025-09-04 12:00:01 INFO  com.example.OrderService [orderId=123] - Validation passed
2025-09-04 12:00:02 INFO  com.example.OrderService [orderId=123] - Shipping started
```

Here %x means “current NDC stack”  
MDC is usually simpler and more common today.


## Attaching attributes to transactions is more efficient and easier to read than MDC

**1. With MDC (classic logging)**

You’d add the attribute into MDC:

```java
MDC.put("securitymode", "strict");
log.info("User logged in");
log.info("Fetching orders");
log.info("Returning response");
MDC.clear();
```

Log output:
```
2025-09-04 12:00:00 INFO [securitymode=strict] User logged in
2025-09-04 12:00:01 INFO [securitymode=strict] Fetching orders
2025-09-04 12:00:02 INFO [securitymode=strict] Returning response
```

✅ Each line shows securitymode=strict.  
❌ But it’s duplicated on every line → log volume grows with number of log entries.

**2. With Transaction Attributes (APM/Tracing API)**

You attach the attribute to the transaction/trace:

```java
Transaction txn = txnMarkingService.currentTransaction();
txn.addAttribute("securitymode", "strict");

log.info("User logged in");
log.info("Fetching orders");
log.info("Returning response");
```

Log output (lighter):
```
2025-09-04 12:00:00 INFO User logged in
2025-09-04 12:00:01 INFO Fetching orders
2025-09-04 12:00:02 INFO Returning response
```

Tracing/observability UI (Jaeger, New Relic, etc.):  
Transaction: /api/getOrders  
Attributes:  
   securitymode = strict  
   order-id     = 12345  
   user-id      = alice  


✅ Attribute shown once at the transaction level.  
✅ Automatically applies to all spans/logs in this transaction.  
✅ Easy to filter/search: e.g., show me all transactions where securitymode=strict.  
✅ Saves log storage (no repetition).

Summary:
1. Use MDC if your system is only using logs (no tracing/metrics).
2. Use Transaction attributes if you have a tracing/APM system — it gives you richer context and less noisy logs.

