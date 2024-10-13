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
                    docker stop juice-shop
                    docker rm juice-shop
                    docker run --name juice-shop -d -p 3000:3000 bkimminich/juice-shop
                    while [ "$(docker inspect -f '{{.State.Running}}' juice-shop)" != "true" ]; do
                        sleep 5
                    done
                    echo "App is ready"
                '''
                sh '''
                    docker stop zap || true
                    docker rm zap || true
                    docker run -u root --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/kali/Desktop/abc_devsecops/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "mkdir -p /zap/wrk/reports && \
                        zap.sh -cmd -addonupdate; \
                        zap.sh -cmd -addoninstall communityScripts && \
                        zap.sh -cmd -addoninstall pscanrulesAlpha && \
                        zap.sh -cmd -addoninstall pscanrulesBeta && \
                        zap.sh -cmd -autorun /zap/wrk/passive.yaml"
                '''
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html ${WORKSPACE}/results/zap_html_report.html
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml ${WORKSPACE}/results/zap_xml_report.xml
                        docker stop zap juice-shop
                    '''
                    // Publikacja raportu do DefectDojo
                    defectDojoPublisher(artifact: "results/zap_xml_report.xml", 
                                        productName: 'Juice Shop', 
                                        scanType: 'ZAP Scan', 
                                        engagementName: 'paweb4@gmail.com')

                    defectDojoPublisher(artifact: "results/zap_html_report.html", 
                                        productName: 'Juice Shop', 
                                        scanType: 'ZAP Scan', 
                                        engagementName: 'paweb4@gmail.com')
                }
            }
        }
    }
}
