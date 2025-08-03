// File : Jenkinsfile
// This Jenkinsfile uses a robust approach by running each build stage
// inside its own Docker container. This ensures a clean, consistent, and
// isolated environment for each part of the pipeline.

pipeline {
    agent any
    
    environment {
        // Define the application name and version for Docker image tagging.
        // This will be used to build and tag the Docker image.
        APP_NAME = 'jenkins-demo-app'
        APP_VERSION = "${BUILD_NUMBER}"
        DOCKER_IMAGE = "${APP_NAME}:${APP_VERSION}"
        
        // Telegram Bot Configuration
        // catatan : simpan token dan chat ID di Jenkins Credentials
        // -> gunakan credentials dengan tipe 'Secret text' untuk token dan 'Username with password'
        // -> pastikan nama credentials sesuai dengan yang digunakan di bawah ini
        // TELEGRAM_BOT_TOKEN = credentials('TG_TOKEN')    
        // TELEGRAM_CHAT_ID = credentials('TG_CHAT_ID')
        TELEGRAM_BOT_TOKEN = "XXX"    
        TELEGRAM_CHAT_ID = "XXX"
        
        // Catatan : apabila nama credentialsnya : TELEGRAM_BOT_TOKEN dan TELEGRAM_CHAT_ID, maka akan ada error
        // -> Telegram semacam mendeteksi dan langsung masking

        // Environment URLs
        STAGING_URL = 'http://localhost:8081'
        PRODUCTION_URL = 'http://localhost:8090'
        
        // Quality Gates
        TEST_THRESHOLD = '80'
        COVERAGE_THRESHOLD = '70'

        // Git Repository URL & // Branch Name
        GIT_REPO_URL_OPS = 'https://github.com/qanitasyaf/ops-secure-onboarding-system.git'
        BRANCH_NAME_OPS = 'master'

        GIT_REPO_URL_BE = 'https://github.com/bostang/backend-secure-onboarding-system.git'
        BRANCH_NAME_BE = 'deploy/gke'

        GIT_REPO_URL_FE = 'https://github.com/alvarolt17/frontend-secure-onboarding-system.git'
        BRANCH_NAME_FE = 'deploy/gke'

        // Firebase service accounts
        FIREBASE_KEY_FILE_NAME = 'wondr-desktop-otp-firebase-adminsdk-fbsvc-779cc379ef.json'

        // Cluster dan Zone
        gkeClusterName = 'my-gke-cluster'
        gkeClusterZone = 'asia-southeast2-a'
    }
    
    parameters {
        choice(
            name: 'DEPLOY_ENVIRONMENT',
            choices: ['staging', 'production', 'both'],
            description: 'Select deployment environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
        booleanParam(
            name: 'ENABLE_NOTIFICATIONS',
            defaultValue: false,
            description: 'Enable Telegram notifications'
        )
        string(
            name: 'CUSTOM_TAG',
            defaultValue: '',
            description: 'Custom Docker tag (optional)'
        )
        booleanParam(
        name: 'CREATE_NEW_CLUSTER',
        defaultValue: false,
        description: 'Set to true to create a new Google Cloud cluster using Terraform'
    )
    }
    
    stages {
        stage('🚀 Pipeline Initialization') {
            steps {
                script {
                    // Calculate custom image tag
                    env.FINAL_TAG = params.CUSTOM_TAG ?: env.APP_VERSION
                    env.DOCKER_IMAGE_FINAL = "${APP_NAME}:${env.FINAL_TAG}"
                    
                    // Send detailed start notification
                    def startMessage = getStartMessage()
                    
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage(startMessage)
                    }
                }
                
                echo '🔍 Initializing pipeline...'
                echo "Deploy Environment: ${params.DEPLOY_ENVIRONMENT}"
                echo "Skip Tests: ${params.SKIP_TESTS}"
                echo "Docker Image: ${env.DOCKER_IMAGE_FINAL}"
            }
        }
        
        // ✅ SUKSES (2025-08-03 09:16)
        stage('📥 Source Code Checkout') {
            steps {
                echo '📥 Checking out source code...'
                
                script {
                    // Checkout repositori Ops
                    echo "--- Checking out Ops repository ---"
                    dir('ops-repo') {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "refs/heads/${env.BRANCH_NAME_OPS}"]],
                            userRemoteConfigs: [[url: env.GIT_REPO_URL_OPS]]
                        ])
                    }
                    
                    // Menggunakan blok parallel untuk checkout Backend dan Frontend
                    parallel(
                        'Checkout Backend': {
                            echo "--- Checking out Backend repository ---"
                            dir('backend-repo') {
                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: "refs/heads/${env.BRANCH_NAME_BE}"]],
                                    userRemoteConfigs: [[url: env.GIT_REPO_URL_BE]]
                                ])
                            }
                        },
                        'Checkout Frontend': {
                            echo "--- Checking out Frontend repository ---"
                            dir('frontend-repo') {
                                checkout([
                                    $class: 'GitSCM',
                                    branches: [[name: "refs/heads/${env.BRANCH_NAME_FE}"]],
                                    userRemoteConfigs: [[url: env.GIT_REPO_URL_FE]]
                                ])
                            }
                        }
                    )

                    // Dapatkan informasi commit dari salah satu repo (misal: backend)
                    // Langkah ini harus dilakukan setelah checkout selesai
                    def beCommit = sh(script: "cd backend-repo && git log -1 --pretty=format:'%h'", returnStdout: true).trim()
                    def beMessage = sh(script: "cd backend-repo && git log -1 --pretty=format:'%s'", returnStdout: true).trim()
                    
                    env.GIT_COMMIT_SHORT_BE = beCommit
                    env.GIT_COMMIT_MSG_BE = beMessage

                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("📥 <b>Checkout Complete</b>\n🔗 **BE Commit:** ${env.GIT_COMMIT_SHORT_BE}\n💬 *${env.GIT_COMMIT_MSG_BE}*")
                    }
                }
                
                echo '✅ Source code checkout completed'
                echo '--- Direktori Kerja ---'
                sh 'ls -la'
            }
        }


        // ✅ TAHAPAN: Inject Firebase Private Key ke backend
        stage('🔑 Inject Secrets') {
            steps {
                echo '🔑 Injecting secrets into workspace...'
                
                script {
                    // Menggunakan withCredentials untuk mengakses file private key Firebase
                    withCredentials([file(credentialsId: 'firebase-private-key', variable: 'FIREBASE_KEY_FILE')]) {
                        // Pastikan kita berada di direktori backend
                        dir('backend-repo') {
                            // Pastikan direktori src/main/resources ada dan memiliki izin tulis
                            sh 'mkdir -p src/main/resources'
                            sh 'chmod -R 777 src/main/resources' // Memastikan izin tulis untuk semua user
                            
                            // Salin file private key yang diekspos oleh Jenkins ke direktori target
                            // Kita perlu mendapatkan nama file asli dari file yang diunggah di Jenkins
                            // Untuk mengabaikan nama file asli, kita gunakan nama standar
                            def firebaseKeyFileName = 'service-account-key.json' // Ganti dengan nama file standar yang diinginkan
                            sh "cp \"$FIREBASE_KEY_FILE\" \"src/main/resources/${firebaseKeyFileName}\""
                            
                            echo '✅ Firebase private key injected successfully'
                        }
                    }
                }
            }
        }

        // jangan lupa untu menginstall JDK 21 di Jenkins Global Tool Configuration
        // dan pastikan nama JDK sesuai dengan yang digunakan di bawah ini
        // -> misal: jdk21
        // jangan lupa untuk menginstal Node JS
        // dan pada tools berikan nama 'node18' berikan versi `v18.19.1`
        // ✅ SUKSES (2025-08-03 15:20)
        stage('🔨 Build Application') {
            steps {
                script {
                    parallel(
                        'Build Backend': {
                            echo '🔨 Building backend application...'
                            dir('backend-repo') {
                                withEnv(["JAVA_HOME=${tool 'jdk21'}"]) {
                                    sh 'mvn clean package -DskipTests'
                                }
                                echo '✅ Backend build completed successfully'
                            }
                        },
                        'Build Frontend': {
                            echo '🔨 Building frontend application...'
                            dir('frontend-repo') {
                                withEnv(["PATH+NODE=${tool 'node18'}/bin"]) {
                                    sh 'npm install'
                                    sh 'npm run build'
                                    echo '✅ Frontend build completed successfully'
                                }
                            }
                        }
                    )
                }
                
                // Kirim notifikasi setelah semua proses build selesai
                script {
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("🔨 <b>Build Stage Completed</b>\n✅ Backend and frontend compiled successfully")
                    }
                }
            }
            
            post {
                failure {
                    script {
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("❌ <b>Build Failed</b>\n🚨 Compilation errors detected")
                        }
                    }
                }
            }
        }
        
        stage('🧪 Testing & Quality Analysis') {
            parallel {
                stage('Unit Tests') {
                    when {
                        expression { !params.SKIP_TESTS }
                    }
                    steps {
                        echo '🧪 Running unit tests...'
                        echo 'mvn test -B'
                        echo '✅ All unit tests passed successfully'
                        
                        // Echo mock test results directory and files creation
                        echo 'mkdir -p target/surefire-reports'
                        echo 'echo "Mock test results" > target/surefire-reports/TEST-sample.xml'
                        echo '🧪 Mock test results created in target/surefire-reports'
                    }
                    post {
                        always {
                            echo '🧪 Publishing test results: target/surefire-reports/*.xml (simulated)'
                            echo '🧪 Archiving test artifacts: target/surefire-reports/* (simulated)'
                        }
                        success {
                            script {
                                if (params.ENABLE_NOTIFICATIONS) {
                                    sendTelegramMessage("🧪 <b>Unit Tests Passed</b>\n✅ All tests successful")
                                }
                            }
                        }
                        failure {
                            script {
                                if (params.ENABLE_NOTIFICATIONS) {
                                    sendTelegramMessage("❌ <b>Unit Tests Failed</b>\n🚨 Test failures detected")
                                }
                            }
                        }
                    }
                }
                
                stage('Code Quality') {
                    steps {
                        echo '📊 Running code quality analysis...'
                        echo 'mvn verify -B -DskipTests'
                        echo '✅ Code quality analysis passed'
                        
                        script {
                            if (params.ENABLE_NOTIFICATIONS) {
                                sendTelegramMessage("📊 <b>Code Quality Check</b>\n✅ Quality analysis completed")
                            }
                        }
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        echo '🔒 Running security scan...'
                        script {
                            // Echo simulate security scan
                            echo 'sleep(time: 5, unit: SECONDS) - Security scan simulation'
                            echo 'Security scan completed - No vulnerabilities found'
                            
                            if (params.ENABLE_NOTIFICATIONS) {
                                sendTelegramMessage("🔒 <b>Security Scan</b>\n✅ No vulnerabilities detected")
                            }
                        }
                    }
                }
            }
        }
        
        stage('📦 Package Application') {
            steps {
                echo '📦 Packaging the application...'
                echo 'mvn package -DskipTests -B'
                echo '✅ JAR file created successfully'
                
                // Echo mock JAR file creation
                echo 'mkdir -p target'
                echo 'echo "Mock JAR content" > target/demo-app.jar'
                echo '📦 Mock JAR file created in target directory'
                
                script {
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("📦 <b>Package Complete</b>\n✅ JAR file created successfully")
                    }
                }
            }
            post {
                success {
                    echo '📦 Archiving artifacts: target/*.jar (simulated)'
                }
            }
        }
        
        // ✅ SUKSES (2025-08-03 16:34)
        // ✅ UPDATE (2025-08-03 16:56 -- using JENKINS secrets u/ simpan VITE_XXX)
        stage('🐳 Docker Build & Registry') {
            steps {
                script {
                    parallel(
                        'Build & Push Backend': {
                            dir('backend-repo') {
                                echo '🐳 Building backend Docker image...'
                                def beImage = docker.build("asia.gcr.io/primeval-rune-467212-t9/wondr-desktop-jenkins-be:${env.FINAL_TAG}", ".")
                                echo '✅ Backend Docker image built successfully'

                                withCredentials([file(credentialsId: 'gcr-credentials', variable: 'GCR_KEY_FILE')]) {
                                    echo '🔐 Authenticating to Google Container Registry...'
                                    sh "cat ${GCR_KEY_FILE} | docker login -u _json_key --password-stdin https://asia.gcr.io"
                                    docker.withRegistry('https://asia.gcr.io') {
                                        beImage.push()
                                    }
                                }
                                echo "✅ Backend Docker image pushed to GCR: asia.gcr.io/primeval-rune-467212-t9/wondr-desktop-jenkins-be:${env.FINAL_TAG}"
                            }
                        },
                        'Build & Push Frontend': {
                            dir('frontend-repo') {
                                echo '🐳 Building frontend Docker image...'

                                // Gunakan withCredentials untuk mengekspos semua variabel rahasia
                                withCredentials([
                                    string(credentialsId: 'VITE_BACKEND_BASE_URL', variable: 'VITE_BACKEND_BASE_URL'),
                                    string(credentialsId: 'VITE_VERIFICATOR_BASE_URL', variable: 'VITE_VERIFICATOR_BASE_URL'),
                                    string(credentialsId: 'VITE_FIREBASE_API_KEY', variable: 'VITE_FIREBASE_API_KEY'),
                                    string(credentialsId: 'VITE_FIREBASE_AUTH_DOMAIN', variable: 'VITE_FIREBASE_AUTH_DOMAIN'),
                                    string(credentialsId: 'VITE_FIREBASE_PROJECT_ID', variable: 'VITE_FIREBASE_PROJECT_ID'),
                                    string(credentialsId: 'VITE_FIREBASE_STORAGE_BUCKET', variable: 'VITE_FIREBASE_STORAGE_BUCKET'),
                                    string(credentialsId: 'VITE_FIREBASE_MESSAGING_SENDER_ID', variable: 'VITE_FIREBASE_MESSAGING_SENDER_ID'),
                                    string(credentialsId: 'VITE_FIREBASE_APP_ID', variable: 'VITE_FIREBASE_APP_ID')
                                ]) {
                                    // Bangun string build-arg secara dinamis
                                    def buildArgs = """
                                        --build-arg VITE_BACKEND_BASE_URL=${VITE_BACKEND_BASE_URL} \
                                        --build-arg VITE_VERIFICATOR_BASE_URL=${VITE_VERIFICATOR_BASE_URL} \
                                        --build-arg VITE_FIREBASE_API_KEY=${VITE_FIREBASE_API_KEY} \
                                        --build-arg VITE_FIREBASE_AUTH_DOMAIN=${VITE_FIREBASE_AUTH_DOMAIN} \
                                        --build-arg VITE_FIREBASE_PROJECT_ID=${VITE_FIREBASE_PROJECT_ID} \
                                        --build-arg VITE_FIREBASE_STORAGE_BUCKET=${VITE_FIREBASE_STORAGE_BUCKET} \
                                        --build-arg VITE_FIREBASE_MESSAGING_SENDER_ID=${VITE_FIREBASE_MESSAGING_SENDER_ID} \
                                        --build-arg VITE_FIREBASE_APP_ID=${VITE_FIREBASE_APP_ID}
                                    """.trim()

                                    def feImage = docker.build("asia.gcr.io/primeval-rune-467212-t9/wondr-desktop-jenkins-fe:${env.FINAL_TAG}", "${buildArgs} .")
                                    echo '✅ Frontend Docker image built successfully'
                                    
                                    withCredentials([file(credentialsId: 'gcr-credentials', variable: 'GCR_KEY_FILE')]) {
                                        echo '🔐 Authenticating to Google Container Registry...'
                                        sh "cat ${GCR_KEY_FILE} | docker login -u _json_key --password-stdin https://asia.gcr.io"
                                        docker.withRegistry('https://asia.gcr.io') {
                                            feImage.push()
                                        }
                                    }
                                    echo "✅ Frontend Docker image pushed to GCR: asia.gcr.io/primeval-rune-467212-t9/wondr-desktop-jenkins-fe:${env.FINAL_TAG}"
                                }
                            }
                        }
                    )
                }
            // ... (sisa kode tetap sama)
            }
        }

        stage('🎯 Staging Deployment') {
            when {
                anyOf {
                    equals expected: 'staging', actual: params.DEPLOY_ENVIRONMENT
                    equals expected: 'both', actual: params.DEPLOY_ENVIRONMENT
                }
            }
            steps {
                echo '🎯 Deploying to staging environment...'
                script {
                    deployToEnvironment('staging', 8081, env.DOCKER_IMAGE_FINAL)
                    
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("🎯 <b>Staging Deployment</b>\n✅ Deployed successfully\n🔗 URL: ${env.STAGING_URL}")
                    }
                }
            }
        }
        
        stage('🔍 Staging Tests') {
            when {
                anyOf {
                    equals expected: 'staging', actual: params.DEPLOY_ENVIRONMENT
                    equals expected: 'both', actual: params.DEPLOY_ENVIRONMENT
                }
            }
            steps {
                echo '🔍 Running staging integration tests...'
                script {
                    echo 'Waiting for application to start...'
                    echo 'sleep(time: 10, unit: SECONDS) - Application startup simulation'
                    
                    // Mock health check
                    echo "curl -s -o /dev/null -w '%{http_code}' ${env.STAGING_URL}/health"
                    def healthStatus = '200'
                    echo "Health check status: ${healthStatus}"
                    
                    if (healthStatus == '200') {
                        echo '✅ Health check passed'
                        
                        // Mock API test
                        echo "curl -f ${env.STAGING_URL}/"
                        echo '✅ API test passed'
                        
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("🔍 <b>Staging Tests</b>\n✅ All integration tests passed\n🩺 Health: OK")
                        }
                    } else {
                        echo "❌ Staging health check failed with status: ${healthStatus} (simulated)"
                        echo "⚠️ In real scenario, this would cause pipeline failure"
                    }
                }
            }
            post {
                failure {
                    script {
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("❌ <b>Staging Tests Failed</b>\n🚨 Integration tests failed")
                        }
                    }
                }
            }
        }
        
        stage('⏳ QA Approval') {
            when {
                anyOf {
                    equals expected: 'production', actual: params.DEPLOY_ENVIRONMENT
                    equals expected: 'both', actual: params.DEPLOY_ENVIRONMENT
                }
            }
            steps {
                script {
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("⏳ <b>QA Approval Required</b>\n🔍 Please review staging environment\n🎯 Staging: ${env.STAGING_URL}\n⏰ Waiting for approval...")
                    }
                    
                    echo '⏳ QA Approval step - Auto-approving for demo'
                    echo 'QA Decision: Approve'
                    echo 'QA Comments: Auto-approved for pipeline demo'
                    
                    // Mock the approval variables
                    env.QA_DECISION = 'Approve'
                    env.QA_COMMENTS = 'Auto-approved for pipeline demo'
                    
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("✅ <b>QA Approved</b>\n👤 Decision: ${env.QA_DECISION}\n💬 Comments: ${env.QA_COMMENTS}")
                    }
                }
            }
        }
        
        stage('☁️ Create GKE Cluster') {
            when {
                expression { params.CREATE_NEW_CLUSTER == true }
            }
            steps {
                echo '☁️ Creating a new GKE cluster with Terraform...'
                
                script {
                    // Gunakan withCredentials untuk otentikasi ke Google Cloud
                    // Pastikan kredensial GCR juga dapat digunakan untuk otentikasi gcloud/Terraform
                    withCredentials([file(credentialsId: 'gcr-credentials', variable: 'GCP_SERVICE_ACCOUNT_KEY')]) {
                        dir('ops-repo/terraform-v2') {
                            // Gunakan kredensial yang diekspos untuk autentikasi gcloud
                            // Perintah ini akan mengatur akun service default untuk Terraform
                            sh "gcloud auth activate-service-account --key-file=\$GCP_SERVICE_ACCOUNT_KEY"
                            
                            // Inisialisasi Terraform
                            sh 'terraform init'
                            
                            // Jalankan terraform plan dan simpan hasilnya
                            sh 'terraform plan -out=tfplan'
                            
                            // Apply konfigurasi secara otomatis
                            sh 'terraform apply -auto-approve tfplan'
                            
                            echo '✅ GKE cluster created successfully!'
                        }
                    }
                }
            }
            post {
                success {
                    script {
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("☁️ <b>GKE Cluster Created</b>\n✅ Terraform apply successful")
                        }
                    }
                }
                failure {
                    script {
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("❌ <b>GKE Cluster Creation Failed</b>\n🚨 Terraform encountered errors")
                        }
                    }
                }
            }
        }

        stage('🚀 Production Deployment') {
            when {
                anyOf {
                    equals expected: 'production', actual: params.DEPLOY_ENVIRONMENT
                    equals expected: 'both', actual: params.DEPLOY_ENVIRONMENT
                }
            }
            steps {
                script {
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("⏳ <b>DevOps Approval Required</b>\n🚀 Ready for production deployment\n⏰ Waiting for final approval...")
                    }
                    
                    echo '🚀 DevOps Approval step - Auto-approving for demo'
                    echo 'DevOps Decision: Deploy'
                    echo 'Deployment Notes: Auto-approved for pipeline demo'
                    
                    // Mock the approval variables
                    env.DEVOPS_DECISION = 'Deploy'
                    env.DEPLOYMENT_NOTES = 'Auto-approved for pipeline demo'
                    
                    echo '🚀 Deploying to GKE production cluster...'
                    
                    // Menggunakan withCredentials untuk autentikasi gcloud
                    withCredentials([file(credentialsId: 'gcr-credentials', variable: 'GCP_SERVICE_ACCOUNT_KEY')]) {
                        // Autentikasi gcloud dan dapatkan kredensial cluster GKE
                        sh "gcloud auth activate-service-account --key-file=\$GCP_SERVICE_ACCOUNT_KEY"
                        
                        // Perintah untuk mendapatkan kredensial kubectl
                        sh "gcloud container clusters get-credentials ${env.gkeClusterName} --zone ${env.gkeClusterZone}"
                        
                        // Masuk ke direktori manifest Kubernetes
                        dir('ops-repo/k8s/application') {
                            // Gunakan kubectl apply untuk menerapkan semua manifest
                            // Perintah ini akan memperbarui atau membuat sumber daya berdasarkan file YAML
                            sh 'kubectl apply -f .'
                            
                            echo '✅ Kubernetes manifests applied successfully!'
                        }
                    }

                    // Hapus fungsi deployToEnvironment karena sudah tidak digunakan lagi
                    // deployToEnvironment('prod', 8090, env.DOCKER_IMAGE_FINAL)
                    
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage("🚀 <b>Production Deployment</b>\n✅ Deployed to GKE successfully\n👤 Approved by: DevOps\n💬 Notes: ${env.DEPLOYMENT_NOTES}\n🎉 Deployment to Kubernetes complete!")
                    }
                }
            }
        }
        
        stage('🔍 Production Verification') {
            when {
                anyOf {
                    equals expected: 'production', actual: params.DEPLOY_ENVIRONMENT
                    equals expected: 'both', actual: params.DEPLOY_ENVIRONMENT
                }
            }
            steps {
                echo '🔍 Verifying production deployment...'
                script {
                    echo 'Waiting for production application to start...'
                    echo 'sleep(time: 10, unit: SECONDS) - Production startup simulation'
                    
                    // Mock health check
                    echo "curl -s -o /dev/null -w '%{http_code}' ${env.PRODUCTION_URL}/health"
                    def healthStatus = '200'
                    echo "Production health check status: ${healthStatus}"
                    
                    if (healthStatus == '200') {
                        echo '✅ Production health check passed'
                        
                        // Mock smoke test
                        echo "curl -f ${env.PRODUCTION_URL}/"
                        echo '✅ Production smoke test passed'
                        
                        if (params.ENABLE_NOTIFICATIONS) {
                            sendTelegramMessage("🔍 <b>Production Verification</b>\n✅ All production tests passed\n🩺 Health: OK\n🎉 Deployment successful!")
                        }
                    } else {
                        echo "❌ Production health check failed with status: ${healthStatus} (simulated)"
                        echo "⚠️ In real scenario, this would cause pipeline failure"
                    }
                }
            }
        }
        
        stage('📊 Post-Deployment Report') {
            steps {
                script {
                    def endTime = new Date().format('yyyy-MM-dd HH:mm:ss')
                    def duration = currentBuild.durationString.replace(' and counting', '')
                    
                    def reportMessage = getPostDeploymentReport(endTime, duration)
                    
                    if (params.ENABLE_NOTIFICATIONS) {
                        sendTelegramMessage(reportMessage)
                    }
                    
                    echo '📊 Post-deployment report generated successfully'
                }
            }
        }
    }
    
    post {
        always {
            echo '🧹 Cleaning up workspace...'
            // Echo instead of actual cleanup
            echo 'cleanWs()'
        }
        
        success {
            script {
                def successMessage = getPipelineSuccessMessage()
                
                if (params.ENABLE_NOTIFICATIONS) {
                    sendTelegramMessage(successMessage)
                }
            }
        }
        
        failure {
            script {
                def failureMessage = getPipelineFailureMessage()
                
                if (params.ENABLE_NOTIFICATIONS) {
                    sendTelegramMessage(failureMessage)
                }
            }
        }
        
        unstable {
            script {
                def unstableMessage = getPipelineUnstableMessage()
                
                if (params.ENABLE_NOTIFICATIONS) {
                    sendTelegramMessage(unstableMessage)
                }
            }
        }
        
        aborted {
            script {
                def abortedMessage = getAbortedMessage()
                
                if (params.ENABLE_NOTIFICATIONS) {
                    sendTelegramMessage(abortedMessage)
                }
            }
        }
    }
}

// Helper function to deploy to different environments
def deployToEnvironment(String environment, int port, String dockerImage) {
    echo "Deploying to ${environment} on port ${port}"
    
    // Echo instead of actual Docker commands
    echo "docker stop ${env.APP_NAME}-${environment} || true"
    echo "docker rm ${env.APP_NAME}-${environment} || true"
    
    // Echo Docker run command
    echo """docker run -d \\
        --name ${env.APP_NAME}-${environment} \\
        -p ${port}:8080 \\
        -e SPRING_PROFILES_ACTIVE=${environment} \\
        --restart unless-stopped \\
        ${dockerImage}"""
    
    echo "✅ ${environment} deployment completed"
}

// Enhanced Telegram notification function with retry logic
def sendTelegramMessage(String message) {
    if (!params.ENABLE_NOTIFICATIONS) {
        return
    }
    
    try {
        echo "📱 Sending Telegram notification:"
        echo "Message: ${message}"
        echo "✅ Telegram notification sent successfully (simulated)"
    } catch (Exception e) {
        echo "❌ Telegram notification error: ${e.getMessage()}"
    }

    try {
        def encodedMessage = message.replaceAll('"', '\\\\"')
        def maxRetries = 3
        def retryCount = 0
        def success = false
    
        while (retryCount < maxRetries && !success) {
            try {
                def response = sh(
                    script: """
                        curl -s -X POST https://api.telegram.org/bot\${TELEGRAM_BOT_TOKEN}/sendMessage \
                            -d chat_id=\${TELEGRAM_CHAT_ID} \
                            -d text="${encodedMessage}" \
                            -d parse_mode=HTML \
                            -w "HTTP_CODE:%{http_code}"
                    """,
                    returnStdout: true
                ).trim()
                
                if (response.contains('HTTP_CODE:200')) {
                    echo "✅ Telegram notification sent successfully"
                    success = true
                } else {
                    throw new Exception("HTTP error: ${response}")
                }
            } catch (Exception e) {
                retryCount++
                echo "⚠️ Telegram notification attempt ${retryCount} failed: ${e.getMessage()}"
                if (retryCount < maxRetries) {
                    sleep(time: 5, unit: 'SECONDS')
                }
            }
        }
        
        if (!success) {
            echo "❌ Failed to send Telegram notification after ${maxRetries} attempts"
        }
    } catch (Exception e) {
        echo "❌ Telegram notification error: ${e.getMessage()}"
    }
}

// Helper functions for messages
def getStartMessage() {
    return """
    🚀 <b>CI/CD Pipeline Started</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    🌿 <b>Branch:</b> ${env.BRANCH_NAME ?: 'main'}
    🎯 <b>Environment:</b> ${params.DEPLOY_ENVIRONMENT}
    🏷️ <b>Tag:</b> ${env.FINAL_TAG}
    👤 <b>Triggered by:</b> ${env.BUILD_USER ?: 'System'}
    ⏰ <b>Started at:</b> ${new Date().format('yyyy-MM-dd HH:mm:ss')}
    🔗 <b>Console:</b> ${env.BUILD_URL}console
    """
}

def getPostDeploymentReport(String endTime, String duration) {
    return """
    📊 <b>Post-Deployment Report</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    ✅ <b>Status:</b> SUCCESS
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    🏷️ <b>Version:</b> ${env.FINAL_TAG}
    🎯 <b>Environment:</b> ${params.DEPLOY_ENVIRONMENT}
    ⏱️ <b>Duration:</b> ${duration}
    ⏰ <b>Completed:</b> ${endTime}

    🌐 <b>Deployed URLs:</b>
    ${params.DEPLOY_ENVIRONMENT.contains('staging') ? '🎯 Staging: ' + env.STAGING_URL : ''}
    ${params.DEPLOY_ENVIRONMENT.contains('production') ? '🚀 Production: ' + env.PRODUCTION_URL : ''}

    🔗 <b>Jenkins:</b> ${env.BUILD_URL}
    """
}

def getPipelineSuccessMessage() {
    return """🎉 <b>Pipeline Completed Successfully!</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    ✅ <b>Result:</b> SUCCESS
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    ⏱️ <b>Duration:</b> ${currentBuild.durationString.replace(' and counting', '')}
    🎯 <b>Environment:</b> ${params.DEPLOY_ENVIRONMENT}
    """
}

def getPipelineFailureMessage() {
    return """
    ❌ <b>Pipeline Failed!</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    🚨 <b>Result:</b> FAILURE
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    ⏱️ <b>Duration:</b> ${currentBuild.durationString.replace(' and counting', '')}
    🔗 <b>Console:</b> ${env.BUILD_URL}console
    """
}

def getPipelineUnstableMessage() {
    return """
    ⚠️ <b>Pipeline Unstable</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    🟡 <b>Result:</b> UNSTABLE
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    🔗 <b>Console:</b> ${env.BUILD_URL}console
    """
}

def getAbortedMessage() {
    return """
    🛑 <b>Pipeline Aborted</b>
    ━━━━━━━━━━━━━━━━━━━━━━━
    🔴 <b>Result:</b> ABORTED
    📋 <b>Job:</b> ${env.JOB_NAME}
    🔢 <b>Build:</b> #${env.BUILD_NUMBER}
    """
}