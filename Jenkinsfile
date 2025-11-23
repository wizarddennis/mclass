pipeline {
    agent any  // 어떤 실행 서버에서든 실행 가능

    tools  {
        maven 'maven 3.9.11' // Jenkins에 등록된 maven 3.9.11을 사용
    }

    environment {
        // 배포에 필요한 변수 설정
        DOCKER_IMAGE = "demo-app" //도커 이미지 이름
        CONTAINER_NAME = "springboot-container"  // 도커 컨테이너 이름
        JAR_FILE_NAME = "app.jar"  // 복사할 JAR 파일 이름
        PORT = "8081"  // 컨테이너와 연결할 포트

        REMOTE_USER = "ec2-user" // 원격(spring) 서버 사용자
        REMOTE_HOST =  "3.38.123.199"// // 원격 Spring 서버 IP(public ip)

        REMOTE_DIR = "/home/ec2-user/deploy" // 원격 서버에 파일 복사할 경로
        SSH_CREDENTIALS_ID = "b7cc6005-e16d-4b90-97ff-92cc50c0d3f4" // Jenkins SSH 자격 증명 ID
    }

    // 단계별로 실행될 코드 작성
    stages {
        stage('Git Checkout') {
            steps { // 실행될 실제 명령어 작성
                // Jenkins가 연결될 Git 저장소에서 최신 코드 체크아웃
                // SCM(Source Code Management)
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                // 테스트는 건너뛰고 Maven 빌드. sh는 쉘스크립트를 사용하겠다라는 의미.
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Prepare Jar') {
            steps {
                // 빌드 결과물 JAR 파일을 지정한 이름(app.jar)으로 복사
                sh 'cp target/demo-0.0.1-SNAPSHOT.jar ${JAR_FILE_NAME}'
            }
        }

        stage('Copy to Remote Server') {
            steps {
                // Jenkins 서버가 원격 서버에 SSH 접속할 수 있도록 sshagent 사용
                // plugin이 설치되어야 함. environment 가 아니고 env 임. 요상함.
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    // 원격 서버에 배포 디렉토리 생성(없으면 새로 만듦)
                    sh "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} \"mkdir -p ${REMOTE_DIR}\""
                    // Jar 파일과 Dockerfile을 원격 서버에 복사
                    sh "scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${JAR_FILE_NAME} Dockerfile ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"
                }
            }
        }

        stage('Remote Docker Build & Deploy') {
            steps {
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${REMOTE_USER}@${REMOTE_HOST} << ENDSSH
                            cd ${REMOTE_DIR} || exit 1
                            docker rm -f ${CONTAINER_NAME} || true
                            docker build -t ${DOCKER_IMAGE} .
                            docker run -d --name ${CONTAINER_NAME} -p ${PORT}:${PORT} ${DOCKER_IMAGE}
                        ENDSSH
                    """
            }
        }
    }
}