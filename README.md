# 최종 - 한준성 - 문자사서함
intensive coursework level 2 cloud native development 과정 최종 과제

##사용계정 :
admin23@gkn2021hotmail.onmicrosoft.com
skcc123!

##리소스이름 :
admin23,
admin23aks,
admin23acr

##깃주소 :
https://github.com/bluecap2/mailbox-mailbox
https://github.com/bluecap2/mailbox-alarm
https://github.com/bluecap2/mailbox-mobile
https://github.com/bluecap2/mailbox-gateway

##메세지보내기 호출 :
http POST 52.141.27.158:8080/mobiles user=01012345678 receiver=01099998888 text="hello"

##VIEW
http 52.141.27.158:8080/readmails

##kafka topic mailbox :
kubectl -n kafka exec -ti my-kafka-0 -- kafka-console-consumer --bootstrap-server my-kafka:9092 --topic mailbox --from-beginning

# Table of contents

- [최종 - 한준성 - 문자사서함](#최종 - 한준성 - 문자사서함)
  - [서비스 시나리오](#서비스 시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현:)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)


# 서비스 시나리오

기능적 요구사항
1. 발신자가 메세지를 전송할 수 있다.
1. 메일함에 메세지가 저장된다.
1. 수신자가 메세지를 읽을 수 있다.
1. 발신자의 메세지가 전송되면 알람을 보낸다.
1. 수신자에게 메세지가 도착하면 알람을 보낸다.

비기능적 요구사항
1. 장애격리
    1. 알람 시스템 장애가 나도 메세지 발송은 문제없어야 한다. 
1. 성능
    1. 수신자는 자주 메일함에서 내용을 확인할 수 있는 메일조회시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 송신과 수신이 정상적으로 완료되면 카톡 등으로 알림을 줄 수 있어야 한다  Event driven
1. 트랜잭션
    1. 전송했을 때 결제를 반드시 거쳐야 발송이 가능하다. Request/Response 



# 분석/설계


* 이벤트스토밍 결과:  http://msaez.io/#/storming/HluP7yEOV7Pf1n7CpyHZMBPUJL32/mine/6230d03790bf283e6336bdcaa169344d/-M5WDffnL3y8ZQvvcRf6

## 이벤트 도출, 액터, 커맨드 부착하여 읽기 좋게-> 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/48303857/79930894-9e068b80-8484-11ea-96f4-289bc5a1f769.png)

    - 이벤트는 꼭 필요한 수준으로 정의 하였음
    - 메세지를 발송하는 mobile, 내용이 저장되는 mailbox, 알람을 보내주는 alarm시스템 각각 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌
    
## 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/48303857/79930894-9e068b80-8484-11ea-96f4-289bc5a1f769.png)

## 폴리시 부착

![image](https://user-images.githubusercontent.com/48303857/79931131-31d85780-8485-11ea-8c7f-6357e0eb28c3.png)


## 도메인 분리

![image](https://user-images.githubusercontent.com/48303857/79931199-68ae6d80-8485-11ea-978d-3ce15a0b308e.png)

    - 도메인 서열 분리 
        - Core Domain:  mobile(front), mailbox : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:   alarm : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
 
## 폴리시의 이동과 컨텍스트 매핑

![image](https://user-images.githubusercontent.com/48303857/79931740-a65fc600-8486-11ea-9f75-66655a5ebc50.png)

## 모델 완성

![image](https://user-images.githubusercontent.com/48303857/79931947-3998fb80-8487-11ea-9129-09b40f63deb4.png)
    - View Model 추가
    
## 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

기능적 요구사항
1. 발신자가 메세지를 전송할 수 있다. (ok)
1. 메일함에 메세지가 저장된다.(ok)
1. 수신자가 메세지를 읽을 수 있다.(ok)
1. 발신자의 메세지가 전송되면 알람을 보낸다.(ok)
1. 수신자에게 메세지가 도착하면 알람을 보낸다.(ok)

비기능적 요구사항
1. 장애격리
    1. 알람 시스템 장애가 나도 메세지 발송은 문제없어야 한다. 손편지는 중간에 없어질 위험이 있기 때문에 이를 충분히 커버할 수 있도록 보낸 메세지는 절대 유실되지 않아야 한다.(ok)
1. 성능
    1. 수신자는 자주 메일함에서 내용을 확인할 수 있는 메일조회시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS(ok)
    1. 송신과 수신이 정상적으로 완료되면 카톡 등으로 알림을 줄 수 있어야 한다  Event driven (ok)

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/48303857/79940785-aa4b1280-849d-11ea-8aae-b1fb8b5511e3.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    - 알람 DB는 DocumentDB 구현을 통해 Polyglot 구조를 가지도록 설계

## Request-Response 모델 구현

![image](https://user-images.githubusercontent.com/48303857/79941104-78867b80-849e-11ea-832d-7b1866691d96.png)
    
    - 전송 요청을 하면 결제를 반드시 거쳐서 메세지가 발송되도록 Req/Res 호출구조를 만들었다.
     


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd mobile
mvn spring-boot:run

cd mailbox
mvn spring-boot:run 

cd alarm
mvn spring-boot:run   
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Mobile 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package mailbox;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Mobile_table")
public class Mobile {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Integer user;
    private Integer receiver;
    private String text;

    @PrePersist
    public void onPrePersist(){
        Sent sent = new Sent();
        BeanUtils.copyProperties(this, sent);
        sent.publish();
    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Integer getUser() {
        return user;
    }

    public void setUser(Integer user) {
        this.user = user;
    }
    public Integer getReceiver() {
        return receiver;
    }

    public void setReceiver(Integer receiver) {
        this.receiver = receiver;
    }
    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package mailbox;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface MobileRepository extends PagingAndSortingRepository<Mobile, Long>{

}
```

- 적용 후 REST API 의 테스트X
```
# app 서비스의 주문처리
http localhost:8081/orders item="통닭"

# store 서비스의 배달처리
http localhost:8083/주문처리s orderId=1

# 주문 상태 확인
http localhost:8081/orders/1
```

## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


메세지가 전송 된 이후 alarm 서비스로 이를 알려주는 부분은 비동기식으로 처리하여 alarm여부가 실제 메시지 전송에 영향을 주지 않도록 처리한다.
 
- 이를 위하여 alarm List 에 기록을 남긴 후에 곧바로 전송 완료 이벤트를 카프카로 송출한다(Publish)
- 동일하게 메세지를 수신한 경우도 메세지 수신 이벤트를 카프카로 Publish 한다.
 
```
  @PostPersist
    public void onPostPersist(){
        Saved saved = new Saved();
        BeanUtils.copyProperties(this, saved);
        saved.publish();
```
- alarm 서비스에서는 전송/수신 완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
@StreamListener(KafkaProcessor.INPUT)
    public void wheneverSent_전송알람(@Payload Sent sent){

        if(sent.isMe()){
            System.out.println("##### listener 전송알람 : " + sent.toJson());
            //외부 MMS전송 모듈 호출
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSaved_수신알람(@Payload Saved saved){

        if(saved.isMe()){
            System.out.println("##### listener 수신알람 : " + saved.toJson());
            //외부 MMS전송 모듈 호출
        }
    }

```
sent 되고나서 Event를 Kafka로 전송하고 이를 수신한 mailbox 시스템에서 saved 처리를 함 - 비동기식 Event Driven 
![image](https://user-images.githubusercontent.com/48303857/79946264-922dc000-84aa-11ea-90b8-587f1e090daa.png)



alarm 시스템은 다른 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, alarm시스템이 유지보수로 인해 잠시 내려간 상태라도 메세지를 주고 받는데 문제가 없다:

```
# alarm 서비스 를 잠시 내려놓음

#메세지 발송처리
http POST localhost:8081/mobiles user=01012345678 receiver=01099998888 text="hello"   #Success

# alarm 서비스 기동
cd alarm
mvn spring-boot:run

#메세지 수신 확인
-처리시간이 상이하더라도 실행되는 결과에는 문제가 없음. (비지니스적으로 시간 Delay는 허용함)
```
![image](https://user-images.githubusercontent.com/48303857/79948095-29e0dd80-84ae-11ea-8750-35f6971a5e74.png)

## CQRS 적용 View
타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이)도 내 서비스의 화면 구성과 잦은 조회가 가능 하도록
Event 방식의 데이터 적제를 통한 메일함 조회 View를 구현함.

```
@StreamListener(KafkaProcessor.INPUT)
    public void whenSaved_then_CREATE_1 (@Payload Saved saved) {
        try {
            if (saved.isMe()) {
                // view 객체 생성
                Readmail readmail = new Readmail();
                // view 객체에 이벤트의 Value 를 set 함
                readmail.setId(saved.getId());
                readmail.setSender(saved.getSender());
                readmail.setReceiver(saved.getReceiver());
                readmail.setText(saved.getText());
                // view 레파지 토리에 save
                readmailRepository.save(readmail);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
![image](https://user-images.githubusercontent.com/48303857/80051102-4df5fa80-8552-11ea-86c6-e7cd8c2c3791.png)



# 운영

## CI/CD 설정 : Azure Devops pipeline 

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure pipeline 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.
![image](https://user-images.githubusercontent.com/48303857/79966563-91a42200-84c8-11ea-9ea7-1f64c5ee7070.png)


Github 소스가 커밋됨과 동시에 pipeline이 실행되며 배포가 된다
![image](https://user-images.githubusercontent.com/48303857/79967770-3c691000-84ca-11ea-9b69-d1c75a62a49d.png)
![image](https://user-images.githubusercontent.com/48303857/79967840-5276d080-84ca-11ea-9ccb-6460ef53105a.png)

$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$

### 오토스케일 아웃 및 무정지 
시스템이 불안정한 경우 사용자의 요청을 100% 받아들여주기 위해 Auto Scailing 적용하여 자동화된 확장 기능을 적용하고자 한다. 


- mobile서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy mobile --min=1 --max=10 --cpu-percent=15
```
- Image를 재배포 하여 pod가 늘어나는지 관찰한다.
```
kubectl set image deployment.apps/mobile mobile=admin23acr.azurecr.io/mobile:latest
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![image](https://user-images.githubusercontent.com/48303857/80052045-a5956580-8554-11ea-8292-967f2b1898d3.png)

![image](https://user-images.githubusercontent.com/48303857/80052226-25bbcb00-8555-11ea-889c-6b8a51e23e18.png)


