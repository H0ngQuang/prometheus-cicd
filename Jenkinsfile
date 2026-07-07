pipeline {
    agent any
    
    environment {
        CHART_NAME = 'prometheus'
        CHART_PATH = '.'
        CHART_VERSION = '0.1.25'  // Tăng version lên
        HARBOR_URL = '192.168.92.128:8088'
        HARBOR_PROJECT = 'helm-charts'
        HARBOR_USERNAME = 'admin'
        HARBOR_PASSWORD = 'Harbor12345'
        K8S_NAMESPACE = 'monitoring'
    }
    
    stages {
        stage('Clone repository') {
            steps {
                // Dọn dẹp workspace trước khi clone
                cleanWs()
                checkout scm
                
                // Dọn dẹp file .tgz cũ (phòng hờ)
                sh 'rm -f templates/*.tgz *.tgz'
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
                    cd ${CHART_PATH}
                    helm package . --version ${CHART_VERSION} --destination /tmp
                    echo "✅ Helm chart đã đóng gói vào /tmp"
                '''
            }
        }
        
        stage('Push Chart lên Harbor') {
            steps {
                sh '''
                    helm registry login ${HARBOR_URL} -u ${HARBOR_USERNAME} -p ${HARBOR_PASSWORD} --plain-http
                    helm push /tmp/${CHART_NAME}-${CHART_VERSION}.tgz oci://${HARBOR_URL}/${HARBOR_PROJECT}
                    echo "✅ Chart đã được đẩy lên Harbor."
                '''
            }
        }
        
        stage('Deploy lên Kubernetes') {
            steps {
                sh '''
                    helm upgrade --install ${CHART_NAME} oci://${HARBOR_URL}/${HARBOR_PROJECT}/${CHART_NAME} \
                        --version ${CHART_VERSION} \
                        --namespace ${K8S_NAMESPACE} \
                        --plain-http \
                        --wait \
                        --timeout 5m \
                        --history-max 5
                    echo "✅ Deploy thành công!"
                '''
            }
        }
    }
    
    post {
        always {
            // Dọn dẹp file .tgz sau mỗi build
            sh '''
                rm -f /tmp/${CHART_NAME}-*.tgz
                echo "✅ Đã dọn dẹp file .tgz"
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