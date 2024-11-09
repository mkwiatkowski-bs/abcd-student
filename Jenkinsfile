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
                    git credentialsId: 'GitHub-pat', url: 'https://github.com/mkwiatkowski-bs/abcd-student.git', branch: 'main'
                }
            }
        }
        /*stage('Example') {
            steps {
                echo 'Hello!'
                sh 'ls -la'
            }
        }*/
        stage('Create reports directory') {
            steps {
                sh 'mkdir -p results/'
            }
        }
        stage('ZAP passive scan') {
            steps {
                sh '''
                    docker run --name juice-shop -d --rm \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /Users/marcinkwiatkowski/Developer/code/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable \
                        bash -c "zap.sh -cmd -addonupdate; zap.sh -cmd -addoninstall communityScripts -addoninstall pscanrulesAlpha -addoninstall pscanrulesBeta -autorun /zap/wrk/passive_scan.yaml" \
                        || true
                '''
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html "${WORKSPACE}/results/zap_html_report.html"
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml "${WORKSPACE}/results/zap_xml_report.xml"
                        docker stop zap juice-shop
                        docker rm zap
                    '''
                }
            }
        }
        stage('OSV') {
            steps {
                sh '''
                    osv-scanner scan --lockfile package-lock.json --format json --output "${WORKSPACE}/results/osv_report.json" \
                        || true
                '''
            }
        }
        stage('TruffleHog') {
            steps {
                sh '''
                    trufflehog git file://. --branch=main --json --only-verified --fail >> "${WORKSPACE}/results/trufflehog_report.json" \
                        || true
                '''
            }
        }
        stage('Semgrep') {
            steps {
                sh '''
                    semgrep ci --config=auto --json > "${WORKSPACE}/results/semgrep_report.json" \
                        || true
                '''
            }
        }

    }
    post {
        always {
            echo 'Archiving reports...'
            archiveArtifacts artifacts: 'results/*.html', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'results/*.xml', fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: 'results/*.json', fingerprint: true, allowEmptyArchive: true
            echo 'Publishing reports to DefectDojo...'
            defectDojoPublisher(artifact: 'results/zap_xml_report.xml', 
                productName: 'Juice Shop', 
                scanType: 'ZAP Scan', 
                engagementName: 'm.kwiatkowski@benefitsystems.pl')
            defectDojoPublisher(artifact: 'results/osv_report.json', 
                productName: 'Juice Shop', 
                scanType: 'OSV Scan', 
                engagementName: 'm.kwiatkowski@benefitsystems.pl')
            defectDojoPublisher(artifact: 'results/trufflehog_report.json', 
                productName: 'Juice Shop', 
                scanType: 'Trufflehog Scan', 
                engagementName: 'm.kwiatkowski@benefitsystems.pl')
             defectDojoPublisher(artifact: 'results/semgrep_report.json', 
                productName: 'Juice Shop', 
                scanType: 'Semgrep JSON Report', 
                engagementName: 'm.kwiatkowski@benefitsystems.pl')
        }
    }
}
