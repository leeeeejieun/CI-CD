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

    // Git 저장소에서 소스 코드 가져오기
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/leeeeejieun/k8s-source-repo.git',
                credentialsId: 'admin'
            }
        }

        // 빌드 번호를 사용하여 Docker 이미지 태그 설정
        stage('Set Image Tag') {
            steps {
                script {
                    env.DOCKER_IMAGE = "arjleun581/ci-cd:${BUILD_NUMBER}.0"
                    echo "Docker Image Tag set to ${env.DOCKER_IMAGE}"  // 로그에 출력
                }
            }
        }

         // 'kaniko' 컨테이너 안에서 이미지 빌드 및 푸시 실행
        stage('Build and Push with Kaniko') {
            steps {
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

        // ArgoCD용 Manifest Repository 업데이트
        stage('ArgoCD Manifest Update') {
            steps {
                script {
                    // Jenkins 자격 증명(admin) 가져오기
                    withCredentials([usernamePassword(credentialsId: 'admin', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                        def encodedPassword = URLEncoder.encode("$GIT_PASS", 'UTF-8')
                        
                        sh """
                        # 깃허브 작성자 정보 등록
                        git config --global user.email "arjleun19@gmail.com"
                        git config --global user.name "leeeeejieun"

                        # 소스 코드 가져오기
                        rm -rf k8s-manifest-repo
                        git clone https://${GIT_USER}:${encodedPassword}@github.com/leeeeejieun/k8s-manifest-repo.git
                        cd k8s-manifest-repo/app/overlays/dev

                        # 이미지 태그 업데이트
                        sed -i "s|value: arjleun581/ci-cd:.*|value: arjleun581/ci-cd:${BUILD_NUMBER}.0|g" kustomization.yml


                        # 소스코드 업데이트
                        git add .
                        git commit -m "Update image to ${DOCKER_IMAGE}" 
                        git push origin main
                        """
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
