pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    stages {
        stage('Code checkout from GitHub') {
            steps {
                script {
                    cleanWs()
                    git credentialsId: 'github-token', url: 'https://github.com/7348313/lukaszw', branch: 'main'
                }
            }
        }
        stage('zap scan') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \\
                        -p 3000:3000 \\
                        bkimminich/juice-shop
                        sleep 5
                '''
                sh '''
                    docker run --name zap --rm \
                    --add-host=host.docker.internal:host-gateway \
                    -v /Users/lukaszwojcik/BezpiecznyKod/lukaszw/.zap:/zap/wrk:rw \
                    -v /Users/lukaszwojcik/Downloads/Reports/:/zap/wrk/reports \
                    -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                    "zap.sh -cmd -addoninstall communityScripts && \
                    zap.sh -cmd -addoninstall pscanrulesAlpha && \
                    zap.sh -cmd -addoninstall pscanrulesBeta && \
                    zap.sh -cmd -autorun /zap/wrk/passive.yaml"
                '''
                sh 'ls -la'
            }
        }
    }
}
post {
    always {
        script{
            sh '''
                docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                docker stop juice-shop || true
            '''
            defectDojoPublisher
                artifact: 'results/zap_xml_report.xml'
                productName: 'Juice Shop'
                scanType 'ZAP Scan'
                engagementName: 'lukasik446@gmail.com'    
        }
    }
}
