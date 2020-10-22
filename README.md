# 서비스 시나리오

기능적 요구사항
1. 사용자가 멤버십에 가입한다.
1. 멤버십에 가입이 되면 포인트가 부여됩니다.
1. 멤버십 탈퇴를 요청할 수 있다.
1. 멤버십 탈퇴가 요청 되면 가지고 있는 포인트는 모두 삭제됩니다.
1. 멤버십 가입 및 탈퇴 시 부여된 포인트를 조회 가능합니다.

# 분석/설계

## Event Storming
![eventStorming](https://user-images.githubusercontent.com/67616972/96808289-815b0880-1453-11eb-8c73-cc3727e7f4d3.JPG)

## 구현
- CQRS
- 폴리글랏퍼시스턴스
    - Member : H2
      		<dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
          </dependency>
    - Mileage : HSQLDB
          <dependency>
            <groupId>org.hsqldb</groupId>
            <artifactId>hsqldb</artifactId>
            <version>2.4.0</version>
            <scope>runtime</scope>
          </dependency>
    - Report : H2
      		<dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
          </dependency>

- 폴리글랏 프로그래밍
* ![폴리글랏 프로그래밍](https://user-images.githubusercontent.com/67616972/96808282-7f914500-1453-11eb-8a5b-11d5a5ccd54a.JPG)

## 동기식 호출 처리

* 멤버십 탈퇴 시 부여받은 포인트를 삭제하여 일관성을 유지하는 트랜잭션으로 처리
* FeignClient 방식을 통해서 Request-Response 처리.

```
# MemberApplication.java

package membership;
import membership.config.kafka.KafkaProcessor;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.openfeign.EnableFeignClients;


@SpringBootApplication
@EnableBinding(KafkaProcessor.class)
@EnableFeignClients
public class MemberApplication {
    protected static ApplicationContext applicationContext;
    public static void main(String[] args) {
        applicationContext = SpringApplication.run(MemberApplication.class, args);
    }
}
```

- 멤버십 탈퇴 전 포인트삭제 요청

```
# MemberMgmt.java (Entity)

    @PreRemove
    public void PreRemove(){
        Seceded seceded = new Seceded();
        BeanUtils.copyProperties(this, seceded);
        seceded.setStatus("end member");
        seceded.publishAfterCommit();

        membership.external.MileageMgmt mileageMgmt = new membership.external.MileageMgmt();
        mileageMgmt.setId(seceded.getMileageId());
        mileageMgmt.setMemberId(seceded.getId());
        mileageMgmt.setPoint(0);
        mileageMgmt.setStatus("removeRequest");
        MemberApplication.applicationContext.getBean(membership.external.MileageMgmtService.class)
            .mileageDelete(mileageMgmt);
    }
```


## 비동기식 호출

* 멤버십 가입을 하면 비동기식 방식으로 포인트를 부여   
```
package membership;
...
@Entity
@Table(name="MemberMgmt_table")
public class MemberMgmt {

...

    @PostPersist
    public void onPostPersist(){
        Signed signed = new Signed();
        BeanUtils.copyProperties(this, signed);
        signed.setStatus("No Point");
        signed.publishAfterCommit();
    }
...
}
```

- Mileage 서비스에서는 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현

```
package membership;
...
@Service
public class PolicyHandler{
   ...
   @Autowired
   MileageMgmtRepository mileageMgmtRepository;

   @StreamListener(KafkaProcessor.INPUT)
   public void wheneverSigned_MileageGive(@Payload Signed signed){

        if(signed.isMe()){
            MileageMgmt mileageMgmt = new MileageMgmt();
            mileageMgmt.setMemberId(signed.getId());
            mileageMgmt.setStatus("give point");
            mileageMgmt.setPoint(10000000);

            mileageMgmtRepository.save(mileageMgmt);
        }
    }
   ...
}
```

#멤버십 가입
http POST http://localhost:8081/memberMgmts name='Test1' grade='clasic'

#포인트 부여 확인
http GET http://localhost:8082/mileageMgmts/1   

#멤버십 탈퇴
http DELETE http://localhost:8081/memberMgmts/1

#멤버십 가입 및 탈퇴 내역 확인
http GET http://localhost:8083/reports

# 운영

## CI/CD 설정

# Repository 구성
![Repository 구성](https://user-images.githubusercontent.com/67616972/96808291-81f39f00-1453-11eb-9e33-6c)ab98f2bba9.JPG

# EKS 클러스터 구성 
![EKS 클러스터 구성](https://user-images.githubusercontent.com/67616972/96808287-80c27200-1453-11eb-818b-7acb9c29e7ff.JPG)

# ECR Repository 구성 (서비스 별 )
![ECR Repository 구성](https://user-images.githubusercontent.com/67616972/96808284-8029db80-1453-11eb-9c38-b7db3ea49067.JPG)



## 클라우드 환경 서비스 테스트 (gateway 외부 URL)

# 멤버십 가입
![멤버십가입](https://user-images.githubusercontent.com/67616972/96808272-7dc78180-1453-11eb-8bc5-e786becd0c60.JPG)

# 멤버십 가입 후 포인트 부여
![멤버십가입2](https://user-images.githubusercontent.com/67616972/96808276-7dc78180-1453-11eb-8147-ad3a5ded5f26.JPG)

# 부여된 포인트 확인
![포인트 부여 확인](https://user-images.githubusercontent.com/67616972/96808281-7f914500-1453-11eb-8d75-53adf4dd47fd.JPG)

# 멤버십 탈퇴
![멤버십탈퇴](https://user-images.githubusercontent.com/67616972/96808280-7ef8ae80-1453-11eb-9437-a95bf01aeaf8.JPG)

# 멤버십 내역 조회
![멤버십가입내역](https://user-images.githubusercontent.com/67616972/96808278-7e601800-1453-11eb-8d2f-b7bcaae11d1c.JPG)

## Configmap , 설정의 외부 주입을 통한 유연성을 제공

* Configmap - rental 서비스 deployment.yaml 구성예
![deployment](https://user-images.githubusercontent.com/67616972/96565408-4942b100-12ff-11eb-870c-3b6d3cdd9262.JPG)

* Configmap 을 기반으로 한 k8s deployment 구성
![configMap 서비스](https://user-images.githubusercontent.com/67616972/96565562-7e4f0380-12ff-11eb-83f6-20628d5c36de.JPG)


## istio (사이드카 패턴) 삽입 및 모니터링 구성 

* team-rent 네임스페이스 사이드카 삽입을 통한 k8s deployment 구성
![istio_1](https://user-images.githubusercontent.com/67616972/96565743-b9513700-12ff-11eb-962e-708a6d50b4f7.JPG)

* kiali 모니터링 - MSA 간 서비스 호출 구조도 트렉킹
![istio_2](https://user-images.githubusercontent.com/67616972/96565826-d5ed6f00-12ff-11eb-9171-a9212a2dd586.JPG)

* Jeager 모니터링 - Rest API 호출 트렉킹 
![istio_3](https://user-images.githubusercontent.com/67616972/96565852-db4ab980-12ff-11eb-97a7-ccec04968e2f.JPG)


## LivenessProbs 설정 적용 시뮬레이션

* configmap deployment.yml 파일 livenessprobs 설정 (product 서비스)
* http rest 호출을 통한 서비스 liveness 상태 확인 적용
* 임의로 비정상적인 url 을 설정하여 비정상 적인 상태 감지 적용 
* (정상 URL : /products , 비정상 URL : /productsTest)
![LivelessConfig](https://user-images.githubusercontent.com/67616972/96566198-3aa8c980-1300-11eb-80ee-4e6bc593dc16.JPG)

* LivenessProbs 설정에 의해 pod container 가 가용성이 미확보 되었다고 판단 
* pod 재생성 및 container 서비스 구동 확인 
![LivelessConfig_1](https://user-images.githubusercontent.com/67616972/96566575-b4d94e00-1300-11eb-863a-114a42fea702.JPG)
![LivelessConfig_2](https://user-images.githubusercontent.com/67616972/96566577-b571e480-1300-11eb-89fe-0474a55ae495.JPG)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함
* 시나리오는 상품(product)-->렌탈(rental) 연결을 기준으로 시뮬레이션이 주성되었고, 상품등록 요청이 과도할 경우 CB 를 통하여 장애격리.

* application.yml 파일 수정, Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
![image](https://user-images.githubusercontent.com/23513745/96563655-26170200-12fd-11eb-8f46-2e71f19f56db.png)

* Product 서비스 product entity prepoersist 에 쓰레드 및 sleep 을 통한 부하처리 적용 
![image](https://user-images.githubusercontent.com/23513745/96563674-2e6f3d00-12fd-11eb-8b2c-072112a9cd71.png)

* 부하발생 테스터 siege(워크로드) 툴을 통한 서킷브레이커 동작확인 
* 속적으로 회로 열림과 담힘 확인 (istio 제어)
![서킷브레이크_1](https://user-images.githubusercontent.com/67616972/96568997-8d37b500-1303-11eb-8076-3df89c4e3496.JPG)
![서킷브레이크_2](https://user-images.githubusercontent.com/67616972/96569005-8f017880-1303-11eb-9202-0627d0d6b9ed.JPG)


## 오토스케일 아웃(HPA)

* prouct 서비스 configmap 에 resource 설정 추가 
![image](https://user-images.githubusercontent.com/67616972/96568422-e6531900-1302-11eb-92e1-3524709fbe5d.png)

* kubelet, product deploy 오토스케일 아웃 설정
![HPA_1](https://user-images.githubusercontent.com/67616972/96568614-1995a800-1303-11eb-889b-b011256bbd3f.JPG)

* siege(워크로드) 서비스 부하 발생에 따른 pod 증가 확인 
![HPA_3](https://user-images.githubusercontent.com/67616972/96568738-3cc05780-1303-11eb-8224-10e57bb68903.JPG)

* 부하발생이 진행됨에 따라, 지속적으로 pod 갯수 증가 모니터링
![HPA_8](https://user-images.githubusercontent.com/67616972/96568855-5a8dbc80-1303-11eb-8bae-4e25453a1d5d.JPG)


## 무정지 재배포

* ReadnessProbs 설정후 무중단 배포 확인
![image](https://user-images.githubusercontent.com/67616972/96568422-e6531900-1302-11eb-92e1-3524709fbe5d.png)

* siege(워크로드) 서비스 로그 결과 가용성 100% 확인 
![무중단배포_4(100%)](https://user-images.githubusercontent.com/67616972/96567171-755f3180-1301-11eb-9902-877c5697b70d.JPG)

* product 서비스 deployment 이미지 변경으로 인한 신규 image 레플리카셋 기반 pod 생성 과정 모니터링
![무중단배포_1](https://user-images.githubusercontent.com/67616972/96567009-3cbf5800-1301-11eb-800d-06eec7f25d8e.JPG)

* product 서비스 deployment image 변경 히스토리 확인 
![무중단배포_3](https://user-images.githubusercontent.com/67616972/96567087-582a6300-1301-11eb-89ff-6328416ef328.JPG)

* ReadnessProbs 설정 제거후 이미지 변경, siege(워크로드) 모니터링 결과 가용성 미확보 확인 (100% -> 27.33%)
![무중단배포_5(readness 미적용)](https://user-images.githubusercontent.com/67616972/96567319-99227780-1301-11eb-8a31-b34a52497ae1.JPG)





