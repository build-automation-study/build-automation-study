# build-automation-study
> 우리FISA6기 클라우드 엔지니어링 \
> 리눅스 inotify-tools 실습

## 🧑‍💻 팀원 소개

| <img src="https://avatars.githubusercontent.com/u/181299322?v=4" width="120" height="120" /> | <img src="https://avatars.githubusercontent.com/u/113874212?v=4" width="120" height="120" /> |
|:---:|:---:|
| **신성혁**<br>[@ssh221](https://github.com/ssh221) | **사재헌**<br>[@Zaixian5](https://github.com/Zaixian5) |
| springboot 앱 개발 | bash 스크립트 작성 |


## 1. 프로젝트 목표
리눅스 VM 두 대를 각각 운영서버, 개발서버로 가정하고 inotify-tools를 활용하여 소프트웨어 업데이트를 배포하는 간이 아키텍처를 구성한다.\
이를 바탕으로 Jenkins나 Github Action을 활용해보기 전, 가장 기본적인 CI/CD 원리를 이해해 보고자 한다.

## 2. 아키텍처

```text
+------------------------------+         SSH/SCP (NAT Port Forwarding)           +-------------------------------+
| 개발 서버 (Linux VM)          |  ------------------------------------------->   | 운영 서버 (Linux VM)          |
|                              |         host:2222  ->  operation-vm:22          |                               |
+------------------------------+                                                 +-------------------------------+
| - springapp.jar 빌드/수정     |                                                 | - springapp.jar 실행/배포     |
| - inotifywait로 변경 감지     |                                                 | - 서비스 반영                  |
| - 변경 시 scp 전송            |                                                 |                               |
+-------------------------------+                                                 +-------------------------------+

흐름:
1) 개발 서버에서 파일 변경 발생
2) inotify-tools(inotifywait)가 이벤트 감지
3) scp로 운영 서버에 새 파일 전송 (SSH: 2222 포워딩 경유)
4) 운영 서버 애플리케이션 업데이트 반영
```

- **개발 서버**: 개발자가 코드를 작성하고 테스트하는 환경
- **운영 서버**: 배포된 코드로 실제 서비스가 구동되는 환경
- **inotify-tools**: 파일 변경(생성, 수정, 삭제) 이벤트를 실시간으로 감지하는 리눅스 전용 툴
- **scp**: ssh 기반 파일 전송 프로토콜.

## 3. inotify-tools 스크립트
```bash
#!/bin/bash


JAR_FILE="./myapp.jar"      # 실행할 자바 파일의 이름과 위치
LOG_FILE="./app.log"        # 프로그램의 실행 기록(로그) 위치
TARGET_PORT=8080            # Spring Boot가 사용하는 포트 번호
COOLDOWN=15                 # 중복 실행을 막기 위한 쉬는 시간 (15초)
LAST_RUN=0                  # 마지막으로 실행했던 시간을 기억하는 변수

echo "👀 $JAR_FILE 파일의 변경 감지를 시작합니다..."

# inotifywait: 파일 감시자
#   -m : 감시를 한 번만 하고 끝내지 말고 계속(monitor) 해라.
#   -e close_write,moved_to : 파일에 글을 다 쓰고 닫거나(수정 완료), 새 파일이 덮어씌워질 때(이동)만 알려달라.
# | (파이프) : 감시자가 변화를 발견하면, 그 정보를 바로 아래의 while(반복문)에 넘겨준다.
inotifywait -m -e close_write -e moved_to "$JAR_FILE" |
while read -r directory events filename; do
    # 변화가 감지될 때마다 이 안의 코드가 실행

    # 현재 시간을 초 단위(숫자)로 계산해서 CURRENT_TIME에 저장
    CURRENT_TIME=$(date +%s)
    
    # 쿨다운 체크 (너무 자주 실행되는 것 방지)
    # 현재 시간에서 마지막 실행 시간을 뺀 값이 15초(COOLDOWN)보다 큰지 계산
    if (( CURRENT_TIME - LAST_RUN > COOLDOWN )); then
        echo "=========================================="
        echo "[$(date)] 🚨 $filename 파일이 새로 들어왔습니다!"
        
        # 마지막 실행 시간을 지금 시간으로 업데이트
        LAST_RUN=$CURRENT_TIME

        # 기존에 켜져 있는 Spring Boot 끄기
        # lsof -t -i:8080 -> 8080번 포트를 쓰고 있는 프로그램의 번호(PID)만 찾아냄.
        OLD_PID=$(lsof -t -i:$TARGET_PORT)

        # [ -n "$OLD_PID" ] : OLD_PID가 비어있지 않은지(-n) 검사
        # 즉, 8080 포트를 쓰는 프로그램이 현재 살아있는지 확인
        if [ -n "$OLD_PID" ]; then
            echo "🔥 현재 8080 포트를 사용 중인 프로그램(번호: $OLD_PID)을 종료합니다..."
            
            # kill -15 : 프로세스 종료
            kill -15 "$OLD_PID" 
            
            # 포트가 비워질 때까지 기다림
            MAX_RETRIES=10  # 최대 10번(10초)까지만 기다림
            COUNT=0         # 기다린 횟수를 셀 카운터

            # while 루프: 8080 포트를 쓰는 프로세스가 있는지 계속 확인
            # > /dev/null : 표준 출력 버리기
            while lsof -i:$TARGET_PORT > /dev/null; do
                
                # 만약 10번(10초)을 넘게 기다렸다면 강제로 종료
                if [ $COUNT -ge $MAX_RETRIES ]; then
                    echo "⚠️ 프로그램이 10초 넘게 안 꺼져서 강제로 종료합니다."
                    # kill -9 : 강제 종료
                    kill -9 "$OLD_PID"
                    sleep 2 # 강제로 죽인 후 2초 정도 기다림
                    break
                fi

                echo "⏳ 포트가 비워지기를 기다리는 중... ($((COUNT+1))초)"
                sleep 1 # 1초 기다림
                ((COUNT++)) # 카운터를 1 올림
            done
            echo "✅ 기존 프로그램이 완전히 종료되었습니다."
        else
            # OLD_PID 상자가 비어있다면 (켜져 있는 프로그램이 없다면)
            echo "ℹ️ 현재 8080 포트가 비어있습니다. (종료할 프로그램이 없음)"
        fi

        
        # 새로운 Spring Boot (jar 파일) 실행하기
        echo "🚀 새로운 $JAR_FILE 을 실행합니다..."
        
        # 1) nohup : 터미널이 닫혀도 백그라운드에서 계속 실행
        # 2) java -jar "$JAR_FILE" : 자바 프로그램을 실행
        # 3) >> "$LOG_FILE" : 표준 출력을 화면에 쓰지 말고, app.log 파일의 맨 끝에 계속 이어 적어라(>>)
        # 4) 2>&1 : "에러 메시지(2)"도 정상 로그(1)도 로그 파일에 씀
        # 5) & : 백그라운드 실행
        nohup java -jar "$JAR_FILE" >> "$LOG_FILE" 2>&1 &
        
        echo "🎉 성공! 새로운 버전이 백그라운드에서 돌아가기 시작했습니다. (로그 기록중: $LOG_FILE)"
        echo "=========================================="
        
    else
        # 15초가 지나기 전에 또 파일이 변경되면 무시
        echo "⏳ 아직 쿨다운 15초가 지나지 않아 실행을 무시합니다."
    fi
done
```

## 4. Spring app
    - 설명\
    spring boot app에서 controller에 return 값을 변환하는 방법을 통해 코드가 수정되는 상황을 만들어준다.
    - 과정
    1. spring boot app에서 controller의 return 값을 수정
    2. 수정된 파일을 jar로 재빌드
    3. 수정된 jar 파일을 개발 서버(Linux 가상머신)로 전송
    4. scp (port:2222)를 통해 수정된 jar 파일을 운영 서버로 전송

## 5. 실습 결과
- 업데이트 전 스크립트 실행 화면
![업데이트전](images\inotify-실행-첫화면.png)

- 업데이트 후 스크립트 실행 화면
![업데이트후](images\업데이트-감지-화면.png)