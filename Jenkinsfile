pipeline{
    agent {
        label 'helm-slave' 
    }

    environment {
        // Harbor Configuration
        HARBOR_URL = '192.168.92.128:8088'
        HARBOR_PROJECT = 'helm-charts'

        // Prometheus    Configuration
        CHART_NAME = 'prometheus'
        CHART_PATH = '.' // Root folder chua Chart.yaml

        // Kubernetes
        K8S_NAMESPACE = 'monitoring'
        // Credentials IDs
        HARBOR_CRED_ID = 'harbor-credentials'

    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'staging', 'prod'],
            description: 'Chọn môi trường deploy'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: false,
            description: 'Chạy dry-run (không deploy thật)'
        )
    }

    options {
        // Tự động xóa builds cũ
        buildDiscarder(logRotator(
            numToKeepStr: '10',           // Giữ 10 builds gần nhất
            daysToKeepStr: '3',           // Giữ 3 ngày
            artifactNumToKeepStr: '5',    // Giữ 5 artifacts
            artifactDaysToKeepStr: '7'
        ))
        
        // Timeout pipeline sau 30 phút
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('1. Clone repository') {
            steps {
                checkout scm
            }
        }
        stage('2. Lint & Validate Helm Chart') {
            steps {
                sh '''
                    # Kiểm tra syntax Helm Chart
                    helm lint ${CHART_PATH}

                    # Render chart thành YAML để kiểm tra
                    helm template ${CHART_NAME} ${CHART_PATH} \
                        --namespace ${K8S_NAMESPACE} \
                        > /tmp/helm-template.yaml

                    echo "✅ Helm chart hợp lệ."
                '''
            }
        }
        
        stage('3. Package Helm Chart') {
            steps {
                sh '''
                    mkdir -p ./chart-packages

                    # Package chart với version từ BUILD_NUMBER
                    helm package ${CHART_PATH} \
                        --version 0.1.${BUILD_NUMBER} \
                        --destination ./chart-packages

                    echo "✅ Đã đóng gói chart:"
                    ls -lh ./chart-packages/
                    '''
            }
        }
        
        stage('4. Push Chart len Harbor'){
            steps {
                sh '''
                    helm registry login ${HARBOR_URL} \
                        -u ${HARBOR_CRED_ID} \
                        --password-stdin <<< $(kubectl get secret ${HARBOR_CRED_ID} -n dev -o jsonpath='{.data.password}' | base64 -d) \
                        --plain-http
                    
                    CHART_FILE=$(ls -t ./chart-packages/*.tgz | head -1)
                    echo "Push file: ${CHART_FILE}"
                    
                    helm push $(CHART_FILE) \
                        oci://${HARBOR_URL}/${HARBOR_PROJECT} \
                        --plain-http
                    
                    echo "✅ Da push Helm chart len Harbor!"
                        
                '''
            }
        }
        stage('5. Deploy lên Kubernetes') {
            steps {
                echo "🚀 Đang deploy Prometheus..."
                sh '''
                    helm upgrade --install prometheus \
                        oci://${HARBOR_URL}/${HARBOR_PROJECT}/prometheus \
                        --version 0.1.${BUILD_NUMBER} \
                        --namespace ${K8S_NAMESPACE} \
                        --wait --timeout 5m
                '''
            }
        }
    }
}