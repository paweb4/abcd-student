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
        stage('Prepare') {
            steps {
                sh 'mkdir -p results/'
            }
        }
        stage('DAST') {
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
                    docker run --name zap \
                        --add-host=host.docker.internal:host-gateway \
                        -v /home/kali/Desktop/abc_devsecops/abcd-student/.zap:/zap/wrk/:rw \
                        -t ghcr.io/zaproxy/zaproxy:stable bash -c \
                        "zap.sh -cmd -addonupdate; \
                        zap.sh -cmd -addoninstall communityScripts && \
                        zap.sh -cmd -addoninstall pscanrulesAlpha && \
                        zap.sh -cmd -addoninstall pscanrulesBeta && \
                        zap.sh -cmd -autorun /zap/wrk/passive.yaml"
                '''
            }
            post {
                always {
                    sh '''
                        docker cp zap:/zap/wrk/reports/zap_html_report.html "${WORKSPACE}/results/zap_html_report.html"
                        docker cp zap:/zap/wrk/reports/zap_xml_report.xml "${WORKSPACE}/results/zap_xml_report.xml"
                        docker stop zap || true
                        docker rm zap || true
                    '''
                    echo 'Archiving results...'
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                    echo 'Sending DAST reports to DefectDojo' 
                    defectDojoPublisher(artifact: "results/zap_xml_report.xml", 
                                        productName: 'Juice Shop', 
                                        scanType: 'ZAP Scan', 
                                        engagementName: 'paweb4@gmail.com')
                }
            }
        }
        stage('SCA SCAN') {
            steps {
                sh '''
                    if ! command -v go; then
                        echo "Go not found. Installing Go..."
                        curl -OL https://golang.org/dl/go1.20.5.linux-amd64.tar.gz
                        sudo tar -C /usr/local -xzf go1.20.5.linux-amd64.tar.gz
                        export PATH=$PATH:/usr/local/go/bin
                        echo "Go installed successfully."
                    fi
                    
                    if ! command -v osv-scanner; then
                        echo "OSV Scanner not found. Installing via Go..."
                        go install github.com/google/osv-scanner/cmd/osv-scanner@v1
                        export PATH=$PATH:$HOME/go/bin
                        echo "OSV Scanner installed successfully."
                    fi
                    
                    osv-scanner scan --lockfile package-lock.json --format json --output results/sca-osv-scanner.json || true
                '''
            }
            post {
                always {
                    echo 'Archiving results...'
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                    echo 'Sending OSV reports to DefectDojo'
                    defectDojoPublisher(artifact: "results/sca-osv-scanner.json",
                                        productName: 'Juice Shop',
                                        scanType: 'OSV Scan',
                                        engagementName: 'paweb4@gmail.com')
                }
            }
        }
        stage('SECRETS') {
            steps {
                sh '''
                    if ! command -v trufflehog; then
                        echo "TruffleHog not found. Installing from GitHub..."
                        git clone https://github.com/trufflesecurity/trufflehog.git
                        cd trufflehog
                        go install
                        export PATH=$PATH:$HOME/go/bin
                        echo "TruffleHog installed successfully."
                    fi
                    
                    trufflehog git file://. --branch main --json > results/trufflehog-secrets.json || true
                '''
            }
            post {
                always {
                     sh '''
                        docker stop juice-shop || true
                    '''
                    echo 'Archiving results...'
                    archiveArtifacts artifacts: 'results/**/*', fingerprint: true, allowEmptyArchive: true
                    echo 'Sending Trufflehog reports to DefectDojo'
                    defectDojoPublisher(artifact: "results/trufflehog-secrets.json",
                                        productName: 'Juice Shop',
                                        scanType: 'Trufflehog Scan',
                                        engagementName: 'paweb4@gmail.com')
                }
            }
        }
        stage('SAST') {
            steps {
                sh '''
                    if ! command -v semgrep; then
                        echo "Semgrep not found. Installing Semgrep..."
                        curl -L https://github.com/returntocorp/semgrep/releases/latest/download/semgrep-linux-amd64 -o /usr/local/bin/semgrep
                        chmod +x /usr/local/bin/semgrep
                        echo "Semgrep installed successfully."
                    fi

                    semgrep --config auto --json --output results/sast.json
                '''
            }
            post {
                always {
                    echo 'Archiving SAST results...'
                    archiveArtifacts artifacts: 'results/sast.json', fingerprint: true, allowEmptyArchive: true
                    echo 'Sending Semgrep report to DefectDojo'
                    defectDojoPublisher(artifact: "results/sast.json",
                                        productName: 'Juice Shop',
                                        scanType: 'Semgrep JSON Report',
                                        engagementName: 'paweb4@gmail.com')
                }
            }
        }
    }
}
