// =============================================
// Job kiểm tra cấu hình (bản PORTAL) – dán toàn bộ file này vào job Pipeline:
//   New Item → "cicd-selftest" → Pipeline → Definition: "Pipeline script"
//   → dán nội dung → tick "Use Groovy Sandbox" → Save → Build Now
// Kiểm tra: tool (tự cài lần đầu), Docker, credential, SonarQube + webhook + Quality Gate.
// Sinh 1 project mẫu (Java + JS) ngay trong workspace, không cần repo Git.
// =============================================
pipeline {
    agent any

    tools {
        jdk    'jdk21'
        maven  'maven3'
        nodejs 'node20'
    }

    options {
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        JACOCO       = 'org.jacoco:jacoco-maven-plugin:0.8.15'
        CI           = 'true'
    }

    stages {
        stage('Toolchain') {
            steps {
                echo '=== Bước 1: Kiểm tra tool (lần đầu Jenkins sẽ tự tải Maven, Node, SonarScanner) ==='
                sh '''
                    java -version
                    mvn -v
                    echo "node $(node -v) / npm $(npm -v)"
                    "$SCANNER_HOME/bin/sonar-scanner" --version
                    docker version --format 'Docker client {{.Client.Version}} -> server {{.Server.Version}}'
                    docker compose version
                '''
            }
        }

        stage('Credentials') {
            steps {
                echo '=== Kiểm tra các credential đã tạo trên portal (thiếu ID nào → stage này đỏ) ==='
                withCredentials([string(credentialsId: 'sonar-token', variable: 'C_SONAR'),
                                 usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'C_REG_USER', passwordVariable: 'C_REG_TOKEN'),
                                 string(credentialsId: 'resume-mongodb-uri', variable: 'C_MONGO')]) {
                    // Chỉ kiểm tra "có giá trị", không in secret ra log
                    sh '''
                        test -n "$C_SONAR" && test -n "$C_REG_USER" && test -n "$C_REG_TOKEN" && test -n "$C_MONGO"
                        echo "sonar-token, registry-credentials, resume-mongodb-uri: OK"
                    '''
                }
            }
        }

        stage('Sinh project mẫu') {
            steps {
                echo '=== Bước 2: Tạo project mẫu (Java + JS) trong workspace ==='
                deleteDir()
                writeFile file: 'backend/pom.xml', text: '''<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>demo</groupId>
  <artifactId>selftest</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.13.4</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.5.4</version>
      </plugin>
    </plugins>
  </build>
</project>
'''
                writeFile file: 'backend/src/main/java/demo/Calculator.java', text: '''package demo;

public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }

    public int divide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("b must not be 0");
        }
        return a / b;
    }
}
'''
                writeFile file: 'backend/src/test/java/demo/CalculatorTest.java', text: '''package demo;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.api.Test;

class CalculatorTest {
    private final Calculator calc = new Calculator();

    @Test
    void addsTwoNumbers() {
        assertEquals(5, calc.add(2, 3));
    }

    @Test
    void rejectsDivideByZero() {
        assertThrows(IllegalArgumentException.class, () -> calc.divide(1, 0));
    }
}
'''
                writeFile file: 'frontend/package.json', text: '''{
  "name": "selftest-frontend",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "test:ci": "mkdir -p reports coverage && node --test --experimental-test-coverage --test-reporter=spec --test-reporter-destination=stdout --test-reporter=junit --test-reporter-destination=reports/junit.xml --test-reporter=lcov --test-reporter-destination=coverage/lcov.info"
  }
}
'''
                writeFile file: 'frontend/src/format.js', text: '''export function formatScore(score) {
  if (score < 0 || score > 100) {
    throw new RangeError('score must be between 0 and 100');
  }
  return score + '%';
}
'''
                writeFile file: 'frontend/test/format.test.js', text: '''import { test } from 'node:test';
import assert from 'node:assert/strict';
import { formatScore } from '../src/format.js';

test('formats a valid score', () => {
  assert.equal(formatScore(85), '85%');
});

test('rejects an out-of-range score', () => {
  assert.throws(() => formatScore(120), RangeError);
});
'''
                writeFile file: 'sonar-project.properties', text: '''sonar.projectKey=cicd-selftest
sonar.projectName=CI/CD self-test
sonar.sourceEncoding=UTF-8
sonar.sources=backend/src/main/java,frontend/src
sonar.tests=backend/src/test/java,frontend/test
sonar.java.binaries=backend/target/classes
sonar.java.test.binaries=backend/target/test-classes
sonar.junit.reportPaths=backend/target/surefire-reports
sonar.coverage.jacoco.xmlReportPaths=backend/target/site/jacoco/jacoco.xml
sonar.javascript.lcov.reportPaths=frontend/coverage/lcov.info
'''
            }
        }

        stage('Unit test') {
            parallel {
                stage('Backend (JUnit + JaCoCo)') {
                    steps {
                        echo '=== Bước 3a: JUnit + coverage JaCoCo cho Java ==='
                        dir('backend') {
                            sh 'mvn -B -ntp clean $JACOCO:prepare-agent verify $JACOCO:report'
                        }
                    }
                }
                stage('Frontend (node:test)') {
                    steps {
                        echo '=== Bước 3b: node:test + coverage lcov cho JavaScript ==='
                        dir('frontend') {
                            sh 'npm run test:ci'
                        }
                    }
                }
            }
            post {
                always {
                    junit testResults: 'backend/target/surefire-reports/*.xml, frontend/reports/junit.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '=== Bước 4: Quét mã nguồn bằng SonarQube ==='
                withSonarQubeEnv('SonarQube') {
                    sh '"$SCANNER_HOME/bin/sonar-scanner" -Dsonar.projectVersion=$BUILD_NUMBER'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '=== Bước 5: Chờ SonarQube trả kết quả Quality Gate qua webhook ==='
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo '✅ Hạ tầng CI/CD hoạt động tốt: toolchain, unit test, SonarQube và Quality Gate đều OK.'
        }
        failure {
            echo '❌ Self-test thất bại – xem log ở stage bị đỏ (README.md → Xử lý sự cố).'
        }
    }
}
