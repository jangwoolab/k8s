# <실습 문제: 롤링 업데이트 실습 : 실습 시간 4시간 - 발표 1시간 : 조별 1명 >

### 각 스텝별 사용한 명령어와 캡쳐 이이지를 넣어서 자신의 notion 에 정리해서 pdf 로 제출 하시오. 
	파일명 : 1조_홍길동.pdf
	제출 : p.jangwoo@gmail.com

## STEP 1. nginx:latest 버젼을 다운 받아서 3개의 도커이미지를 생성 하세요.  : [docker commit을 이용해서 도커 이미지 생성하세요.]

- 참고: docker-host VM에서 작업 하시오. 
- 참고: /usr/share/nginx/html/index.html
- 참고: `<body style="background-color: red;">`

```bash
nginx:v1.0   		<== 배경색 [red]
nginx:v2.0		 	<== 배경색 [blue]
nginx:v3.0		 	<== 배경색 [yellow] 
nginx:latest		<== 배경색 [white]
```

## STEP 2 : docker-host VM의 각 포트로 접속시           <== 출력 내용을 URL을 포함 해서 캡쳐 잡으시오. [제출시 필요] 

	http://10.10.10.200:8081/   접속시 : 배경색 [red]
	http://10.10.10.200:8082/ 	접속시 : 배경색 [blue]
	http://10.10.10.200:8083/ 	접속시 : 배경색 [yellow] 
	http://10.10.10.200:8084/ 	접속시 : 배경색 [white]

## STEP 3. 자신의 도커 허브의 레파지토리에 PUSH 하세요. [위 에서 생성한 4개 도커 이미지 모두 push]     <== 출력 내용을 캡쳐 잡으시오. [레파지토리의 모든 버젼이 출력되도록]

## STEP 4. Deployment를 사용해서 nginx:v1.0을 다운받아서 nginx-web이라는 이름으로 3개의 pod를 작동 시키세요. 

	히스토리에서 알수 있도록 어노테이션 넣으세요. 

```bash
		kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx:1.0-red"
```

## STEP 5. Deployment/nginx-web 을 port 80 으로 LoadBalancer 를 사용해서 노출 시키시오.  LoadBalancer IP로 접속시 접속 확인하시오. 

	확인[예시] : http://10.10.10.50       으로 접속시 배경이 red 로 출력됨 <== 출력 내용을 캡쳐 잡으시오. 

## STEP 6. Deployment의 롤링 업데이트를 적용해서 nginx:v2.0 로 롤링 업데이트 하세요. 
	확인[예시] : http://10.10.10.50       으로 접속시 배경이 blue 로 출력됨 <== 출력 내용을 캡쳐 잡으시오. 
	히스토리에서 알수 있도록 어노테이션 넣으세요. 
		kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx:2.0-blue"

## STEP 7. Deployment의 롤링 업데이트를 적용해서 nginx:v3.0 로 롤링 업데이트 하세요. 
	확인[예시] : http://10.10.10.50       으로 접속시 배경이 yellow 로 출력됨 <== 출력 내용을 캡쳐 잡으시오. 
	히스토리에서 알수 있도록 어노테이션 넣으세요. 
		kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx:3.0-yellow"

## STEP 8. Deployment의 롤링 업데이트를 적용해서 nginx:latest 로 롤링 업데이트 하세요. 
	확인[예시] : http://10.10.10.50       으로 접속시 배경이 white 로 출력됨 <== 출력 내용을 캡쳐 잡으시오. 
	히스토리에서 알수 있도록 어노테이션 넣으세요. 
		kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx:latest-white"

## STEP 9. 이전 버젼으로 롤백 하시오.  [출력 내용 캡쳐]
	확인[예시] : http://10.10.10.50       으로 접속시 배경이 yellow 로 출력됨 <== 출력 내용을 캡쳐 잡으시오.

## STEP 10. RED 버젼으로 롤백 하시오. [캡쳐]
	확인[예시] : http://10.10.10.50       으로 접속시 배경이 red 로 출력됨 <== 출력 내용을 캡쳐 잡으시오.

## STEP 10. Deployment/nginx-web 의 파드를 6개로 확장 하시오.[scale out]  <== 확장된 파드를 캡쳐 잡으시오. 

## STEP 11. Deployment/nginx-web 의 파드를 3개로 축소 하시오.[scale in]  <== 축소된 파드를 캡쳐 잡으시오. 






















