# Experimental OpenTelemetry Java Agent 2.32.0 compatibility for IBM J9

This branch is based on upstream OpenTelemetry Java Instrumentation **v2.32.0**, not the fork's moving `main` branch.

## Goal
Test whether WAS Traditional **8.5.5.14**, IBM Java 8 **SR5 FP17**, can start without a native SIGSEGV in `InstrumentationImpl.retransformClasses` / `libj9jvmti29.so`.

## Change
A minimal opt-in switch is added in `AgentInstaller`:

```properties
-Dotel.experimental.ibmj9.disable-retransformation=true
```

Default is **false**, preserving upstream retransformation. With the flag set, the main Byte Buddy agent builder switches to `RedefinitionStrategy.DISABLED` immediately before `installOn(inst)`. The existing early instrumentation remains unchanged.

**This is a POC, not a proven workaround.** Already loaded classes are not retransformed and Servlet/JDBC tracing may be missing or incomplete. Other JVMTI calls and JVM bugs are still possible. Do not deploy to production until validated.

## UAT plan
1. Remove AppDynamics and the previous OTel Java agent. Temporarily remove the JMX Exporter for an isolated run.
2. Build this branch on a development machine with **JDK 25** (per upstream v2.32.0 CONTRIBUTING), using `./gradlew assemble`. The resulting agent targets Java 8+.
3. Deploy the resulting agent JAR to UAT, not a stock v2.32.0 JAR. Confirm the warning containing `Experimental IBM J9 compatibility mode` appears in logs.
4. Start with `-Dotel.traces.exporter=none -Dotel.metrics.exporter=none -Dotel.logs.exporter=none` to isolate startup from collector connectivity.
5. After stable startup, configure OTLP/HTTP to Collector port 4318; generate Servlet requests involving DB2 JDBC queries.
6. Verify same trace ID contains HTTP SERVER and JDBC CLIENT spans; check gaps, performance, dumps, and rollback.

## Example WAS Generic JVM Arguments
```properties
-javaagent:/opt/otel-custom/opentelemetry-javaagent.jar
-Dotel.experimental.ibmj9.disable-retransformation=true
-Dotel.service.name=was-mobile-fabric-uat
-Dotel.traces.exporter=none
-Dotel.metrics.exporter=none
-Dotel.logs.exporter=none
```

## Rollback
Remove the custom `-javaagent` JVM argument, restore previous known-good configuration, restart WAS.

## Upstream
https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/v2.32.0

License: Apache-2.0; upstream notices and headers preserved.
