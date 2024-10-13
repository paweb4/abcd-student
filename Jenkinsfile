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
                    git credentialsId: 'github-pat', url: 'https://github.com/paweb4/abcd-student', branch: 'main'
                }
            }
        }
        stage('ZAP SCAN') {
            steps {
                sh '''
                    docker run --name juice-shop -d \
                        -p 3000:3000 \
                        bkimminich/juice-shop
                    sleep 5
                '''
                sh '''
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/kali/Desktop/abc_devsecops/abcd-student/.zap:/zap/wrk/:rw
                        -v /home/kali/Desktop/abc_devsecops/reports:/zap/wrk/reports \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c "
                        zap.sh -cmd -addonupdate
                        zap.sh -cmd -addoninstall communityScripts
                        zap.sh -cmd -addoninstall pscanrulesAlpha
                        zap.sh -cmd -addoninstall pscanrulesBeta
                        zap.sh -cmd -autorun /zap/wrk/passive_scan.yaml"
                '''
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                        docker stop zap juice-shop
                    '''
                    defectDojoPublisher(artifact: '${WORKSPACE}/results/zap_xml_report.xml', 
                    productName: 'Juice Shop', 
                    scanType: 'ZAP Scan', 
                    engagementName: 'paweb4@gmail.com')
                }
            }
        }
    }
}
