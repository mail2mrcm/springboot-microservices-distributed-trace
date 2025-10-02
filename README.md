# springboot-microservices-distributed-trace
- Trace a request across multiple services.
- Each service appends trace IDs/span IDs.
- Tools: OpenTelemetry,  Zipkin.
- Example: Request → API Gateway →Student Service→ Payment Service
  - Trace shows latency at each hop.
- Note - Please refer repo https://github.com/mail2mrcm/springboot-microservices-distributed-trace.git  for  details.
	
Integration Micrometer and Zipkin in microservice
- Micrometer Tracing → collects trace info and enriches logs.
- Brave Bridge → actual tracing library that integrates with Zipkin.
- Zipkin Reporter → exports spans to Zipkin over HTTP.
- Zipkin Server → stores and visualizes traces.
- RestTemplate/WebClient interceptors → propagate context (traceId/spanId) between services.

Step 1 - Download and Run Zipkin JAR
# download the latest zipkin.jar
curl -sSL https://zipkin.io/quickstart.sh | bash -s

# run zipkin on port 9411 with JMX disabled (to avoid that RMI port conflict)
java -Dcom.sun.management.jmxremote=false -jar zipkin.jar

Note- I have created /Zipkin folder and copied binaries of zipkin into the same for  convenience and directly jar can be executed by navigation /Zipkin

By default, Zipkin starts at:
📍 http://localhost:9411

Step 2 - Add Dependencies in each microservice

```
<dependencies>
  <!-- Actuator enables tracing & metrics -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <!-- Micrometer Tracing using Brave -->
  <dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
  </dependency>

  <!-- Reporter to send spans to Zipkin -->
  <dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
  </dependency>
</dependencies>
```
• actuator: without it, tracing won’t be activated.
• micrometer-tracing-bridge-brave: hooks into Spring Boot, instruments RestTemplate/WebClient, injects trace IDs in logs.
• zipkin-reporter-brave: knows how to send spans to Zipkin server.

Step 3 -Configure Application

```
spring:
  application:
    name: student-service   # or payment-service

management:
  tracing:
    sampling:
      probability: 1.0   # 100% traces for dev
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```
zipkin.tracing.endpoint: where spans are exported.

Step 4 -Ensure Context Propagation Between Services

```
@Bean
RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder.build(); // auto-instrumented
}
```
Don’t new RestTemplate() yourself — headers won’t propagate.

Step 5 -Modify logback.xml for supporting trace id and span id in log files.

```
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder>
   <pattern>
       %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %5level traceId=%X{traceId:-} spanId=%X{spanId:-} %logger{36} - %msg%n
    </pattern>        
  </encoder>
</appender>
```
Step 6 - Verify in Logs & Zipkin
	• Start Zipkin.
	• Start microservices.(student-service + payment-service).
	• Call student-service endpoint that internally calls payment-service.
	• In logs → both services should share the same traceId.

Student-Service

```ruby
2025-09-16 23:07:47.649 [http-nio-auto-1-exec-2] DEBUG traceId=68c9dfb3017ed9070c62460552000eca spanId=0c62460552000eca c.h.s.controller.StudentController - Start of findById
2025-09-16 23:07:48.041 [http-nio-auto-1-exec-2] DEBUG traceId=68c9dfb3017ed9070c62460552000eca spanId=0c62460552000eca c.h.s.controller.StudentController - End of findById
```
			
Payment-Service

```ruby
2025-09-16 23:07:47.807 [http-nio-auto-1-exec-2]  INFO traceId=68c9dfb3017ed9070c62460552000eca spanId=411a64fa309e59f9 c.h.p.controller.PaymentController - Start of getPaymentDetailsByStudentId
2025-09-16 23:07:47.987 [http-nio-auto-1-exec-2]  INFO traceId=68c9dfb3017ed9070c62460552000eca spanId=411a64fa309e59f9 c.h.p.controller.PaymentController - End of getPaymentDetailsByStudentId
```		
Go to http://localhost:9411 → search by student-service or traceId → you’ll see a timeline of spans across both services.<img width="1138" height="3420" alt="image" src="https://github.com/user-attachments/assets/16ce2aa3-1a45-4044-a5c0-aa02cd693e39" />
