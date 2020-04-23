# 개인 Final project - 네일샵 예약시스템 구축

이 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성하였습니다.
project 내 각 Package에 대한 간단한 Description은 아래와 같습니다.
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
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
    - [API 게이트웨이](#API-게이트웨이)
    - [Oauth](#oauth)
  - [운영](#운영)
    - [CI/CD 설정](#cicd-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출--서킷-브레이킹--장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
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

* 이벤트스토밍 결과:  http://msaez.io/#/storming/0vtSW2vBLoZTFiAsgdwS6H7ODs33/every/2dac041f4e652d598a042694dfa26b20/-M5LTyP4cBS9IpsqYq0h



- Core Domain : 예약 (Reservation) 및 진료 (Diagnosis) 도메인
- Supporting Domain : Lookup(CQRS) 도메인
- General Domain : 알림(notice) 시스템.

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/38850007/79833622-aad4a200-83e6-11ea-80f1-6eb9a59503af.png)


# 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

동물병원 예약/진료 시스템은 아래의 7가지 마이크로 서비스로 구성되어 있다.

1. 게이트 웨이: [https://github.com/AnimalHospital2/gateway.git](https://github.com/AnimalHospital2/gateway.git)
1. Oauth 시스템: [https://github.com/AnimalHospital2/ouath.git](https://github.com/AnimalHospital2/ouath.git)
1. 예약 시스템: [https://github.com/AnimalHospital2/reservation.git](https://github.com/AnimalHospital2/reservation.git)
1. 진료 시스템: [https://github.com/AnimalHospital2/diagnosis.git](https://github.com/AnimalHospital2/diagnosis.git)
1. 수납 시스템: [https://github.com/AnimalHospital2/acceptance.git](https://github.com/AnimalHospital2/acceptance.git)
1. 알림 시스템: [https://github.com/AnimalHospital2/notice.git](https://github.com/AnimalHospital2/notice.git)

- 게이트웨이 시스템은 수업시간에 이용한 예제를 프로젝트에 맞게 설정을 변경하였다. 
- Oauth 시스템은 수업시간에 이용한 예제를 그대로 활용하였다.

모든 시스템은 Spring Boot로 구현하였고 `mvn spring-boot:run` 명령어로 실행할 수 있다.

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 예약 시스템의 Reservation.class). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

``` java
package com.example.reservation;

import com.example.reservation.external.MedicalRecord;
import com.example.reservation.external.MedicalRecordService;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.cloud.stream.messaging.Processor;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

import javax.persistence.*;

@Entity
@Table(name = "RESERVATION")
public class Reservation {

    @Id
    @GeneratedValue
    private Long id;

    private String reservatorName;

    private String reservationDate;

    private String phoneNumber;

    @PostPersist
    public void publishReservationReservedEvent() {

        MedicalRecord medicalRecord = new MedicalRecord();

        medicalRecord.setReservationId(this.getId());
        medicalRecord.setDoctor("Brad pitt");
        medicalRecord.setMedicalOpinion("별 이상 없습니다.");
        medicalRecord.setTreatment("그냥 집에서 푹 쉬면 나을 것입니다.");

        ReservationApplication.applicationContext.getBean(MedicalRecordService.class).diagnosis(medicalRecord);


        // Reserved 이벤트 발생

        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(new ReservationReserved(this));
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        Processor processor = ReservationApplication.applicationContext.getBean(Processor.class);
        MessageChannel outputChannel = processor.output();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());


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

        Processor processor = ReservationApplication.applicationContext.getBean(Processor.class);
        MessageChannel outputChannel = processor.output();

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

        Processor processor = ReservationApplication.applicationContext.getBean(Processor.class);
        MessageChannel outputChannel = processor.output();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
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

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다.
RDB로는 H2를 사용하였다. 
``` java
package com.example.reservation;

import org.springframework.data.repository.CrudRepository;

public interface ReservationRepository extends CrudRepository<Reservation, Long> {
}

}
```
- 적용 후 REST API 의 테스트

주의!!! reservation 서비스에는 FeignClient가 적용되어 있다. 여기에 diagnosis 시스템의 api 주소가 하드코딩되어 있어 로컬 테스트 환경과
Cloud 테스트 환경에서는 그 값을 달리하여 테스트하여야 한다.

package com.example.reservation.external.MedicalRecordService의 내용을 테스트 환경에 따라 변경해준다.;


- Local 환경 테스트시 
``` java
@FeignClient(name = "diagnosis", url = "http://localhost:8083")
public interface MedicalRecordService {

    @RequestMapping(method = RequestMethod.POST, path = "/medicalRecords")
    public void diagnosis(@RequestBody MedicalRecord medicalRecord);
}
```

- Cloud 환경 테스트시
``` java
@FeignClient(name = "diagnosis", url = "http://diagnosis:8080")
public interface MedicalRecordService {

    @RequestMapping(method = RequestMethod.POST, path = "/medicalRecords")
    public void diagnosis(@RequestBody MedicalRecord medicalRecord);
}
```

아래의 명령어는 httpie 프로그램을 사용하여 입력한다.
```
# 예약 서비스의 예약
http post localhost:8081/reservations reservatorName="Jackson" reservationDate="2020-04-30" phoneNumber="010-1234-5678"

# 예약 서비스의 예약 취소
http delete localhost:8081/reservations/1

# 예약 서비스의 예약 변경
http patch localhost:8081/reservations/1 reservationDate="2020-05-01"

# 진료 기록 리스트 확인
http localhost:8083/medicalRecords

```

## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 예약(reservation)->진료(diagnosis) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 진료서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (app) 결제이력Service.java
package com.example.reservation.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name = "diagnosis", url = "http://diagnosis:8080")
  public interface MedicalRecordService {

    @RequestMapping(method = RequestMethod.POST, path = "/medicalRecords")
    public void diagnosis(@RequestBody MedicalRecord medicalRecord);
}

```

- 예약완료 직후(@PostPersist) 진단을 요청하도록 처리
```
# Reservation.java (Entity)
    @PostPersist
       public void publishReservationReservedEvent() {
   
           // 예약이 발생하면 바로 진료 진행.
           MedicalRecord medicalRecord = new MedicalRecord();
   
           medicalRecord.setReservationId(this.getId());
           medicalRecord.setDoctor("Brad pitt");
           medicalRecord.setMedicalOpinion("별 이상 없습니다.");
           medicalRecord.setTreatment("그냥 집에서 푹 쉬면 나을 것입니다.");
   
           ReservationApplication.applicationContext.getBean(MedicalRecordService.class).diagnosis(medicalRecord);
   
   
           // Reserved 이벤트 발생
           ObjectMapper objectMapper = new ObjectMapper();
           String json = null;
   
           try {
               json = objectMapper.writeValueAsString(new ReservationReserved(this));
           } catch (JsonProcessingException e) {
               throw new RuntimeException("JSON format exception", e);
           }
   
           Processor processor = ReservationApplication.applicationContext.getBean(Processor.class);
           MessageChannel outputChannel = processor.output();
   
           outputChannel.send(MessageBuilder
                   .withPayload(json)
                   .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                   .build());
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 진단 시스템이 장애가 나면 예약도 못받는다는 것을 확인.(비즈상 무리가 있음..) 

```
# 진료 (diagnosis) 서비스를 잠시 내려놓음 (ctrl+c)

# 예약 처리
http post localhost:8081/reservations reservatorName="Jackson" reservationDate="2020-04-30" phoneNumber="010-1234-5678" #Fail

#진료 서비스 재기동
cd diagnosis
mvn spring-boot:run

#예약처리
http post localhost:8081/reservations reservatorName="Jackson" reservationDate="2020-04-30" phoneNumber="010-1234-5678" #Success
```

## 클러스터 적용 후 REST API 의 테스트
- http://52.231.118.148:8080/medicalRecords/     		//diagnosis 조회
- http://52.231.118.148:8080/reservations/       		//reservation 조회 
- http://52.231.118.148:8080/reservations reservatorName="pdc" reservationDate="202002" phoneNumber="0103701" //reservation 요청 
- Delete http://52.231.118.148:8080/reservations/1 	//reservation Cancel  Sample
- http://52.231.118.148:8080/reservationStats/   	  //lookup  조회
- http://52.231.118.148:8080/financialManagements/ 	//acceptance 조회


- 또한 과도한 예약 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)


## 비동기식 호출과 Eventual Consistency


진료가 이루어진 후에 수납시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 수납 시스템의 처리를 위하여 예약/진료 시스템이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 진료이력을 남긴 후에 곧바로 진료가 이루어졌다는 이벤트를를 카프카로 송출한다(Publish)
 
```
// package Animal.Hospital.MedicalRecord;

    @PrePersist
    public void onPrePersist(){
        Treated treated = new Treated();
        BeanUtils.copyProperties(this, treated);
        treated.publish();
    }

```

- 수납 서비스에서는 진료완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

``` java
@Service
public class KafkaListener {

    @Autowired
    FinancialManagementRepository financialManagementRepository;

    @StreamListener(Processor.INPUT)

    public void TreatedEvent(@Payload Treated treated) {
        if(treated.getEventType().equals("Treated")) {
            System.out.println("수납요청 되었습니다.");

            FinancialManagement financialManagement = new FinancialManagement();
            financialManagement.setReservationId(treated.getReservationId());
            financialManagement.setFee(10000L);
            financialManagementRepository.save(financialManagement);
        }
    }
}
```

알림 시스템은 실제로 문자를 보낼 수는 없으므로, 예약/변경/취소 이벤트에 대해서 System.out.println 처리 하였다.
  
``` java
package com.example.notice;

@Service
public class KafkaListener {
    @StreamListener(Processor.INPUT)
    public void onReservationReservedEvent(@Payload ReservationReserved reservationReserved) {
        if(reservationReserved.getEventType().equals("ReservationReserved")) {
            System.out.println("예약 되었습니다.");
        }
    }

    @StreamListener(Processor.INPUT)
    public void onReservationChangedEvent(@Payload ReservationChanged reservationChanged) {
        if(reservationChanged.getEventType().equals("ReservationChanged")) {
            System.out.println("예약 변경 되었습니다.");
        }
    }

    @StreamListener(Processor.INPUT)
    public void onReservationCanceledEvent(@Payload ReservationCanceled reservationCanceled) {
        if(reservationCanceled.getEventType().equals("ReservationCanceled")) {
            System.out.println("예약 취소 되었습니다.");
        }
    }
}

```

수납, Lookup(CQRS) 시스템은 예약/진료와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 수납/Lookup 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약/진료를 하는데 문제가 없다:
```
# 수납 서비스 (acceptance) 를 잠시 내려놓음 (ctrl+c)

#예약처리
http post localhost:8081/reservations reservatorName="Jackson" reservationDate="2020-04-30" phoneNumber="010-1234-5678" #Success

#예약상태 확인
http localhost:8081/reservations     # 예약 추가 된 것 확인

#수납 서비스 기동
cd acceptance
mvn spring-boot:run

#수납상태 확인
http localhost:8085/financialManagements     # 모든 예약-진료에 대해서 요금이 청구되엇음을 확인.

```
## API 게이트웨이
- Local 테스트 환경에서는 localhost:8080에서 Gateway API 가 작동.
- Cloud 환경에서는 http://52.231.118.148:8080 에서 Gateway API가 작동.
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
        - id: diagnosis
          uri: http://localhost:8083
          predicates:
            - Path=/medicalRecords/**
        - id: lookup
          uri: http://localhost:8084
          predicates:
            - Path=/reservationStats/**
        - id: acceptance
          uri: http://localhost:8085
          predicates:
            - Path=/financialManagements/**
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
        - id: diagnosis
          uri: http://diagnosis:8080
          predicates:
            - Path=/medicalRecords/**
        - id: lookup
          uri: http://lookup:8080
          predicates:
            - Path=/reservationStats/**
        - id: acceptance
          uri: http://acceptance:8080
          predicates:
            - Path=/financialManagements/**
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

## Oauth 인증 적용.
- Oauth 인증 적용. 
- But, 수업 중에 사용한 Oauth 프로젝트를 그대로 이용하여 Gateway에 붙이기만 함.

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
  value: 'order'
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
        options: '-Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'
        javaHomeOption: 'JDKVersion'
        jdkVersionOption: '1.8'
        jdkArchitectureOption: 'x64'
        goals: 'package'
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
              secretType: 'dockerRegistry'
              containerRegistryType: 'Azure Container Registry'
```


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약 시스템(reservation)-->진료(diagnosis) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 진료 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

server:
  port: 8081
spring:
  profiles: default
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        output:
          destination: animal
          contentType: application/json
feign:
  hystrix:
    enabled: true
    
    

```

- 피호출 서비스(진료:diagnosis) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (diagnosis) MedicalRecord.java (Entity)

    @PrePersist
    public void onPrePersist(){  //진료이력을 저장한 후 적당한 시간 끌기
        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60s -r10 --content-type "application/json" 'http://localhost:8081/reservations POST {"reservatorName": "Jackson", "phoneNumber": "01032713104", "reservationDate": "2020-05-01"}'

Windows 안에서 작동하는 Ubuntu에서 siege 실행시 "[error] unable to set close control sock.c:141: Invalid argument" 이 발생하여 중간 과정은 알 수 없음.

그러나 아래와 같은 결과를 확인.

Lifting the server siege...
Transactions:                   1067 hits
Availability:                  78.92 %
Elapsed time:                  59.46 secs
Data transferred:               0.37 MB
Response time:                  5.36 secs
Transaction rate:              17.94 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   96.13
Successful transactions:        1067
Failed transactions:             285
Longest transaction:            7.01
Shortest transaction:           0.02

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 78.92% 가 성공.

## 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 진료서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy diagnosis --min=1 --max=10 --cpu-percent=15
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

