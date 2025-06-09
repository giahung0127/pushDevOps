// Define the function outside the pipeline block
def generateCleanupLink() {
    def jobUrl = "${env.JENKINS_URL}job/developer_cleanup"
    def htmlLink = """
        <h2>Deployment Cleanup</h2>
        <p>To clean up this deployment, click here: <a href='${jobUrl}/build?delay=0sec'>Run Cleanup Job</a></p>
    """
    return htmlLink
}

pipeline {
    agent any
    environment {
        // Độ phủ tối thiểu
        MINIMUM_COVERAGE = "70.0"
        // Tên được thiết lập dựa trên thư mục nào có thay đổi
        SERVICE_NAME = ""
        // Add Jenkins URL environment variable
        JENKINS_URL = "${env.JENKINS_URL}"
    }
    tools {
        maven 'Maven 3.9.8'
    }
    parameters {
        string(name: 'customers-service', defaultValue: 'main', description: 'Branch for customers-service')
        string(name: 'visits-service', defaultValue: 'main', description: 'Branch for visits-service')
        string(name: 'vets-service', defaultValue: 'main', description: 'Branch for vets-service')
        string(name: 'genai-service', defaultValue: 'main', description: 'Branch for genai-service')
        choice(name: 'NAMESPACE', choices: ['dev', 'staging'], description: 'Namespace to deploy')
    }
    stages {
        stage('Check User') {
            steps {
                sh 'whoami'
            }
        }
        // Xác định service nào đã thay đổi
        stage('Determine Changed Service') {
            steps {
                script {
                    // Lấy danh sách các file đã thay đổi trong commit
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1 HEAD || git diff --name-only origin/main HEAD', returnStdout: true).trim()
                    
                    // Định nghĩa các thư mục service cần kiểm tra
                    def services = [
                        'spring-petclinic-customers-service',
                        'spring-petclinic-visits-service',
                        'spring-petclinic-vets-service',
                        'spring-petclinic-genai-service'
                    ]
                    
                    // Kiểm tra từng thư mục service xem có thay đổi không
                    def changedServices = []
                    for (service in services) {
                        if (changedFiles.contains(service + '/')) {
                            changedServices.add(service)
                        }
                    }
                    
                    // Nếu không có service nào thay đổi, build tất cả
                    if (changedServices.size() == 0) {
                        changedServices = services
                    }
                    
                    // Kiểm tra file pom.xml hoặc các file khác có bị thay đổi không
                    if (changedFiles.contains('pom.xml') || changedFiles.contains('Jenkinsfile') || changedFiles.contains('docker-compose.yml')) {
                        echo "Root configuration files changed, building all services"
                        env.BUILD_ALL = 'true'
                        env.SERVICE_NAME = 'spring-petclinic-microservices'
                    } else if (changedServices.size() == 1) { // Nếu chỉ có 1 service bị thay đổi, chỉ build service đó
                        env.SERVICE_NAME = changedServices[0]
                        env.BUILD_ALL = 'false'
                        echo "Building only ${env.SERVICE_NAME}"
                    } else { // Nếu nhiều service bị thay đổi, build tất cả
                        echo "Multiple services changed: ${changedServices.join(', ')}"
                        env.BUILD_ALL = 'true'
                        env.SERVICE_NAME = 'spring-petclinic-microservices'
                    }

                    env.AFFECTED_SERVICES = changedServices.join(' ')

                    echo "changedFiles: ${changedFiles}"
                    echo "changedServices: ${changedServices}"
                    echo "AFFECTED_SERVICES: ${env.AFFECTED_SERVICES}"
                }
            }
        }

        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                    }
                }
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    def testCmd = ""

                    // Xác định lệnh test dựa trên việc build 1 hay tất cả service
                    if (env.BUILD_ALL == 'true') {
                        echo "Building and testing all services"
                        testCmd = "mvn clean test"
                    } else {
                        echo "Building and testing only ${env.SERVICE_NAME}"
                        // Sử dụng -pl để chỉ build 1 service cụ thể
                        testCmd = "mvn clean test -pl :${env.SERVICE_NAME}"
                    }

                    // Chạy test và lưu trạng thái
                    def testStatus = sh(script: testCmd, returnStatus: true)
                    // Xuất báo cáo test JUnit
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    
                    if (testStatus != 0) {
                        echo "Tests failed, continuing pipeline"
                    }
                    
                    try {
                        // Khi chỉ build 1 service, kiểm tra xem thư mục và xác định vị trí tệp báo cáo độ phủ
                        def jacocoPath = env.BUILD_ALL == 'true' ? 
                            'target/site/jacoco/jacoco.xml' : 
                            "${env.SERVICE_NAME}/target/site/jacoco/jacoco.xml"

                        // Kiểm tra xem file Jacoco có tồn tại không
                        def jacocoExists = sh(script: "[ -f ${jacocoPath} ] && echo true || echo false", returnStdout: true).trim()
                        
                        if (jacocoExists == "true") {
                            // Ghi lại báo cáo độ phủ Jacoco
                            recordCoverage(
                                tools: [[parser: 'JACOCO', pattern: "${jacocoPath}"]],
                                id: "${env.SERVICE_NAME}-coverage", 
                                name: "${env.SERVICE_NAME} Coverage",
                                qualityGates: [[threshold: env.MINIMUM_COVERAGE.toDouble(), metric: 'LINE', unstable: true]]
                            )

                            // Xác định thư mục báo cáo dựa trên chế độ build
                            def reportDir = env.BUILD_ALL == 'true' ? 
                                'target/site/jacoco' : 
                                "${env.SERVICE_NAME}/target/site/jacoco"

                            // Xuất báo cáo HTML cho độ phủ
                            publishHTML([
                                allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
                                reportDir: reportDir, reportFiles: 'index.html',
                                reportName: "${env.SERVICE_NAME} Coverage Report"
                            ])

                            // Xác định đường dẫn file CSV cho việc kiểm tra độ phủ
                            def csvPath = env.BUILD_ALL == 'true' ? 
                                'target/site/jacoco/jacoco.csv' : 
                                "${env.SERVICE_NAME}/target/site/jacoco/jacoco.csv"

                            // Script tính toán độ phủ và so sánh với ngưỡng
                            def coverageScript = '''
                                if [ -f ''' + csvPath + ''' ]; then
                                    COVERAGE=$(awk -F"," '{ instructions += $4 + $5; covered += $5 } END { if (instructions > 0) print (covered/instructions) * 100; else print 0 }' ''' + csvPath + ''')
                                    if (( $(awk -v cov="$COVERAGE" -v min="''' + env.MINIMUM_COVERAGE + '''" 'BEGIN {print (cov < min) ? 1 : 0}') )); then
                                        exit 1
                                    fi
                                fi
                            '''

                            // Nếu độ phủ dưới ngưỡng, đánh dấu build là không ổn định
                            if (sh(script: coverageScript, returnStatus: true) != 0) {
                                unstable "Coverage below threshold: ${env.MINIMUM_COVERAGE}%"
                            }
                        } else {
                            echo "JaCoCo XML not found at ${jacocoPath}. Add JaCoCo plugin to pom.xml."
                        }
                    } catch (Exception e) {
                        echo "Coverage reporting failed: ${e.message}"
                        unstable "Coverage reporting failed"
                    }
                }
            }
        }
        
        stage('Package') {
            steps {
                script {
                    // Đóng gói dựa trên việc build 1 hay tất cả service
                    if (env.BUILD_ALL == 'true') {
                        sh "mvn package -DskipTests"
                        // Lưu trữ các artifacts được tạo ra
                        archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true, allowEmptyArchive: true
                    } else {
                        // Chỉ đóng gói service cụ thể
                        sh "mvn package -pl :${env.SERVICE_NAME} -DskipTests"
                        // Lưu trữ artifact từ service cụ thể
                        archiveArtifacts artifacts: "${env.SERVICE_NAME}/target/*.jar", fingerprint: true, allowEmptyArchive: true
                    }
                }
            }
        }

        stage('Get Commit IDs') {
            steps {
                script {
                    // Fetch latest for all branches
                    sh "git fetch --all"
                    env.CUSTOMERS_COMMIT = sh(script: "git rev-parse origin/${params['customers-service']}", returnStdout: true).trim()
                    env.VISITS_COMMIT = sh(script: "git rev-parse origin/${params['visits-service']}", returnStdout: true).trim()
                    env.VETS_COMMIT = sh(script: "git rev-parse origin/${params['vets-service']}", returnStdout: true).trim()
                    env.GENAI_COMMIT = sh(script: "git rev-parse origin/${params['genai-service']}", returnStdout: true).trim()
                }
            }
        }

        stage('Debug Dockerfile') {
            steps {
                script {
                    sh 'find . -name Dockerfile'
                }
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    def CONTAINER_TAG = env.TAG_NAME ? env.TAG_NAME : env.GIT_COMMIT.take(7)
                    echo "Using tag: ${CONTAINER_TAG}"
                    echo "Building images for services: ${env.AFFECTED_SERVICES}"
                    def services = env.AFFECTED_SERVICES.split(' ').findAll { it?.trim() }
                    if (!env.DOCKER_REGISTRY) {
                        error "DOCKER_REGISTRY is not set!"
                    }
                    for (service in services) {
                        if (!fileExists("${service}/pom.xml")) {
                            echo "Skipping ${service} because pom.xml not found"
                            continue
                        }
                        echo "Building and pushing Docker image for ${service}"
                        sh """
                            cd ${service}
                            mvn clean install -P buildDocker -Dmaven.test.skip=true \
                                -Ddocker.image.prefix=${env.DOCKER_REGISTRY} \
                                -Ddocker.image.tag=${CONTAINER_TAG}
                            docker push ${env.DOCKER_REGISTRY}/${service}:${CONTAINER_TAG}
                            cd ..
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'minikube-kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    script {
                        // Bước 1: Kiểm tra thư mục helm-chart
                        if (!fileExists('./helm-chart')) {
                            error "Helm chart directory not found"
                        }

                        // Bước 2: Copy và set quyền cho kubeconfig
                        sh """
                            cp $KUBECONFIG_FILE ./kubeconfig
                            chmod 600 ./kubeconfig
                        """

                        // Bước 3: Tạo namespace (bỏ qua kiểm tra)
                        sh """
                            kubectl --kubeconfig=./kubeconfig create namespace ${params.NAMESPACE} --dry-run=client -o yaml | kubectl --kubeconfig=./kubeconfig apply -f -
                        """

                        // Bước 4: Deploy với Helm
                        sh """
                            helm --kubeconfig=./kubeconfig upgrade --install petclinic ./helm-chart \
                                --namespace ${params.NAMESPACE} \
                                --set global.dockerRegistry=${env.DOCKER_REGISTRY} \
                                --set customers.image.tag=${env.CUSTOMERS_COMMIT} \
                                --set visits.image.tag=${env.VISITS_COMMIT} \
                                --set vets.image.tag=${env.VETS_COMMIT} \
                                --set genai.image.tag=${env.GENAI_COMMIT}
                        """

                        // Bước 5: Kiểm tra pods
                        sh """
                            echo "Waiting for pods to be ready..."
                            sleep 10
                            kubectl --kubeconfig=./kubeconfig get pods -n ${params.NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Display Access Information') {
            steps {
                script {
                    // Lấy thông tin NodePort và IP của cluster
                    def nodePort = sh(script: "kubectl --kubeconfig=./kubeconfig get svc -n ${params.NAMESPACE} -o jsonpath='{.items[?(@.spec.type==\"NodePort\")].spec.ports[0].nodePort}'", returnStdout: true).trim()
                    def clusterIP = sh(script: "kubectl --kubeconfig=./kubeconfig config view --minify -o jsonpath='{.clusters[0].cluster.server}' | sed 's|https://||' | sed 's|:.*||'", returnStdout: true).trim()
                    
                    // Tạo thông báo hướng dẫn
                    def accessInfo = """
                    ============================================
                    THÔNG TIN TRUY CẬP ỨNG DỤNG
                    ============================================
                    
                    Để truy cập ứng dụng, bạn cần thêm entry sau vào file /etc/hosts:
                    ${clusterIP} petclinic.local
                    
                    Truy cập ứng dụng qua URL:
                    http://petclinic.local:${nodePort}
                    
                    Các service khác:
                    - API Gateway: http://petclinic.local:${nodePort}/api
                    - Customers Service: http://petclinic.local:${nodePort}/api/customers
                    - Visits Service: http://petclinic.local:${nodePort}/api/visits
                    - Vets Service: http://petclinic.local:${nodePort}/api/vets
                    
                    Kiểm tra trạng thái:
                    kubectl --kubeconfig=./kubeconfig get svc -n ${params.NAMESPACE}
                    kubectl --kubeconfig=./kubeconfig get pods -n ${params.NAMESPACE}
                    
                    Xem logs:
                    kubectl --kubeconfig=./kubeconfig logs -n ${params.NAMESPACE} -l app=petclinic
                    
                    ============================================
                    """
                    
                    // Hiển thị thông tin
                    echo accessInfo
                    
                    // Lưu thông tin vào file để dễ tham khảo
                    writeFile file: 'access-info.txt', text: accessInfo
                    archiveArtifacts artifacts: 'access-info.txt', fingerprint: true
                }
            }
        }

        stage('Display Cleanup Link') {
            steps {
                script {
                    def cleanupUrl = "${env.JENKINS_URL}job/developer_cleanup/build?delay=0sec"
                    echo """
                    ============================================
                    Deployment Cleanup Instructions:
                    To clean up this deployment, visit:
                    ${cleanupUrl}
                    ============================================
                    """
                }
            }
        }
    }
    
    post {
        success { 
            script {
                echo 'Pipeline successful'
                echo "To cleanup this deployment, visit: ${env.JENKINS_URL}job/developer_cleanup"
            }
        }
        unstable { echo 'Pipeline unstable' }
        failure { echo 'Pipeline failed' }
        always {
            // Clean up Docker
            sh """
                docker logout
                docker system prune -f
            """
        }
    }
}