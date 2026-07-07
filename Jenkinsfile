pipeline {
    agent any
    
    environment {
        CHART_NAME = 'prometheus'
        CHART_PATH = '.'
        CHART_VERSION = '0.1.25'
        HARBOR_URL = '192.168.92.128:8088'
        HARBOR_PROJECT = 'helm-charts'
        HARBOR_USERNAME = 'admin'
        HARBOR_PASSWORD = 'Harbor12345'
        K8S_NAMESPACE = 'monitoring'
    }
    
    stages {
        stage('Clone repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Clean up old .tgz files') {
            steps {
                sh '''
                    # Xóa SẠCH file .tgz TRƯỚC khi lint
                    find . -name "*.tgz" -type f -delete
                    echo "✅ Đã dọn dẹp file .tgz cũ"
                '''
            }
        }
        
        stage('Lint & Validate Helm Chart') {
            steps {
                sh '''
                    helm lint ${CHART_PATH}
                    helm template ${CHART_NAME} ${CHART_PATH} --namespace ${K8S_NAMESPACE} > /tmp/helm-template.yaml
                    echo "✅ Helm chart hợp lệ."
                '''
            }
        }
        
        stage('Package Helm Chart') {
            steps {
                sh '''
                    helm package ${CHART_PATH} --version ${CHART_VERSION} --destination /tmp
                    echo "✅ Helm chart đã đóng gói vào /tmp"
                '''
            }
        }
        
        stage('4. Push Chart lên Harbor') {
            steps {
                sh '''
                    # ✅ ĐÃ SỬA: Login Harbor bằng biến trực tiếp
                    helm registry login ${HARBOR_URL} \
                        -u ${HARBOR_USER} \
                        -p ${HARBOR_PASS} \
                        --plain-http
                    
                    CHART_FILE=$(ls -t ./chart-packages/*.tgz | head -1)
                    echo "📦 Push file: ${CHART_FILE}"
                    
                    # ✅ ĐÃ SỬA: ${CHART_FILE} thay vì $(CHART_FILE)
                    helm push "${CHART_FILE}" \
                        oci://${HARBOR_URL}/${HARBOR_PROJECT} \
                        --plain-http
                    
                    echo "✅ Đã push Helm chart lên Harbor!"
                '''
            }
        }
        
        stage('5. Deploy lên Kubernetes') {
            steps {
                echo "🚀 Đang deploy Prometheus..."
                sh '''
                    # ✅ ĐÃ THÊM: Login Harbor để pull chart
                    helm registry login ${HARBOR_URL} \
                        -u ${HARBOR_USER} \
                        -p ${HARBOR_PASS} \
                        --plain-http
                    
                    # ✅ ĐÃ THÊM: --plain-http để pull từ Harbor HTTP
                    helm upgrade --install prometheus \
                        oci://${HARBOR_URL}/${HARBOR_PROJECT}/prometheus \
                        --version 0.1.${BUILD_NUMBER} \
                        --namespace ${K8S_NAMESPACE} \
                        --plain-http \
                        --wait --timeout 5m
                        
                    echo "✅ Deploy thành công!"
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                rm -f /tmp/${CHART_NAME}-*.tgz
                echo "✅ Đã dọn dẹp file .tgz trong /tmp"
            '''
        }
        success {
            echo '✅ Pipeline thành công!'
        }
        failure {
            echo '❌ Pipeline thất bại!'
        }
    }
}