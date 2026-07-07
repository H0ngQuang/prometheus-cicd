pipeline{
    agent any

    environment {
        HARBOR_URL = '192.168.92.128:8088'
        HARBOR_PROJECT = 'helm-charts'
        CHART_NAME = 'prometheus'
        CHART_PATH = '.'
        K8S_NAMESPACE = 'monitoring'
        
        // Thêm 2 biến này để login Harbor trực tiếp
        HARBOR_USER = 'admin'
        HARBOR_PASS = 'Harbor12345'
    }

    // ✅ ĐÃ BỎ: Block parameters (không còn DRY_RUN và DEPLOY_ENV)

    options {
        buildDiscarder(logRotator(
            numToKeepStr: '10',
            daysToKeepStr: '3',
            artifactNumToKeepStr: '5',
            artifactDaysToKeepStr: '7'
        ))
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
                    helm lint ${CHART_PATH}
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
                    helm package ${CHART_PATH} \
                        --version 0.1.${BUILD_NUMBER} \
                        --destination ./chart-packages
                    echo "✅ Đã đóng gói chart:"
                    ls -lh ./chart-packages/
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
}