pipeline {
    agent {
        kubernetes {
            // Jenkins가 동적으로 빌드용 파드를 생성하도록 설정
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  namespace: jenkins
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - sleep
    args:
    - infinity
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker/config.json  # Kaniko에서 사용할 Docker 인증 정보 경로
      subPath: .dockerconfigjson
  volumes:
  - name: docker-config
    secret:
      secretName: regcred  # Docker Registry 인증 정보 Secret
            '''
        }
    }

    stages {
        stage('Checkout') {
            steps {
                // Git 저장소에서 소스 코드 체크아웃
                git branch: 'main',
                    url: 'https://github.com/leeeeejieun/k8s-source-repo.git',
                    credentialsId: 'admin'
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    // 이미지 태그를 정의 (필요시 동적 생성 가능)
                    def customTag = "v1.0"
                    env.DOCKER_IMAGE = "arjleun581/ci-cd:${customTag}"
                    echo "Docker Image Tag set to ${env.DOCKER_IMAGE}"  // 로그에 출력
                }
            }
        }

        stage('Check Workspace') {
            steps {
                sh '''
                    # 현재 Jenkins workspace 확인
                    echo "Current workspace is: $WORKSPACE"
                    ls -al $WORKSPACE
                '''
            }
        }

        stage('Build and Push with Kaniko') {
            steps {
                // 'kaniko' 컨테이너 안에서 빌드 & 이미지 푸시 실행
                container('kaniko') {
                    script {
                        sh '''
                        /kaniko/executor --context ${WORKSPACE}/source \
                                         --dockerfile ${WORKSPACE}/source/Dockerfile \
                                         --destination ${DOCKER_IMAGE}
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()  // 빌드 후 워크스페이스 정리
        }
    }
}
