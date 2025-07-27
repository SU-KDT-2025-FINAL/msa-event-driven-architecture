# 성능 테스트 및 최적화 가이드

## 1. JMeter 성능 테스트

### 기본 테스트 계획 구성
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.5">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="E-commerce API Performance Test">
      <elementProp name="TestPlan.arguments" elementType="Arguments" guiclass="ArgumentsPanel">
        <collectionProp name="Arguments.arguments">
          <elementProp name="host" elementType="Argument">
            <stringProp name="Argument.name">host</stringProp>
            <stringProp name="Argument.value">${__P(host,localhost)}</stringProp>
          </elementProp>
          <elementProp name="port" elementType="Argument">
            <stringProp name="Argument.name">port</stringProp>
            <stringProp name="Argument.value">${__P(port,8080)}</stringProp>
          </elementProp>
          <elementProp name="threads" elementType="Argument">
            <stringProp name="Argument.name">threads</stringProp>
            <stringProp name="Argument.value">${__P(threads,100)}</stringProp>
          </elementProp>
          <elementProp name="duration" elementType="Argument">
            <stringProp name="Argument.name">duration</stringProp>
            <stringProp name="Argument.value">${__P(duration,300)}</stringProp>
          </elementProp>
        </collectionProp>
      </elementProp>
    </TestPlan>

    <hashTree>
      <!-- Thread Group for Load Testing -->
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Load Test">
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <intProp name="LoopController.loops">-1</intProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">${threads}</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <longProp name="ThreadGroup.start_time">1</longProp>
        <longProp name="ThreadGroup.end_time">1</longProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
        <stringProp name="ThreadGroup.duration">${duration}</stringProp>
        <stringProp name="ThreadGroup.delay">0</stringProp>
      </ThreadGroup>

      <hashTree>
        <!-- HTTP Request Defaults -->
        <ConfigTestElement guiclass="HttpDefaultsGui" testclass="ConfigTestElement" testname="HTTP Request Defaults">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments"/>
          <stringProp name="HTTPSampler.domain">${host}</stringProp>
          <stringProp name="HTTPSampler.port">${port}</stringProp>
          <stringProp name="HTTPSampler.protocol">http</stringProp>
          <stringProp name="HTTPSampler.contentEncoding">UTF-8</stringProp>
          <stringProp name="HTTPSampler.path"></stringProp>
        </ConfigTestElement>

        <!-- User Authentication -->
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Login">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments">
              <elementProp name="username" elementType="HTTPArgument">
                <boolProp name="HTTPArgument.always_encode">false</boolProp>
                <stringProp name="Argument.value">testuser${__threadNum}</stringProp>
                <stringProp name="Argument.metadata">=</stringProp>
                <stringProp name="Argument.name">username</stringProp>
              </elementProp>
              <elementProp name="password" elementType="HTTPArgument">
                <boolProp name="HTTPArgument.always_encode">false</boolProp>
                <stringProp name="Argument.value">password123</stringProp>
                <stringProp name="Argument.metadata">=</stringProp>
                <stringProp name="Argument.name">password</stringProp>
              </elementProp>
            </collectionProp>
          </elementProp>
          <stringProp name="HTTPSampler.domain"></stringProp>
          <stringProp name="HTTPSampler.port"></stringProp>
          <stringProp name="HTTPSampler.protocol"></stringProp>
          <stringProp name="HTTPSampler.contentEncoding"></stringProp>
          <stringProp name="HTTPSampler.path">/auth/login</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
          <boolProp name="HTTPSampler.auto_redirects">false</boolProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
          <boolProp name="HTTPSampler.DO_MULTIPART_POST">false</boolProp>
        </HTTPSamplerProxy>

        <hashTree>
          <!-- JSON Extractor for JWT Token -->
          <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="Extract JWT Token">
            <stringProp name="JSONPostProcessor.referenceNames">jwt_token</stringProp>
            <stringProp name="JSONPostProcessor.jsonPathExprs">$.token</stringProp>
            <stringProp name="JSONPostProcessor.match_numbers">1</stringProp>
            <stringProp name="JSONPostProcessor.defaultValues">INVALID_TOKEN</stringProp>
          </JSONPostProcessor>
        </hashTree>

        <!-- Main Test Scenarios -->
        <RandomController guiclass="RandomControlGui" testclass="RandomController" testname="User Journey">
          <intProp name="InterleaveControl.style">1</intProp>
        </RandomController>

        <hashTree>
          <!-- Browse Products -->
          <ThroughputController guiclass="ThroughputControllerGui" testclass="ThroughputController" testname="Browse Products (40%)">
            <intProp name="ThroughputController.style">1</intProp>
            <stringProp name="ThroughputController.perThread">false</stringProp>
            <floatProp name="ThroughputController.maxThroughput">40.0</floatProp>
          </ThroughputController>

          <hashTree>
            <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Get Products">
              <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                <collectionProp name="Arguments.arguments">
                  <elementProp name="page" elementType="HTTPArgument">
                    <stringProp name="Argument.name">page</stringProp>
                    <stringProp name="Argument.value">${__Random(0,10)}</stringProp>
                  </elementProp>
                  <elementProp name="size" elementType="HTTPArgument">
                    <stringProp name="Argument.name">size</stringProp>
                    <stringProp name="Argument.value">20</stringProp>
                  </elementProp>
                </collectionProp>
              </elementProp>
              <stringProp name="HTTPSampler.path">/products</stringProp>
              <stringProp name="HTTPSampler.method">GET</stringProp>
            </HTTPSamplerProxy>

            <hashTree>
              <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager">
                <collectionProp name="HeaderManager.headers">
                  <elementProp name="Authorization" elementType="Header">
                    <stringProp name="Header.name">Authorization</stringProp>
                    <stringProp name="Header.value">Bearer ${jwt_token}</stringProp>
                  </elementProp>
                </collectionProp>
              </HeaderManager>
              
              <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion">
                <collectionProp name="Asserion.test_strings">
                  <stringProp>200</stringProp>
                </collectionProp>
                <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
                <boolProp name="Assertion.assume_success">false</boolProp>
                <intProp name="Assertion.test_type">1</intProp>
              </ResponseAssertion>
            </hashTree>
          </hashTree>

          <!-- Create Order -->
          <ThroughputController guiclass="ThroughputControllerGui" testclass="ThroughputController" testname="Create Order (30%)">
            <intProp name="ThroughputController.style">1</intProp>
            <stringProp name="ThroughputController.perThread">false</stringProp>
            <floatProp name="ThroughputController.maxThroughput">30.0</floatProp>
          </ThroughputController>

          <hashTree>
            <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="Create Order">
              <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
              <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                <collectionProp name="Arguments.arguments">
                  <elementProp name="" elementType="HTTPArgument">
                    <boolProp name="HTTPArgument.always_encode">false</boolProp>
                    <stringProp name="Argument.value">{
  "items": [
    {
      "productId": "product-${__Random(1,100)}",
      "quantity": ${__Random(1,5)},
      "price": ${__Random(10,100)}.99
    }
  ],
  "shippingAddress": {
    "street": "123 Test St",
    "city": "Test City",
    "zipCode": "12345"
  },
  "paymentMethod": "CREDIT_CARD"
}</stringProp>
                    <stringProp name="Argument.metadata">=</stringProp>
                  </elementProp>
                </collectionProp>
              </elementProp>
              <stringProp name="HTTPSampler.path">/orders</stringProp>
              <stringProp name="HTTPSampler.method">POST</stringProp>
            </HTTPSamplerProxy>

            <hashTree>
              <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager">
                <collectionProp name="HeaderManager.headers">
                  <elementProp name="Authorization" elementType="Header">
                    <stringProp name="Header.name">Authorization</stringProp>
                    <stringProp name="Header.value">Bearer ${jwt_token}</stringProp>
                  </elementProp>
                  <elementProp name="Content-Type" elementType="Header">
                    <stringProp name="Header.name">Content-Type</stringProp>
                    <stringProp name="Header.value">application/json</stringProp>
                  </elementProp>
                </collectionProp>
              </HeaderManager>
            </hashTree>
          </hashTree>
        </hashTree>

        <!-- Think Time -->
        <GaussianRandomTimer guiclass="GaussianRandomTimerGui" testclass="GaussianRandomTimer" testname="Think Time">
          <stringProp name="ConstantTimer.delay">2000</stringProp>
          <stringProp name="RandomTimer.range">1000.0</stringProp>
        </GaussianRandomTimer>
      </hashTree>

      <!-- Listeners for Results -->
      <ResultCollector guiclass="ViewResultsFullVisualizer" testclass="ResultCollector" testname="View Results Tree"/>
      <ResultCollector guiclass="SummaryReport" testclass="ResultCollector" testname="Summary Report"/>
      <ResultCollector guiclass="StatVisualizer" testclass="ResultCollector" testname="Aggregate Report"/>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### JMeter 테스트 실행 스크립트
```bash
#!/bin/bash
# scripts/run-performance-test.sh

set -e

# 설정 변수
JMETER_HOME=${JMETER_HOME:-"/opt/apache-jmeter"}
TEST_PLAN="performance-tests/ecommerce-load-test.jmx"
RESULTS_DIR="performance-results/$(date +%Y%m%d_%H%M%S)"
HOST=${TEST_HOST:-"localhost"}
PORT=${TEST_PORT:-"8080"}
THREADS=${TEST_THREADS:-"100"}
DURATION=${TEST_DURATION:-"300"}

# 결과 디렉토리 생성
mkdir -p "$RESULTS_DIR"

echo "Starting performance test..."
echo "Host: $HOST:$PORT"
echo "Threads: $THREADS"
echo "Duration: ${DURATION}s"
echo "Results will be saved to: $RESULTS_DIR"

# JMeter 실행
$JMETER_HOME/bin/jmeter \
  -n \
  -t "$TEST_PLAN" \
  -l "$RESULTS_DIR/results.jtl" \
  -e \
  -o "$RESULTS_DIR/dashboard" \
  -Jhost="$HOST" \
  -Jport="$PORT" \
  -Jthreads="$THREADS" \
  -Jduration="$DURATION" \
  -Djmeter.reportgenerator.overall_granularity=60000

# 결과 요약 생성
echo "Generating performance summary..."
cat > "$RESULTS_DIR/test-summary.md" << EOF
# Performance Test Results

**Test Configuration:**
- Host: $HOST:$PORT
- Concurrent Users: $THREADS
- Test Duration: ${DURATION}s
- Test Date: $(date)

**Key Metrics:**
EOF

# JTL 파일에서 주요 메트릭 추출
awk -F',' '
BEGIN {
    sum_elapsed = 0;
    count = 0;
    errors = 0;
    max_elapsed = 0;
    min_elapsed = 999999;
}
NR > 1 {
    elapsed = $2;
    success = $8;
    
    count++;
    sum_elapsed += elapsed;
    
    if (elapsed > max_elapsed) max_elapsed = elapsed;
    if (elapsed < min_elapsed) min_elapsed = elapsed;
    if (success == "false") errors++;
}
END {
    if (count > 0) {
        avg_response_time = sum_elapsed / count;
        error_rate = (errors / count) * 100;
        throughput = count / ('$DURATION');
        
        print "- Average Response Time: " avg_response_time " ms";
        print "- Min Response Time: " min_elapsed " ms";
        print "- Max Response Time: " max_elapsed " ms";
        print "- Error Rate: " error_rate " %";
        print "- Throughput: " throughput " req/s";
        print "- Total Requests: " count;
        print "- Failed Requests: " errors;
    }
}' "$RESULTS_DIR/results.jtl" >> "$RESULTS_DIR/test-summary.md"

echo "Performance test completed!"
echo "Results available at: $RESULTS_DIR"
echo "Dashboard: $RESULTS_DIR/dashboard/index.html"
```

## 2. Gatling 부하 테스트

### Gatling 시나리오 작성
```scala
// src/test/scala/simulations/ECommerceSimulation.scala
package simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._
import scala.util.Random

class ECommerceSimulation extends Simulation {

  // HTTP 프로토콜 설정
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling Performance Test")

  // 테스트 데이터 준비
  val userCredentials = csv("users.csv").readRecords
  val productIds = (1 to 1000).map(i => s"product-$i").toArray

  // 사용자 시나리오
  val userJourney = scenario("E-commerce User Journey")
    .feed(userCredentials)
    .exec(session => session.set("productId", productIds(Random.nextInt(productIds.length))))
    
    // 1. 로그인
    .exec(http("Login")
      .post("/auth/login")
      .body(StringBody("""{
        "username": "${username}",
        "password": "${password}"
      }""")).asJson
      .check(status.is(200))
      .check(jsonPath("$.token").saveAs("jwtToken"))
    )
    .pause(1, 3)
    
    // 2. 상품 목록 조회
    .exec(http("Browse Products")
      .get("/products")
      .queryParam("page", "0")
      .queryParam("size", "20")
      .header("Authorization", "Bearer ${jwtToken}")
      .check(status.is(200))
      .check(jsonPath("$.content").exists)
    )
    .pause(2, 5)
    
    // 3. 상품 상세 조회
    .exec(http("Get Product Details")
      .get("/products/${productId}")
      .header("Authorization", "Bearer ${jwtToken}")
      .check(status.is(200))
      .check(jsonPath("$.id").saveAs("selectedProductId"))
      .check(jsonPath("$.price").saveAs("productPrice"))
    )
    .pause(3, 8)
    
    // 4. 장바구니에 추가 (70% 확률)
    .randomSwitch(
      70.0 -> exec(http("Add to Cart")
        .post("/cart/items")
        .header("Authorization", "Bearer ${jwtToken}")
        .body(StringBody("""{
          "productId": "${selectedProductId}",
          "quantity": ${__Random(1,3)}
        }""")).asJson
        .check(status.is(201))
      ).pause(1, 3)
        
        // 5. 주문 생성 (50% 확률)
        .randomSwitch(
          50.0 -> exec(http("Create Order")
            .post("/orders")
            .header("Authorization", "Bearer ${jwtToken}")
            .body(StringBody("""{
              "items": [{
                "productId": "${selectedProductId}",
                "quantity": 1,
                "price": ${productPrice}
              }],
              "shippingAddress": {
                "street": "123 Test St",
                "city": "Test City",
                "zipCode": "12345"
              },
              "paymentMethod": "CREDIT_CARD"
            }""")).asJson
            .check(status.is(201))
            .check(jsonPath("$.orderId").saveAs("orderId"))
          )
          .pause(2, 5)
          
          // 6. 주문 확인
          .exec(http("Check Order Status")
            .get("/orders/${orderId}")
            .header("Authorization", "Bearer ${jwtToken}")
            .check(status.is(200))
            .check(jsonPath("$.status").in("PENDING", "PROCESSING", "COMPLETED"))
          )
        )
    )

  // 관리자 시나리오
  val adminJourney = scenario("Admin Operations")
    .exec(http("Admin Login")
      .post("/auth/login")
      .body(StringBody("""{
        "username": "admin",
        "password": "admin123"
      }""")).asJson
      .check(status.is(200))
      .check(jsonPath("$.token").saveAs("adminToken"))
    )
    .pause(1, 2)
    
    .repeat(5) {
      exec(http("Get Order Analytics")
        .get("/admin/analytics/orders")
        .queryParam("period", "daily")
        .header("Authorization", "Bearer ${adminToken}")
        .check(status.is(200))
      )
      .pause(10, 20)
      
      .exec(http("Get Inventory Status")
        .get("/admin/inventory/status")
        .header("Authorization", "Bearer ${adminToken}")
        .check(status.is(200))
      )
      .pause(15, 30)
    }

  // 부하 테스트 시나리오 설정
  setUp(
    // 일반 사용자 시뮬레이션
    userJourney.inject(
      // 점진적 부하 증가
      rampUsers(50).during(2.minutes),
      // 일정 부하 유지
      constantUsersPerSec(20).during(5.minutes),
      // 피크 부하
      rampUsers(100).during(1.minute),
      constantUsersPerSec(50).during(3.minutes),
      // 부하 감소
      rampUsers(0).during(1.minute)
    ),
    
    // 관리자 사용자 시뮬레이션
    adminJourney.inject(
      constantUsersPerSec(2).during(10.minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.max.lt(5000),
     global.responseTime.mean.lt(1000),
     global.successfulRequests.percent.gt(95),
     forAll.failedRequests.percent.lt(5)
   )
}
```

### Gatling 실행 및 보고서 생성
```bash
#!/bin/bash
# scripts/run-gatling-test.sh

set -e

GATLING_HOME=${GATLING_HOME:-"/opt/gatling"}
SIMULATION=${SIMULATION:-"simulations.ECommerceSimulation"}
TARGET_HOST=${TARGET_HOST:-"localhost:8080"}
RESULTS_DIR="gatling-results/$(date +%Y%m%d_%H%M%S)"

echo "Running Gatling performance test..."
echo "Simulation: $SIMULATION"
echo "Target: $TARGET_HOST"

# 환경변수로 타겟 호스트 설정
export JAVA_OPTS="-Dtarget.host=$TARGET_HOST"

# Gatling 실행
$GATLING_HOME/bin/gatling.sh \
  -s "$SIMULATION" \
  -rf "$RESULTS_DIR" \
  -sf "src/test/scala" \
  -bf "target/test-classes"

# 결과 분석 및 요약 생성
LATEST_RESULT=$(find "$RESULTS_DIR" -name "index.html" | head -1)
STATS_FILE=$(find "$RESULTS_DIR" -name "stats.json" | head -1)

if [ -f "$STATS_FILE" ]; then
    echo "Generating performance summary from $STATS_FILE..."
    
    # JSON 결과에서 주요 메트릭 추출
    jq -r '
    .stats.contents | to_entries[] | select(.key == "Global Information") | 
    "Performance Test Summary:
    - Total Requests: \(.value.stats.numberOfRequests.total)
    - Successful Requests: \(.value.stats.numberOfRequests.ok) (\(.value.stats.numberOfRequests.ko | tonumber | . / (.value.stats.numberOfRequests.total | tonumber) * 100 | floor)% error rate)
    - Mean Response Time: \(.value.stats.meanResponseTime.total) ms
    - 95th Percentile: \(.value.stats.percentiles1.total) ms
    - 99th Percentile: \(.value.stats.percentiles2.total) ms
    - Max Response Time: \(.value.stats.maxResponseTime.total) ms
    - Mean Requests/sec: \(.value.stats.meanNumberOfRequestsPerSecond.total)"
    ' "$STATS_FILE" > "$RESULTS_DIR/summary.txt"
    
    cat "$RESULTS_DIR/summary.txt"
fi

echo "Gatling test completed!"
echo "Results: $LATEST_RESULT"
```

## 3. Kafka 파티션 튜닝

### Kafka 성능 벤치마크
```bash
#!/bin/bash
# scripts/kafka-performance-test.sh

KAFKA_HOME=${KAFKA_HOME:-"/opt/kafka"}
BOOTSTRAP_SERVERS=${BOOTSTRAP_SERVERS:-"localhost:9092"}
TOPIC_NAME="performance-test-topic"
NUM_RECORDS=1000000
RECORD_SIZE=1024
THROUGHPUT=-1  # 무제한

echo "Starting Kafka performance test..."

# 테스트 토픽 생성
$KAFKA_HOME/bin/kafka-topics.sh \
  --create \
  --topic $TOPIC_NAME \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --partitions 12 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config unclean.leader.election.enable=false

# Producer 성능 테스트
echo "Running producer performance test..."
$KAFKA_HOME/bin/kafka-producer-perf-test.sh \
  --topic $TOPIC_NAME \
  --num-records $NUM_RECORDS \
  --record-size $RECORD_SIZE \
  --throughput $THROUGHPUT \
  --producer-props \
    bootstrap.servers=$BOOTSTRAP_SERVERS \
    acks=1 \
    batch.size=32768 \
    linger.ms=10 \
    compression.type=snappy \
    buffer.memory=67108864 \
  > producer-results.txt

# Consumer 성능 테스트
echo "Running consumer performance test..."
$KAFKA_HOME/bin/kafka-consumer-perf-test.sh \
  --topic $TOPIC_NAME \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --messages $NUM_RECORDS \
  --consumer-props \
    fetch.min.bytes=50000 \
    fetch.max.wait.ms=500 \
    max.poll.records=500 \
  > consumer-results.txt

# 결과 출력
echo "Producer Performance Results:"
cat producer-results.txt

echo "Consumer Performance Results:"
cat consumer-results.txt

# 토픽 삭제
$KAFKA_HOME/bin/kafka-topics.sh \
  --delete \
  --topic $TOPIC_NAME \
  --bootstrap-server $BOOTSTRAP_SERVERS
```

### Kafka 파라미터 튜닝 가이드
```properties
# server.properties - 프로듀서 성능 최적화
# 네트워크 스레드 수 (CPU 코어 수와 동일하게 설정)
num.network.threads=8

# I/O 스레드 수 (디스크 수의 2배)
num.io.threads=16

# 소켓 버퍼 크기
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# 로그 세그먼트 크기 (1GB)
log.segment.bytes=1073741824

# 로그 보관 시간 (7일)
log.retention.hours=168

# 압축 활성화
compression.type=snappy

# 배치 크기 최적화
batch.size=32768
linger.ms=10

# 백그라운드 스레드 수
background.threads=10

# 리플리케이션 최적화
replica.fetch.max.bytes=1048576
replica.fetch.wait.max.ms=500

# JVM 힙 크기 (8GB 서버 기준)
# export KAFKA_HEAP_OPTS="-Xmx6G -Xms6G"

# G1 GC 설정
# export KAFKA_JVM_PERFORMANCE_OPTS="-XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:G1HeapRegionSize=16m"
```

## 4. JVM 최적화

### Spring Boot JVM 튜닝
```bash
# JVM 옵션 설정
export JAVA_OPTS="
  # 힙 메모리 설정 (컨테이너 메모리의 70%)
  -Xms2g -Xmx2g
  
  # G1 가비지 컬렉터 사용
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1HeapRegionSize=16m
  -XX:G1NewSizePercent=20
  -XX:G1MaxNewSizePercent=40
  
  # GC 로깅
  -XX:+PrintGC
  -XX:+PrintGCDetails
  -XX:+PrintGCTimeStamps
  -XX:+UseGCLogFileRotation
  -XX:NumberOfGCLogFiles=5
  -XX:GCLogFileSize=10M
  -Xloggc:/app/logs/gc.log
  
  # 힙 덤프 설정
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/app/logs/
  
  # 성능 튜닝
  -XX:+UseStringDeduplication
  -XX:+OptimizeStringConcat
  -XX:+UseCompressedOops
  -XX:+UseCompressedClassPointers
  
  # 컨테이너 인식
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=70.0
  
  # JIT 컴파일러 최적화
  -XX:+TieredCompilation
  -XX:TieredStopAtLevel=1
  
  # 모니터링
  -Dcom.sun.management.jmxremote
  -Dcom.sun.management.jmxremote.port=9999
  -Dcom.sun.management.jmxremote.authenticate=false
  -Dcom.sun.management.jmxremote.ssl=false
"
```

### 애플리케이션 성능 모니터링
```java
@Component
@Slf4j
public class PerformanceMonitor {
    
    private final MeterRegistry meterRegistry;
    private final MemoryMXBean memoryBean;
    private final List<GarbageCollectorMXBean> gcBeans;
    
    public PerformanceMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.memoryBean = ManagementFactory.getMemoryMXBean();
        this.gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        // JVM 메트릭 등록
        registerJvmMetrics();
    }
    
    private void registerJvmMetrics() {
        // 힙 메모리 사용량
        Gauge.builder("jvm.memory.heap.used")
            .description("Used heap memory")
            .register(meterRegistry, this, monitor -> memoryBean.getHeapMemoryUsage().getUsed());
        
        Gauge.builder("jvm.memory.heap.max")
            .description("Max heap memory")
            .register(meterRegistry, this, monitor -> memoryBean.getHeapMemoryUsage().getMax());
        
        // 논힙 메모리 사용량
        Gauge.builder("jvm.memory.non_heap.used")
            .description("Used non-heap memory")
            .register(meterRegistry, this, monitor -> memoryBean.getNonHeapMemoryUsage().getUsed());
        
        // GC 메트릭
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            Gauge.builder("jvm.gc.collection.count")
                .tag("gc", gcBean.getName())
                .description("GC collection count")
                .register(meterRegistry, gcBean, GarbageCollectorMXBean::getCollectionCount);
            
            Gauge.builder("jvm.gc.collection.time")
                .tag("gc", gcBean.getName())
                .description("GC collection time")
                .register(meterRegistry, gcBean, GarbageCollectorMXBean::getCollectionTime);
        }
    }
    
    @EventListener
    @Async
    public void handleApplicationEvent(ApplicationEvent event) {
        // 애플리케이션 이벤트별 성능 메트릭 수집
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // 이벤트 처리 시간 측정
            Thread.sleep(10); // 실제 처리 시간 시뮬레이션
            
        } finally {
            sample.stop(Timer.builder("application.event.processing.time")
                .tag("event.type", event.getClass().getSimpleName())
                .description("Application event processing time")
                .register(meterRegistry));
        }
    }
    
    @Scheduled(fixedRate = 60000) // 1분마다 실행
    public void collectPerformanceMetrics() {
        // CPU 사용률
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        if (osBean instanceof com.sun.management.OperatingSystemMXBean) {
            com.sun.management.OperatingSystemMXBean sunOsBean = 
                (com.sun.management.OperatingSystemMXBean) osBean;
            
            meterRegistry.gauge("system.cpu.usage", sunOsBean.getProcessCpuLoad());
            meterRegistry.gauge("system.load.average.1m", osBean.getSystemLoadAverage());
        }
        
        // 스레드 정보
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        meterRegistry.gauge("jvm.threads.count", threadBean.getThreadCount());
        meterRegistry.gauge("jvm.threads.daemon", threadBean.getDaemonThreadCount());
        meterRegistry.gauge("jvm.threads.peak", threadBean.getPeakThreadCount());
        
        // 힙 메모리 사용률 계산
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        double heapUtilization = (double) heapUsage.getUsed() / heapUsage.getMax() * 100;
        meterRegistry.gauge("jvm.memory.heap.utilization", heapUtilization);
        
        // 알림 임계값 체크
        if (heapUtilization > 80) {
            log.warn("High heap memory utilization: {}%", heapUtilization);
            // 알림 발송 로직
        }
    }
}
```

## 5. 리소스 사용량 모니터링

### Prometheus 메트릭 수집
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  - job_name: 'kafka-cluster'
    static_configs:
      - targets: ['kafka-1:9101', 'kafka-2:9102', 'kafka-3:9103']
    scrape_interval: 30s

  - job_name: 'spring-boot-apps'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: '.*-service'
```

### 알럿 규칙 정의
```yaml
# alert_rules.yml
groups:
- name: performance-alerts
  rules:
  # High CPU usage
  - alert: HighCPUUsage
    expr: rate(process_cpu_seconds_total[5m]) * 100 > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage detected"
      description: "CPU usage is above 80% for {{ $labels.instance }}"

  # High memory usage
  - alert: HighMemoryUsage
    expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) * 100 > 85
    for: 3m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage detected"
      description: "Heap memory usage is above 85% for {{ $labels.instance }}"

  # High response time
  - alert: HighResponseTime
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High response time detected"
      description: "95th percentile response time is above 2s for {{ $labels.instance }}"

  # High error rate
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is above 5% for {{ $labels.instance }}"

  # Kafka consumer lag
  - alert: KafkaConsumerLag
    expr: kafka_consumer_lag_sum > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer lag is high"
      description: "Consumer lag is {{ $value }} for {{ $labels.topic }}"

  # Database connection pool
  - alert: DatabaseConnectionPoolHigh
    expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
    for: 3m
    labels:
      severity: warning
    annotations:
      summary: "Database connection pool usage is high"
      description: "Connection pool usage is above 80% for {{ $labels.instance }}"
```

### Grafana 대시보드 설정
```json
{
  "dashboard": {
    "id": null,
    "title": "E-commerce Performance Dashboard",
    "tags": ["performance", "microservices"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "id": 2,
        "title": "Throughput",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{ method }} {{ uri }}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "Error Rate"
          }
        ]
      },
      {
        "id": 4,
        "title": "JVM Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}",
            "legendFormat": "Heap Used"
          },
          {
            "expr": "jvm_memory_max_bytes{area=\"heap\"}",
            "legendFormat": "Heap Max"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

## 실습 과제

1. **완전한 성능 테스트 스위트**: JMeter와 Gatling을 사용한 포괄적인 성능 테스트 구현
2. **Kafka 성능 최적화**: 다양한 파티션 전략과 설정을 통한 Kafka 처리량 최적화
3. **JVM 튜닝**: 애플리케이션 특성에 맞는 JVM 파라미터 튜닝 및 성능 비교
4. **자동화된 성능 모니터링**: Prometheus + Grafana를 활용한 실시간 성능 모니터링 시스템 구축
5. **성능 회귀 테스트**: CI/CD 파이프라인에 통합된 자동 성능 회귀 테스트 구현

## 참고 자료

- [Apache JMeter Documentation](https://jmeter.apache.org/usermanual/index.html)
- [Gatling Documentation](https://gatling.io/docs/gatling/)
- [Kafka Performance Tuning](https://kafka.apache.org/documentation/#design_performance)
- [JVM Performance Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)