### GCP 서버 생성하기 
영역 : us-central-a
이미지 : ubuntu-1604-xenial-v20200722
크기 : 10GB (남은 크레딧이 적어서 최대한 작게 만듬)
방화벽 : HTTP/HTTPS 허용

---
### GCP DB 사용하기 , 연결까지 완료
GCP SQL MySQL5.7 로 사용
크기 : 10GB
---
### GCP 서버에 배포하기

git에 있는 프로젝트를 clone 해온다.
프로젝트 경로 직전 디렉토리에 배포 스크립트 작성한다.
```shell script
#!/bin/bash

REPOSITORY=/home/pwj4119/app/step1
# 디렉토리 주소 경로 변수에 저장
PROJECT_NAME=webp
# clone 받았던 디렉토리 변수 저장
cd $REPOSITORY/$PROJECT_NAME/
# clone 받았던 디렉토리로 이동
echo "> Git Pull"

git pull
# master 브랜치의 최신 내용을 받는다.
echo "> 프로젝트 Build 시작"

./gradlew build
# 프로젝트 내부의 graldew로 build를 수행한다. 만약 build 수행권한이 없으면 "chmod -x ./gradlew" 수행한다.


echo "> step1 디렉토리로 이동"

cd $REPOSITORY

echo "> Build 파일 복사"

cp $REPOSITORY/$PROJECT_NAME/build/libs/*.jar $REPOSITORY/
# build 결과물인 jar 파일을 복사해 jar 파일을 모아둔 위치로 복사한다.
echo "> 현재 구동중인 애플리케이션 pid 확인"

CURRENT_PID=$(pgrep -f ${PROJECT_NAME}*.jar)
# 기존에 수행중이던 스프링 부트 애플리케이션을 종료하기 위해 process id만 추출한다, -f 옵션은 프로세스 이름으로 찾는다.
echo "현재 구동중인 애플리케이션 pid: $CURRENT_PID"

if [ -z "$CURRENT_PID" ]; then
        echo ">현재구동중인 애플리케이션이 없으므로 종료하지 않습니다."
else
        echo "> kill -15 $CURRENT_PID"
        kill -15 $CURRENT_PID
        sleep 5
fi

echo "> 새애플리케이션 배포"
JAR_NAME=$(ls -tr $REPOSITORY/ | grep *.jar | tail -n 1)
# 새로 실행할 jar 파일명을 찾는다. 여러 jar파일이 생기기 때문에 tail -n로 가장 나중의 jar파일을 변수에 저장한다.
echo "> JAR_NAME:$JAR_NAME"

echo "> $REPOSITORY"

nohup java -jar \
        -Dspring.config.location=classpath:/application.properties,/home/pwj4119/app/application-oauth.properties \
        $REPOSITORY/$JAR_NAME 2>&1 &
# nohup은 쉘 스크립트파일을 데몬 형태로 실행 시키는 명령어, 터미널이 끊겨도 실행한 프로세스는 계속 동작하게 한다.
# -Dspring.config.location 스프링 설정 파일 위치를 지정한다. 
```

MySQL 드라이버를 build.gralde에 등록하고 apllication-real-db.properties 파일을 생성
applciation-real-db.properties 쓸수 있도록
```shell script
...
nohup java -jar \
	-Dspring.config.location=classpath:/application.properties,/home/pwj4119/app/application-oauth.properties,/home/pwj4119/app/application-real-db.properties \
	-Dspring.profiles.active=real \
	$REPOSITORY/$JAR_NAME 2>&1 &
# deploy.sh nohpup 부분을 수정해야 된다.
```

Travis CI는 깃허브에서 제공하는 무료 CI 오픈소스 웹 서비스이다. 또한 젠킨스도 무료이긴 하지만 젠킨스는 설치형이다.
```yaml
language: java
jdk:
  - openjdk8

branches:
  only:
    - master

before_install:
  - chmod +x gradlew

# Travis CI 서버의 HOME
cache:
  directories:
    - '$HOME/.m2/repository'
    - '$HOME/.gradle'

script: "./gradlew clean build"
# deploy 명령어가 실행되기전에 수행
before_deploy:
  - zip -r webp *
  - mkdir -p deploy
  - mv webp.zip deploy/webp.zip

deploy:
  - provider: gcs
    access_key_id: $GPC_ACCESS_KEY
    secret_access_key: $GCP_SECRET_KEY
    bucket: freelec-springboot-build
    skip_cleanup: true
    acl: private
    local_dir: deploy
    wait-until-delpoy: true
notifications:
  email:
    recipients:
      - smartwj@naver.com
```

GCP로 진행하다 지속적 배포 부분에서 AWS와 GCP 서비스 차이점에 대한 이해가 부족한거 같아 다시 AWS로 진행 마무리하고 추후에 GCP로 변경할 예정이다.