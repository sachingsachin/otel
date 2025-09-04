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




   
