# 개인 Final project - 네일샵 예약시스템 구축

이 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성하였습니다.
project 내 각 domain(package)에 대한 간단한 Description은 아래와 같습니다.
  * gateway : gateway
  * reservation : 예약
  * view : 예약/완료 현황 (CQRS)
  * work : 네일작업
  
# Table of contents

- [네일샵 예약시스템 구축]
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [FeignClient 의 적용](#FeignClient-의-적용)
    - [Test 시나리오](#Test-시나리오)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [클러스터 적용 후 REST API 의 테스트](#클러스터-적용-후-REST-API-의-테스트)
    - [API 게이트웨이](#API-게이트웨이)
  - [운영](#운영)
    - [CI/CD 설정](#cicd-설정)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오

기능적 요구사항
1. 고객은 네일샵에 예약 / 예약 취소 / 변경을 한다.
1. 예약이 완료된 고객은 네일을 받는다. 
1. 예약이 변경/취소되면 네일작업 변경/취소된다.
1. 고객은 view 시스템을 통해 예약 상태를 조회할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 네일이 불가능 할 때는 예약이 불가능해야 한다. Sync 호출
1. 장애격리
    1. 예약/네일작업 시스템(core)만 온전하면 시스템은 정상적으로 수행되어야 한다.  Async (event-driven), Eventual Consistency
    1. 네일작업이 과중되면 예약을 잠시후에 하도록 유도한다.  Circuit breaker, fallback
1. 성능
    1. 고객이 예약/네일 결과를 시스템에서 확인할 수 있어야 한다.(view 시스템으로 구현, CQRS)


# 분석/설계

* 이벤트스토밍 결과

![이벤트스토밍](https://user-images.githubusercontent.com/40315778/80058687-e85f3980-8564-11ea-8af0-3226ce170729.jpg)

- Core Domain : 예약 (Reservation) 및 네일 (work) 도메인
- Supporting Domain : view(CQRS) 도메인

## 헥사고날 아키텍처 다이어그램 도출
    
![헥사고날](https://user-images.githubusercontent.com/40315778/80059113-14c78580-8566-11ea-821a-4a235cc94197.jpg)


# 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

네일샵 예약시스템은 아래의 4가지 마이크로 서비스로 구성되어 있다.

1. 게이트 웨이: [https://github.com/dani-vov/nail_gateway.git](https://github.com/dani-vov/nail_gateway.git)
1. 예약 시스템: [https://github.com/dani-vov/nail_reservation.git](https://github.com/dani-vov/nail_reservation.git)
1. 네일 시스템: [https://github.com/dani-vov/nail_work.git](https://github.com/dani-vov/nail_work.git)
1. 조회 시스템: [https://github.com/dani-vov/nail_view.git](https://github.com/dani-vov/nail_view.git)

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 예약 시스템의 Reservation.class)

``` java
package nailshop;

import nailshop.config.kafka.KafkaProcessor;
import nailshop.external.Nail;
import nailshop.external.NailService;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.util.MimeTypeUtils;

import javax.persistence.*;

@Entity
@Table(name = "RESERVATION")
public class Reservation {

    @Id
    @GeneratedValue
    private Long reservationId;

    private String reservatorName;

    private String reservationDate;

    private String phoneNumber;

    @PostPersist
    public void publishReservationReservedEvent() throws InterruptedException {
        // Reserved 이벤트 발생
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(new ReservationReserved(this));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor;
        processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());

        Nail nail = new Nail();
        nail.setReservationId(this.getId());
        nail.setEmployee("파이리");
        nail.setDescription("젤네일");
        nail.setFee(50000L);
        nail.setReservatorName(this.getReservatorName());
        nail.setReservationDate(this.getReservationDate());
        nail.setPhoneNumber(this.getPhoneNumber());

        Application.applicationContext.getBean(NailService.class).work(nail);
    }

    @PostUpdate
    public void publishReservationChangedEvent() {
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(new ReservationChanged(this));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor;
        processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }

    @PostRemove
    public void publishReservationCanceledEvent() {
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(new ReservationCanceled(this));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor;
        processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }

    public Long getId() {
        return reservationId;
    }

    public void setId(Long id) {
        this.reservationId = id;
    }

    public String getReservatorName() {
        return reservatorName;
    }

    public void setReservatorName(String reservatorName) {
        this.reservatorName = reservatorName;
    }

    public String getReservationDate() {
        return reservationDate;
    }

    public void setReservationDate(String reservationDate) {
        this.reservationDate = reservationDate;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }
}

```

- Entity Pattern 과 Repository Pattern을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다.
RDB로는 H2를 사용하였다. 

``` java
package nailshop;

import org.springframework.data.repository.CrudRepository;

public interface ReservationRepository extends CrudRepository<Reservation, Long> {
}

```
## FeignClient 의 적용
- FeignClient를 적용하였고, url을 하드코딩으로 작성하였던 부분의 소스를 변수타입으로 수정하여 문제를 해결하였다.
  (NailService 파일과 application.yml 파일 둘 다 수정하였다)

- NailService
``` java
package nailshop.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@Service
@FeignClient(name = "work", url = "${api.url.work}")
public interface NailService {

    @RequestMapping(method = RequestMethod.POST, path = "/nails")
    public void work(@RequestBody Nail nail);
}
```
- application.yml
```
server:
  port: 8080
---

spring:
  profiles: default
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
        streams:
          binder:
            configuration:
              default:
                key:
                  serde: org.apache.kafka.common.serialization.Serdes$StringSerde
                value:
                  serde: org.apache.kafka.common.serialization.Serdes$StringSerde
      bindings:
#        event-in:
#          group: reservation
#          destination: NailShop
#          contentType: application/json
        event-out:
          destination: NailShop
          contentType: application/json

  logging:
    level:
      org.hibernate.type: trace
      org.springframework.cloud: debug

server:
  port: 8081

api:
  url:
    work: http://localhost:8082
---

spring:
  profiles: docker
  cloud:
    stream:
      kafka:
        binder:
          brokers: my-kafka.kafka.svc.cluster.local:9092
        streams:
          binder:
            configuration:
              default:
                key:
                  serde: org.apache.kafka.common.serialization.Serdes$StringSerde
                value:
                  serde: org.apache.kafka.common.serialization.Serdes$StringSerde
      bindings:
#        event-in:
#          group: reservation
#          destination: NailShop
#          contentType: application/json
        event-out:
          destination: NailShop
          contentType: application/json

api:
  url:
    work: http://work:8080
```


## Test 시나리오

- Local 환경 테스트시 
  아래의 명령어는 httpie 프로그램을 사용하여 입력한다.
  ```
  # 예약 서비스의 예약
  http post localhost:8081/reservations reservatorName="dani" reservationDate="2020-04-22" phoneNumber="010-1234-5678"

  # 예약된 서비스의 예약 취소
  http delete localhost:8081/reservations/1

  # 예약된 서비스의 예약 변경
  http patch localhost:8081/reservations/1 reservationDate="2020-05-01"

  # 네일작업 리스트 확인
  http localhost:8083/allStats
  ```
  수행결과 다음과 같다 (예시 : Post)
  
  ![로컬테스트](https://user-images.githubusercontent.com/40315778/80060422-789f7d80-8569-11ea-9467-4ed1eaf793d3.jpg)

- Cloud 환경 테스트시
  1) 'kubectl get all' 로 서비스 상태 및 gateway의 External IP/Port를 확인한다. (gateway IP : 52.231.117.106 / Port : 8080)
  
  ![서버확인](https://user-images.githubusercontent.com/40315778/80060481-a684c200-8569-11ea-99d3-4abed4e80dfd.jpg)
  
  2) 아래의 명령어는 httpie 프로그램을 사용하여 입력한다.
   ```
   # 예약 서비스의 예약
   http post 52.231.117.106:8080/reservations reservatorName="dani" reservationDate="2020-04-22" phoneNumber="010-1234-5678"

   # 예약된 서비스의 예약 취소
   http delete 52.231.117.106:8080/reservations/1

   # 예약된 서비스의 예약 변경
   http patch 52.231.117.106:8080/reservations/1 reservationDate="2020-05-01"

   # 네일작업 리스트 확인
   http 52.231.117.106:8080/allStats
   ```
   
   수행결과 다음과 같다 (예시 : patch / delete)
   
   ![클라우드테스트1](https://user-images.githubusercontent.com/40315778/80060750-46dae680-856a-11ea-8325-95e81ef6114b.jpg)
   
   ![클라우드테스트2](https://user-images.githubusercontent.com/40315778/80060842-886b9180-856a-11ea-9b6e-bcbc3fffdbdf.jpg)

## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 예약(reservation)->네일(work) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 진료서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

``` java
package nailshop.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@Service
@FeignClient(name = "work", url = "${api.url.work}")
public interface NailService {

    @RequestMapping(method = RequestMethod.POST, path = "/nails")
    public void work(@RequestBody Nail nail);
}
```

- 예약완료 직후(@PostPersist) 네일 작업을 요청하도록 처리
```
    @PostPersist
    public void publishReservationReservedEvent() throws InterruptedException {
        // Reserved 이벤트 발생
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(new ReservationReserved(this));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor;
        processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());

       // 예약이 발생하면 바로 네일 작업 진행.
        Nail nail = new Nail();
        nail.setReservationId(this.getId());
        nail.setEmployee("파이리");
        nail.setDescription("젤네일");
        nail.setFee(50000L);
        nail.setReservatorName(this.getReservatorName());
        nail.setReservationDate(this.getReservationDate());
        nail.setPhoneNumber(this.getPhoneNumber());

        Application.applicationContext.getBean(NailService.class).work(nail);
    }
```

- 동기식 호출 확인
```
# 네일(work) 서비스를 잠시 내려놓음 (ctrl+c)

# 예약 처리
http post localhost:8081/reservations reservatorName="dani" reservationDate="2020-04-22" phoneNumber="010-1234-5678" #Fail

#진료 서비스 재기동
cd work
mvn spring-boot:run

#예약처리
http post localhost:8081/reservations reservatorName="dani" reservationDate="2020-04-22" phoneNumber="010-1234-5678" #Success
```

## 클러스터 적용 후 REST API 의 테스트
- http://52.231.117.106:8080/reservations //reservation 조회 
- http://52.231.117.106:8080/reservations reservatorName="dani" reservationDate="2020-04-22" phoneNumber="010-1234-5678"   //reservation 요청 
- Delete http://52.231.117.106:8080/reservations/1 	//reservation Cancel  Sample
- http://52.231.117.106:8080/allStats //view  조회
- http://52.231.117.106:8080/nails/ 	//work 조회

## API 게이트웨이
- Local 테스트 환경에서는 localhost:8080에서 Gateway API 가 작동.
- Cloud 환경에서는 http://52.231.117.106:8080 에서 Gateway API가 작동.
- application.yml 파일에 프로파일 별로 Gateway 설정.

### Gateway 설정 파일
```yaml

server:
  port: 8088

---
spring:
  profiles: default
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:8088/.well-known/jwks.json
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://localhost:8081
          predicates:
            - Path=/reservations/**
        - id: work
          uri: http://localhost:8082
          predicates:
            - Path=/nails/**
        - id: view
          uri: http://localhost:8083
          predicates:
            - Path=/allStats/**
        - id: oauth
          uri: http://localhost:8090
          predicates:
            - Path=/oauth/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---
spring:
  profiles: docker
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:8080/.well-known/jwks.json
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://reservation:8080
          predicates:
            - Path=/reservations/**
        - id: work
          uri: http://work:8080
          predicates:
            - Path=/nails/**
        - id: view
          uri: http://view:8080
          predicates:
            - Path=/allStats/**
        - id: oauth
          uri: http://oauth:8080
          predicates:
            - Path=/oauth/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


server:
  port: 8080
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 Git을 통해 빌드되며, Git Master에 트리거 되어 있다. pipeline build script 는 각 프로젝트 폴더 이하에 azure_pipeline.yml 에 포함되었다.

azure_pipelist.yml 참고

kubernetes Service
```yaml

trigger:
- master

resources:
- repo: self

variables:
- group: common-value
  # containerRegistry: 'event.azurecr.io'
  # containerRegistryDockerConnection: 'acr'
  # environment: 'aks.default'
- name: imageRepository
  value: 'gateway'
- name: dockerfilePath
  value: '**/Dockerfile'
- name: tag
  value: '$(Build.BuildId)'
  # Agent VM image name
- name: vmImageName
  value: 'ubuntu-latest'
- name: MAVEN_CACHE_FOLDER
  value: $(Pipeline.Workspace)/.m2/repository
- name: MAVEN_OPTS
  value: '-Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'


stages:
- stage: Build
  displayName: Build stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: CacheBeta@1
      inputs:
        key: 'maven | "$(Agent.OS)" | **/pom.xml'
        restoreKeys: |
           maven | "$(Agent.OS)"
           maven
        path: $(MAVEN_CACHE_FOLDER)
      displayName: Cache Maven local repo
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        options: '-Dmaven.test.skip=true -Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'
        javaHomeOption: 'JDKVersion'
        jdkVersionOption: '1.8'
        jdkArchitectureOption: 'x64'
        goals: 'clean package'
    - task: Docker@2
      inputs:
        containerRegistry: $(containerRegistryDockerConnection)
        repository: $(imageRepository)
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(tag)

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build

  jobs:
  - deployment: Deploy
    displayName: Deploy
    pool:
      vmImage: $(vmImageName)
    environment: $(environment)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configurationType: 'inline'
              inline: |
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: $(imageRepository)
                  labels:
                    app: $(imageRepository)
                spec:
                  replicas: 1
                  selector:
                    matchLabels:
                      app: $(imageRepository)
                  template:
                    metadata:
                      labels:
                        app: $(imageRepository)
                    spec:
                      containers:
                        - name: $(imageRepository)
                          image: $(containerRegistry)/$(imageRepository):$(tag)
                          ports:
                            - containerPort: 8080
              secretType: 'dockerRegistry'
              containerRegistryType: 'Azure Container Registry'
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configurationType: 'inline'
              inline: |
                apiVersion: v1
                kind: Service
                metadata:
                  name: $(imageRepository)
                  labels:
                    app: $(imageRepository)
                spec:
                  ports:
                    - port: 8080
                      targetPort: 8080
                  selector:
                    app: $(imageRepository)
                  type:
                      LoadBalancer
              secretType: 'dockerRegistry'
              containerRegistryType: 'Azure Container Registry'
```

## 무정지 재배포
- 모든 프로젝트의 readiness probe 및 liveness probe 설정 완료.
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
livenessProbe:
  httpGet:
     path: /actuator/health
     port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5
```

