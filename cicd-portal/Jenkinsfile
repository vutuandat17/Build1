// =============================================
// Jenkinsfile - CI/CD cho Resume Analyser (bản cấu hình qua PORTAL)
// Copy file này vào thư mục gốc repo (cùng cấp backend/, frontend/, docker-compose.yml).
//
// File này chỉ chứa CÁC BƯỚC của pipeline. Tool, secret, kết nối SonarQube đều tạo
// trên giao diện web và được gọi ở đây bằng TÊN/ID – tên phải khớp tuyệt đối:
//   Tools        : jdk21, maven3, node20, sonar-scanner     (Manage Jenkins → Tools)
//   Sonar server : SonarQube                                (Manage Jenkins → System)
//   Credentials  : sonar-token        (dùng gián tiếp qua server "SonarQube")
//                  git-credentials    (khai báo trong cấu hình job, để checkout)
//                  registry-credentials (push image)
//                  resume-mongodb-uri (chuỗi kết nối MongoDB khi deploy)
//
// Luồng: Checkout → Unit test (song song) → SonarQube → Quality Gate
//        → Build image → Push image → Deploy → Smoke test
// =============================================
pipeline {
    agent any

    tools {
        jdk    'jdk21'
        maven  'maven3'
        nodejs 'node20'
    }

    // Hiện ở nút "Build with Parameters" (từ lần build thứ 2)
    parameters {
        booleanParam(name: 'DEPLOY', defaultValue: true,
                     description: 'Build image và deploy bằng docker compose sau khi qua Quality Gate')
        booleanParam(name: 'PUSH_IMAGE', defaultValue: false,
                     description: 'Push image lên registry (cần credential "registry-credentials")')
        string(name: 'REGISTRY', defaultValue: 'docker.io',
               description: 'Registry để push image, ví dụ docker.io hoặc registry.gitlab.com')
    }

    // Tự build khi có commit mới (Jenkins hỏi Git mỗi ~2 phút).
    // Có hiệu lực sau lần build đầu tiên – Jenkins đọc Jenkinsfile rồi mới đăng ký trigger.
    triggers {
        pollSCM('H/2 * * * *')
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        // Gọi JaCoCo bằng dòng lệnh → không cần sửa pom.xml vẫn có báo cáo coverage
        JACOCO        = 'org.jacoco:jacoco-maven-plugin:0.8.15'
        APP_VERSION   = "v0.${BUILD_NUMBER}"
        COMPOSE_PROJECT_NAME = 'resume-analyser'
        // Port app mở ra máy host – docker-compose.yml của app và smoke test cùng đọc
        BACKEND_PORT  = '8180'
        FRONTEND_PORT = '3100'
        CI            = 'true'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '=== Bước 1: Lấy mã nguồn (Jenkins tự checkout bằng credential khai báo trong job) ==='
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
                echo "Commit ${env.GIT_COMMIT_SHORT} -> phiên bản ${env.APP_VERSION}"
            }
        }

        stage('Unit test') {
            parallel {
                stage('Backend') {
                    steps {
                        echo '=== Bước 2a: JUnit cho backend + đo coverage (JaCoCo) ==='
                        dir('backend') {
                            sh 'mvn -B -ntp clean $JACOCO:prepare-agent verify $JACOCO:report'
                        }
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Frontend') {
                    steps {
                        echo '=== Bước 2b: Cài thư viện và chạy test cho frontend ==='
                        dir('frontend') {
                            sh 'npm ci'
                            // Ưu tiên script "test:ci" (có coverage + JUnit report), nếu chưa có thì dùng "npm test"
                            sh '''
                                if node -e "process.exit(require('./package.json').scripts?.['test:ci'] ? 0 : 1)"; then
                                    npm run test:ci
                                else
                                    npm test
                                fi
                            '''
                        }
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'frontend/reports/*.xml'
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '=== Bước 3: Quét mã nguồn bằng SonarQube ==='
                // 'SonarQube' = tên server khai báo ở Manage Jenkins → System.
                // Plugin tự lấy URL + token (credential sonar-token) và đưa vào biến môi trường cho scanner.
                withSonarQubeEnv('SonarQube') {
                    sh '"$SCANNER_HOME/bin/sonar-scanner" -Dsonar.projectVersion=$APP_VERSION'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '=== Bước 4: Chờ SonarQube báo kết quả Quality Gate (fail → dừng, không deploy) ==='
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build image') {
            when { expression { params.DEPLOY || params.PUSH_IMAGE } }
            steps {
                echo '=== Bước 5: Build image Docker cho backend và frontend ==='
                sh '''
                    docker compose build
                    docker tag ${COMPOSE_PROJECT_NAME}-backend:latest  ${COMPOSE_PROJECT_NAME}-backend:${APP_VERSION}
                    docker tag ${COMPOSE_PROJECT_NAME}-frontend:latest ${COMPOSE_PROJECT_NAME}-frontend:${APP_VERSION}
                '''
            }
        }

        stage('Push image') {
            when { expression { params.PUSH_IMAGE } }
            steps {
                echo "=== Bước 6: Push image lên ${params.REGISTRY} ==="
                // Username/token lấy từ credential "registry-credentials"; Jenkins tự che (****) trong log
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REGISTRY_USER',
                                                  passwordVariable: 'REGISTRY_TOKEN')]) {
                    // Dùng nháy đơn ''' để shell (không phải Groovy) đọc secret → secret không lọt vào log
                    sh '''
                        echo "$REGISTRY_TOKEN" | docker login "$REGISTRY" -u "$REGISTRY_USER" --password-stdin
                        for svc in backend frontend; do
                            target="$REGISTRY/$REGISTRY_USER/resume-$svc"
                            docker tag "$COMPOSE_PROJECT_NAME-$svc:$APP_VERSION" "$target:$APP_VERSION"
                            docker tag "$COMPOSE_PROJECT_NAME-$svc:$APP_VERSION" "$target:latest"
                            docker push "$target:$APP_VERSION"
                            docker push "$target:latest"
                        done
                    '''
                }
            }
            post {
                always {
                    sh 'docker logout "$REGISTRY" || true'
                }
            }
        }

        stage('Deploy') {
            when { expression { params.DEPLOY } }
            steps {
                echo '=== Bước 7: Deploy bằng docker compose ==='
                // Chuỗi kết nối MongoDB là secret → lấy từ credential "resume-mongodb-uri",
                // docker compose đọc biến SPRING_MONGODB_URI và truyền vào container backend
                withCredentials([string(credentialsId: 'resume-mongodb-uri', variable: 'SPRING_MONGODB_URI')]) {
                    sh 'docker compose up -d --remove-orphans'
                }
                sh 'docker compose ps'
            }
        }

        stage('Smoke test') {
            when { expression { params.DEPLOY } }
            steps {
                echo '=== Bước 8: Kiểm tra app sau deploy còn sống ==='
                sh '''
                    check() {  # check <tên> <url>: chờ tối đa 120s tới khi URL trả về HTTP < 500
                        for i in $(seq 1 40); do
                            code=$(curl -s -o /dev/null -w '%{http_code}' "$2" || true)
                            if [ "$code" != "000" ] && [ "$code" -lt 500 ]; then
                                echo "$1 OK ($2 -> HTTP $code)"; return 0
                            fi
                            sleep 3
                        done
                        echo "$1 KHÔNG phản hồi: $2"; return 1
                    }
                    check Frontend "http://host.docker.internal:$FRONTEND_PORT"
                    check Backend  "http://host.docker.internal:$BACKEND_PORT"
                '''
            }
            post {
                failure {
                    sh 'docker compose ps; docker compose logs --tail=80'
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline thành công: ${env.APP_VERSION} (${env.GIT_COMMIT_SHORT})" +
                 (params.PUSH_IMAGE ? " – đã push lên ${params.REGISTRY}" : '') +
                 (params.DEPLOY ? ' – đã deploy.' : '.')
        }
        failure {
            echo '❌ Pipeline thất bại: xem log ở stage bị đỏ. Lỗi Quality Gate → mở SonarQube để xem chi tiết.'
        }
    }
}
