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
        stage('Prepare') {
            steps {
                sh 'mkdir -p results/'
            }
        }
        // DAST
        stage('DAST') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \
                    -p 3000:3000 bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    docker run --name zap \
                        -v /Users/lukaszwojcik/BezpiecznyKod/lukaszw/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable \
                        bash -c "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive.yaml" || true
                '''
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                        docker stop zap
                        docker stop juice-shop
                        docker rm zap
                        docker rm juice-shop
                    '''
                }
            }
        }
        // SCA
        stage('SCA scan') {
            steps {
                sh 'osv-scanner scan --lockfile package-lock.json --format json --output results/sca-osv-scanner.json || true'
            }
        }
        // TruffleHog scanning for secrets
        stage('TruffleHog Scan') {
            steps {
                script {
                   // sh 'trufflehog --json . > results/trufflehog_results.json'
                    trufflehog git file://. --only-verified --json > results/trufflehog_results.json
                }
            }
        }
        // Run Semgrep Scan
        stage('Run Semgrep Scan') {
            steps {
                script {
                    // Run Semgrep scan
                    sh 'semgrep --config "auto" --output results/semgrep_results.json .'
                }
            }
        }
    }
    post {
        always {
            echo 'Archiving results...'
            archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
            echo 'Sending reports to DefectDojo...'
            defectDojoPublisher(artifact: 'results/zap_xml_report.xml', productName: 'Juice Shop', scanType: 'ZAP Scan', engagementName: 'lukasik446@gmail.com')
            defectDojoPublisher(artifact: 'results/sca-osv-scanner.json', productName: 'Juice Shop', scanType: 'OSV Scan', engagementName: 'lukasik446@gmail.com')
            defectDojoPublisher(artifact: 'results/trufflehog_results.json', productName: 'Juice Shop', scanType: 'TruffleHog Scan', engagementName: 'lukasik446@gmail.com')
            defectDojoPublisher(artifact: 'results/semgrep_results.json', productName: 'Juice Shop', scanType: 'TruffleHog Scan', engagementName: 'lukasik446@gmail.com')
        }
    }
}