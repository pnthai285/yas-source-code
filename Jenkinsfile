// ============================================================
// JENKINSFILE - YAS CI/CD PIPELINE 
// Monorepo: 22 microservices (Java Spring Boot + Next.js)
// Infrastructure: AWS VPC, Spot Agents (NVMe), Local Registry, S3 Cache
// Best-Practices: Retry at Master, Smart Routing, Maven Reactor, Graceful Error Handling
// ============================================================

def isPR = false
def maxInfraRetry = 1
def buildErrors = []

pipeline {
    // ============================================================
    // 1. AGENT & GLOBAL OPTIONS
    // ============================================================
    agent none

    options {
        // ✅ Retry handled at Heavy Build stage
        // ✅ Chống zombie pipeline
        timeout(time: 60, unit: 'MINUTES')
        
        // ✅ Tránh lãng phí Spot capacity
        disableConcurrentBuilds()
        
        // ✅ Giữ log để debug
        buildDiscarder(logRotator(numToKeepStr: '10'))
        
        // ✅ Log có timestamp
        timestamps()
        
        // ✅ GitHub commit status handled by SCM integration
    }

    // ============================================================
    // 2. ENVIRONMENT VARIABLES (Global - không gọi AWS ở đây)
    // ============================================================
    environment {
        BRANCH_NAME        = "${env.BRANCH_NAME}"
        CHANGE_ID          = "${env.CHANGE_ID}"
        CHANGE_TARGET      = "${env.CHANGE_TARGET ?: 'main'}"
        GIT_COMMIT_SHORT   = "unknown"
        
        // SonarCloud project keys mapping (đã tạo trên UI)
        PROJECT_KEYS = '''
            backoffice:pnthai285_yas-backoffice
            backoffice-bff:pnthai285_yas-backoffice-bff
            cart:pnthai285_yas-cart
            common-library:pnthai285_yas-common-library
            customer:pnthai285_yas-customer
            delivery:pnthai285_yas-delivery
            inventory:pnthai285_yas-inventory
            location:pnthai285_yas-location
            media:pnthai285_yas-media
            order:pnthai285_yas-order
            payment:pnthai285_yas-payment
            payment-paypal:pnthai285_yas-payment-paypal
            product:pnthai285_yas-product
            promotion:pnthai285_yas-promotion
            rating:pnthai285_yas-rating
            recommendation:pnthai285_yas-recommendation
            sampledata:pnthai285_yas-sampledata
            search:pnthai285_yas-search
            storefront:pnthai285_yas-storefront
            storefront-bff:pnthai285_yas-storefront-bff
            tax:pnthai285_yas-tax
            webhook:pnthai285_yas-webhook
        '''

        SONAR_ORG       = 'pnthai285'
        SONAR_HOST_URL  = 'https://sonarcloud.io'

        SNYK_CREDENTIALS_ID = 'snyk-api-token-yas'

        SLACK_TEAM_DOMAIN   = 'pnt-8oe4827'
        SLACK_CHANNEL       = '#cicd-notifications'
        SLACK_CREDENTIALS_ID = 'slack-bot-token-yas'

        // GitHub API – dùng để phát hiện open PR và abort branch build thừa
        // Credential 'github-token-yas' type: GitHub App (Jenkins GitHub Branch Source)
        GITHUB_OWNER = 'pnthai285'
        GITHUB_REPO  = 'pnthai285/yas-source-code'

        // Module lists are set in Smart Routing
    }

    // ============================================================
    // 3. STAGES
    // ============================================================
    stages {
        
        // ============================================================
        // STAGE 1: SMART ROUTING (Chạy trên Master - nhẹ)
        // ============================================================
        stage('Smart Routing') {
            agent { label 'built-in' }
            steps {
                script {
                    runSmartRoutingStage()
                }
            }
        }

        // ============================================================
        // STAGE 2: GITLEAKS SCAN (Security - chạy trên built-in)
        // ============================================================
        stage('Gitleaks Scan') {
            when { expression { env.SHOULD_BUILD == 'true' } }
            agent { label 'built-in' }
            steps {
                script { runGitleaksStage() }
            }
        }

        // ============================================================
        // STAGE 3: SKIP HEAVY BUILD (Nếu không có thay đổi)
        // ============================================================
        stage('Skip Heavy Build') {
            when { expression { env.SHOULD_BUILD == 'false' } }
            steps {
                echo "[INFO] === NO CHANGES DETECTED ==="
                echo "[INFO] No changes in any module. Skipping heavy build."
                echo "[INFO] Pipeline completed successfully (no-op)."
            }
        }

        // ============================================================
        // STAGE 4: HEAVY BUILD TRÊN SPOT AGENT (NVMe)
        // ============================================================
        stage('Heavy Build') {
            when { expression { env.SHOULD_BUILD == 'true' } }
            agent { label 'aws-spot-nvme' }
            
            stages {
                // ------------------------------------------------
                // 4.1: Checkout & Setup AWS config
                // ------------------------------------------------
                stage('Checkout & Setup') {
                    options { retry(maxInfraRetry) }
                    steps {
                        script {
                            echo "[INFO] === CHECKOUT & SETUP STARTED ==="
                            
                            try {
                                def scmVars = checkout scm
                                resolveCommitInfo(scmVars)
                                
                                // Lấy AWS config với error handling
                                env.ACCOUNT_ID = withAWS(region: 'us-east-1') {
                                    sh(script: 'aws sts get-caller-identity --query Account --output text', 
                                       returnStdout: true, label: 'get-account-id').trim()
                                }
                                env.CACHE_BUCKET = "yas-cache-${env.ACCOUNT_ID}"
                                
                                // Lấy Hub IP từ SSM với retry
                                env.HUB_IP = sh(script: """
                                    aws ssm get-parameter \
                                        --name /yas/hub/private_ip \
                                        --region us-east-1 \
                                        --query Parameter.Value \
                                        --output text
                                """, returnStdout: true, label: 'get-hub-ip').trim()
                                
                                env.REGISTRY = sanitizeRegistry("${env.HUB_IP}:5000")
                                
                                echo "[INFO] Account ID: ${env.ACCOUNT_ID}"
                                echo "[INFO] Cache bucket: ${env.CACHE_BUCKET}"
                                echo "[INFO] Local registry: ${env.REGISTRY}"
                                echo "[INFO] Hub IP: ${env.HUB_IP}"
                                
                            } catch (Exception e) {
                                echo "[ERROR] Setup failed: ${e.message}"
                                error "❌ Failed to initialize AWS config: ${e.message}. Check IAM role on Spot Agent."
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.2: Verify Registry Connectivity
                // ------------------------------------------------
                stage('Verify Registry Connectivity') {
                    steps {
                        script {
                            echo "[INFO] === VERIFYING REGISTRY CONNECTIVITY ==="
                            
                            try {
                                retry(maxInfraRetry) {
                                    sh """
                                        echo "[INFO] Testing registry connection to ${env.HUB_IP}:5000"
                                        curl -f --max-time 10 --retry 2 --retry-delay 2 \
                                            http://${env.HUB_IP}:5000/v2/_catalog \
                                            || { echo "[ERROR] Registry not reachable!"; exit 1; }
                                        echo "[OK] Registry is accessible"
                                    """
                                }
                            } catch (Exception e) {
                                echo "[ERROR] Registry connectivity check failed: ${e.message}"
                                error "❌ Cannot connect to Local Registry at ${env.REGISTRY}. Pipeline aborted."
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.3: Restore Maven Cache từ S3 (via Gateway Endpoint)
                // ------------------------------------------------
                stage('Restore Maven Cache') {
                    options { retry(maxInfraRetry) }
                    steps {
                        script {
                            echo "[INFO] === RESTORING MAVEN CACHE ==="
                            
                            try {
                                // Generate cache key dựa trên content (pom.xml + package.json)
                                env.CACHE_KEY = sh(
                                    script: """
                                        find . '(' -name 'pom.xml' -o -name 'package.json' ')' -exec md5sum {} + 2>/dev/null |
                                        md5sum | awk '{print \$1}'
                                    """,
                                    returnStdout: true,
                                    label: 'generate-cache-key'
                                ).trim()
                                
                                echo "[INFO] Cache key: ${env.CACHE_KEY}"
                                
                                // Restore cache với fallback strategy
                                def cacheRestored = false
                                
                                // Try 1: Branch cache
                                if (!cacheRestored) {
                                    def exitCode = sh(
                                        script: """
                                            set +e
                                            aws s3 cp s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz ./cache.tar.gz 2>/dev/null
                                            
                                            if [ -f cache.tar.gz ]; then
                                                aws s3 cp s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz.md5 ./cache.tar.gz.md5 2>/dev/null || true
                                                
                                                mkdir -p ~/.m2
                                                
                                                if [ -f cache.tar.gz.md5 ]; then
                                                    echo "[INFO] Verifying cache checksum..."
                                                    md5sum -c cache.tar.gz.md5 || {
                                                        echo "[WARN] Cache corrupt or mismatch. Deleting and falling back to fresh build."
                                                        rm -f cache.tar.gz cache.tar.gz.md5
                                                        exit 1
                                                    }
                                                fi
                                                
                                                tar -xzf cache.tar.gz -C ~/.m2
                                                echo "[OK] Cache restored successfully"
                                            else
                                                exit 1
                                            fi
                                        """,
                                        returnStatus: true,
                                        label: 'restore-branch-cache'
                                    )
                                    if (exitCode == 0) {
                                        echo "[OK] Restored branch cache"
                                        cacheRestored = true
                                    }
                                }
                                
                                // Try 2: Main cache fallback
                                if (!cacheRestored) {
                                    def exitCode = sh(
                                        script: """
                                            set +e
                                            aws s3 cp s3://${env.CACHE_BUCKET}/maven/main-${env.CACHE_KEY}.tar.gz ./cache.tar.gz 2>/dev/null
                                            
                                            if [ -f cache.tar.gz ]; then
                                                aws s3 cp s3://${env.CACHE_BUCKET}/maven/main-${env.CACHE_KEY}.tar.gz.md5 ./cache.tar.gz.md5 2>/dev/null || true
                                                
                                                mkdir -p ~/.m2
                                                
                                                if [ -f cache.tar.gz.md5 ]; then
                                                    echo "[INFO] Verifying cache checksum..."
                                                    md5sum -c cache.tar.gz.md5 || {
                                                        echo "[WARN] Cache corrupt or mismatch. Deleting and falling back to fresh build."
                                                        rm -f cache.tar.gz cache.tar.gz.md5
                                                        exit 1
                                                    }
                                                fi
                                                
                                                tar -xzf cache.tar.gz -C ~/.m2
                                                echo "[OK] Cache restored successfully"
                                            else
                                                exit 1
                                            fi
                                        """,
                                        returnStatus: true,
                                        label: 'restore-main-cache'
                                    )
                                    if (exitCode == 0) {
                                        echo "[OK] Restored main cache"
                                        cacheRestored = true
                                    }
                                }
                                
                                if (!cacheRestored) {
                                    echo "[INFO] No cache found, starting fresh build"
                                }
                                
                                // Cleanup temp files
                                sh "rm -f cache.tar.gz cache.tar.gz.md5 2>/dev/null || true"
                                
                            } catch (Exception e) {
                                echo "[WARN] Cache restore failed: ${e.message}. Continuing with fresh build."
                                // Không fail pipeline, chỉ log warning và build từ đầu
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.4: Compile & Package (Maven Reactor)
                // ------------------------------------------------
                stage('Compile & Package') {
                    steps {
                        script {
                            echo "[INFO] === COMPILE & PACKAGE STARTED ==="
                            runStageOrWarn('Compile & Package') { runCompileAndPackageStage() }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.5: Snyk Security Scan
                // ------------------------------------------------
                stage('Snyk Security Scan') {
                    steps {
                        script {
                            echo "[INFO] === SNYK SECURITY SCAN STARTED ==="
                            // PR  → isPR=true  → runStageOrWarn calls body() directly → error() propagates → FAILED
                            // Push → isPR=false → runStageOrWarn catches exception → UNSTABLE + continues
                            runStageOrWarn('Snyk Security Scan') { runSnykSecurityScanStage() }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.6: Unit Tests
                // ------------------------------------------------
                stage('Unit Tests') {
                    steps {
                        script {
                            echo "[INFO] === UNIT TESTS STARTED ==="
                            runStageOrWarn('Unit Tests') { runUnitTestsStage() }
                        }
                    }
                    post {
                        always {
                            script {
                                // Archive test results với graceful handling
                                try {
                                    junit allowEmptyResults: true,
                                          testResults: '**/target/surefire-reports/*.xml',
                                          skipPublishingChecks: true
                                } catch (Exception e) {
                                    echo "[WARN] Failed to archive JUnit results: ${e.message}"
                                }
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.6: Integration Tests (Testcontainers)
                // ------------------------------------------------
                stage('Integration Tests') {
                    steps {
                        script {
                            echo "[INFO] === INTEGRATION TESTS STARTED ==="
                            runStageOrWarn('Integration Tests') { runIntegrationTestsStage() }
                        }
                    }
                    post {
                        always {
                            script {
                                try {
                                    // Archive Failsafe reports
                                    junit allowEmptyResults: true,
                                          testResults: '**/target/failsafe-reports/*.xml',
                                          skipPublishingChecks: true
                                    
                                    // Archive JaCoCo coverage
                                    archiveArtifacts artifacts: '**/target/site/jacoco/jacoco.xml',
                                                 allowEmptyArchive: true,
                                                 fingerprint: true

                                    // Publish JaCoCo report for Jenkins charts (Coverage API plugin)
                                    recordCoverage tools: [[parser: 'JACOCO', pattern: '**/target/site/jacoco/jacoco.xml']],
                                                   sourceDirectories: [[path: '.']]
                                } catch (Exception e) {
                                    echo "[WARN] Failed to archive integration test artifacts: ${e.message}"
                                }
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.7: SonarQube Analysis
                // ------------------------------------------------
                stage('SonarQube Analysis') {
                    when {
                        anyOf {
                            branch 'main'
                            expression { env.CHANGE_ID != null }
                        }
                        expression { currentBuild.result == null || currentBuild.result in ['SUCCESS', 'UNSTABLE'] }
                    }
                    steps {
                        script {
                            echo "[INFO] === SONARQUBE ANALYSIS STARTED ==="
                            runStageOrWarn('SonarQube Analysis') { runSonarAnalysisStage() }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.8: Quality Gate (Coverage Check cho PR)
                // ------------------------------------------------
                stage('Quality Gate') {
                    when { expression { env.CHANGE_ID != null && env.SONAR_RAN == 'true' } }
                    steps {
                        script {
                            echo "[INFO] === QUALITY GATE CHECK STARTED ==="
                            
                            try {
                                // ✅ Timeout CRITICAL: tránh zombie pipeline
                                timeout(time: 15, unit: 'MINUTES') {
                                    def gateResult = waitForQualityGate abortPipeline: false
                                    if (gateResult.status != 'OK') {
                                        echo "[ERROR] Quality Gate failed: ${gateResult.status}"
                                        // Log chi tiết để debug
                                        echo "[DEBUG] Quality Gate details: ${gateResult}"
                                        
                                        // Fail pipeline cho PR
                                        if (env.CHANGE_ID) {
                                            error "❌ Quality Gate failed for PR. Coverage or quality metrics below threshold."
                                        }
                                    } else {
                                        echo "[OK] Quality Gate passed"
                                    }
                                }
                                
                            } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
                                echo "[ERROR] Quality Gate check timed out after 15 minutes"
                                error "❌ Quality Gate timeout. Check SonarCloud webhook connectivity."
                            } catch (Exception e) {
                                echo "[ERROR] Quality Gate check failed: ${e.message}"
                                // Cho PR thì fail, cho main thì warning
                                if (env.CHANGE_ID) {
                                    throw e
                                } else {
                                    echo "[WARN] Quality Gate error on main branch, continuing"
                                }
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.10: Build & Push Docker Image
                // ------------------------------------------------
                stage('Build & Push Docker Image') {
                    when { expression { env.SHOULD_BUILD == 'true' } }
                    steps {
                        script {
                            echo "[INFO] === DOCKER BUILD & PUSH STARTED ==="
                            runStageOrWarn('Build & Push Docker Image') { runBuildAndPushStage() }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.11: GitOps Handoff (Log Only - Placeholder)
                // ------------------------------------------------
                stage('GitOps Handoff (Template)') {
                    when { branch 'main' }
                    steps {
                        script {
                            echo "[INFO] === GITOPS HANDOFF (PLACEHOLDER) ==="
                            echo "=================================================="
                            echo "[GITOPS] Khi có repo yas-gitops-config, implement:"
                            echo "1. Clone repo với lock('gitops-repo') để tránh conflict"
                            echo "2. Update deployment.yaml với immutable tag: ${env.GIT_COMMIT_SHORT}"
                            echo "3. Commit với message: 'Auto-update: ${env.GIT_COMMIT_SHORT}'"
                            echo "4. git pull --rebase origin main trước push"
                            echo "5. Push để ArgoCD sync"
                            echo ""
                            echo "Affected modules:"
                            (env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',') : []).each { module ->
                                echo "  - ${module}: image = ${env.REGISTRY}/yas-${module}:${env.GIT_COMMIT_SHORT}"
                            }
                            echo "=================================================="
                            
                            // TODO: Uncomment khi có repo content
                            /*
                            lock('gitops-repo') {
                                retry(3) {
                                    withCredentials([usernamePassword(credentialsId: 'gitops-github-token', 
                                                                      usernameVariable: 'GIT_USER', 
                                                                      passwordVariable: 'GIT_PASS')]) {
                                        sh """
                                            git clone https://\${GIT_USER}:\${GIT_PASS}@github.com/your-org/yas-gitops-config.git temp-gitops
                                            cd temp-gitops
                                            # Update manifests...
                                            git add .
                                            git commit -m "Auto-update images to ${GIT_COMMIT_SHORT}"
                                            git pull --rebase origin main
                                            git push origin main
                                            cd .. && rm -rf temp-gitops
                                        """
                                    }
                                }
                            }
                            */
                        }
                    }
                }

                // ------------------------------------------------
                // 4.12: Save Maven Cache to S3
                // ------------------------------------------------
                stage('Save Maven Cache') {
                    when { expression { currentBuild.result == null || currentBuild.result in ['SUCCESS', 'UNSTABLE'] } }
                    steps {
                        script {
                            echo "[INFO] === SAVING MAVEN CACHE ==="
                            runSaveMavenCacheStage()
                        }
                    }
                }
            }
        }

        // ============================================================
        // STAGE 5: Pipeline Metrics (Optional)
        // ============================================================
        stage('Finalize Build Result') {
            steps {
                script {
                    if (!isPR && buildErrors.size() > 0) {
                        echo "[WARN] Pipeline completed with ${buildErrors.size()} error(s):"
                        buildErrors.each { err -> echo "  - ${err}" }
                        currentBuild.result = 'UNSTABLE'
                    } else if (isPR && currentBuild.result == 'UNSTABLE') {
                        currentBuild.result = 'FAILURE'
                        error 'PR build unstable. Fix issues before merging.'
                    }
                }
            }
        }

        stage('Pipeline Metrics') {
            steps {
                script {
                    def duration = currentBuild.duration / 1000
                    echo "⏱️ Pipeline completed in ${duration} seconds"
                    
                    // Có thể gửi metrics về CloudWatch/Grafana nếu cần
                    // Ví dụ: aws cloudwatch put-metric-data ...
                }
            }
        }
    }

    // ============================================================
    // 5. POST ACTIONS (Always run)
    // ============================================================
    post {
        always {
            script {
                node('built-in') {
                    echo "[INFO] === POST ACTIONS: CLEANUP ==="
                    
                    try {
                        // Archive test results (final fallback)
                        junit allowEmptyResults: true,
                              testResults: '**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml',
                              skipPublishingChecks: true
                        
                        // Archive coverage reports
                        archiveArtifacts artifacts: '**/target/site/jacoco/jacoco.xml, **/coverage/coverage-final.json',
                                     allowEmptyArchive: true,
                                     fingerprint: true
                        
                        // Clean workspace
                        cleanWs(disableDeferredWipeout: true, deleteDirs: true)
                        echo "[INFO] Workspace cleaned."
                        
                    } catch (Exception e) {
                        echo "[WARN] Post-action cleanup failed: ${e.message}"
                    }
                }
            }
        }
        
        failure {
            script {
                echo "[ERROR] === PIPELINE FAILED ==="
                
                // Slack notification với error handling
                try {
                    slackSend teamDomain: env.SLACK_TEAM_DOMAIN,
                              channel: env.SLACK_CHANNEL,
                              tokenCredentialId: env.SLACK_CREDENTIALS_ID,
                              color: 'danger',
                              message: "❌ Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                                       "Branch: ${env.BRANCH_NAME}\n" +
                                       "Commit: ${env.GIT_COMMIT_SHORT}\n" +
                                       "URL: ${env.BUILD_URL}"
                } catch (Exception e) {
                    echo "[WARN] Failed to send Slack notification: ${e.message}"
                }
            }
        }
        
        success {
            script {
                echo "[SUCCESS] === PIPELINE COMPLETED SUCCESSFULLY ==="
                
                try {
                    slackSend teamDomain: env.SLACK_TEAM_DOMAIN,
                              channel: env.SLACK_CHANNEL,
                              tokenCredentialId: env.SLACK_CREDENTIALS_ID,
                              color: 'good',
                              message: "✅ Pipeline PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                                       "Branch: ${env.BRANCH_NAME}\n" +
                                       "Commit: ${env.GIT_COMMIT_SHORT}\n" +
                                       "URL: ${env.BUILD_URL}"
                } catch (Exception e) {
                    echo "[WARN] Failed to send Slack notification: ${e.message}"
                }
            }
        }
        
        unstable {
            script {
                echo "[WARN] === PIPELINE COMPLETED WITH WARNINGS ==="
                try {
                    slackSend teamDomain: env.SLACK_TEAM_DOMAIN,
                              channel: env.SLACK_CHANNEL,
                              tokenCredentialId: env.SLACK_CREDENTIALS_ID,
                              color: 'warning',
                              message: "⚠️ Pipeline UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                                       "Branch: ${env.BRANCH_NAME}\n" +
                                       "Check test failures or quality warnings"
                } catch (Exception e) {
                    echo "[WARN] Failed to send Slack notification: ${e.message}"
                }
            }
        }
    }
}

// ============================================================
// HELPER FUNCTIONS (Define outside pipeline block)
// ============================================================

/**
 * Retry với exponential backoff cho các operation có thể fail transient
 * @param maxAttempts Số lần retry tối đa
 * @param baseDelaySeconds Delay cơ bản giữa các lần retry (giây)
 * @param closure Code block cần retry
 */
def retryWithBackoff(int maxAttempts, int baseDelaySeconds, Closure closure) {
    def attempts = 0
    def lastException
    
    while (attempts < maxAttempts) {
        try {
            return closure.call()
        } catch (Exception e) {
            attempts++
            lastException = e
            
            if (attempts >= maxAttempts) {
                echo "[ERROR] Operation failed after ${maxAttempts} attempts: ${e.message}"
                throw e
            }
            
            // Exponential backoff: baseDelay * 2^(attempt-1)
            def delay = baseDelaySeconds * (1 << (attempts - 1))
            echo "[WARN] Attempt ${attempts}/${maxAttempts} failed. Retrying in ${delay}s... Error: ${e.message}"
            sleep time: delay, unit: 'SECONDS'
        }
    }
    
    // Fallback (không nên reach here)
    throw lastException ?: new RuntimeException("Retry failed with unknown error")
}

/**
 * Run a stage with fail-fast for PR and continue-on-failure for branch builds.
 */
def runStageOrWarn(String stageName, Closure body) {
    if (this.binding?.hasVariable('buildErrors') == false) {
        this.binding.setVariable('buildErrors', [])
    }
    if (isPR) {
        body()
    } else {
        try {
            body()
        } catch (Exception e) {
            buildErrors << "[${stageName}] ${e.message}"
            currentBuild.result = 'UNSTABLE'
            echo "[WARN] ${stageName} failed, continuing pipeline."
        }
    }
}

def runSmartRoutingStage() {
    echo "[INFO] === SMART ROUTING STARTED ==="
    echo "[INFO] Target branch: ${CHANGE_TARGET}"
    env.AFFECTED_MODULES = ''
    env.COMMON_LIB_CHANGED = 'false'
    env.SHOULD_BUILD = 'false'
    env.SONAR_RAN = 'false'
    isPR = env.CHANGE_ID && env.CHANGE_ID.toString() != 'null' && env.CHANGE_ID.toString().trim() != ''
    maxInfraRetry = isPR ? 1 : 2

    try {
        def computedCommit = env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : ''
        if (!computedCommit?.matches(/^[a-fA-F0-9]{7}$/)) {
            computedCommit = sh(
                script: "git rev-parse --short=7 HEAD 2>/dev/null || true",
                returnStdout: true,
                label: 'resolve-commit-short'
            ).trim()
        }
        env.GIT_COMMIT_SHORT = (computedCommit?.matches(/^[a-fA-F0-9]{7}$/)) ? computedCommit.toLowerCase() : 'unknown'
        // rawBranchName giữ tên gốc (có '/') để dùng query GitHub API
        def rawBranchName = env.BRANCH_NAME?.trim()
            ?: env.CHANGE_BRANCH?.trim()
            ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceFirst(/^origin\//, '').trim() : null)
            ?: 'detached'
        env.BRANCH_NAME = rawBranchName.replaceAll('/', '-')

        // ============================================================
        // "Single Optimal Build per Commit"
        // ============================================================
        // Khi branch ĐANG MỞ PR vào main, GitHub bắn 2 webhook (push + pull_request).
        // Jenkins sẽ kích hoạt cả 2 job: Branch Build + PR Build.
        // Branch Build lúc này hoàn toàn vô nghĩa — chỉ tốn tài nguyên Spot.
        // → Nếu đây là Branch Build (không phải PR job) VÀ GitHub API xác nhận
        //   nhánh này đang có open PR, ta ABORT ngay lập tức để nhường chỗ cho PR Build.
        if (!isPR && env.BRANCH_NAME != 'main') {
            try {
                // GitHub App credential: usernamePassword — username=App ID, password=installation token
                withCredentials([usernamePassword(credentialsId: 'github-token-yas', usernameVariable: 'GH_APP_ID', passwordVariable: 'GH_TOKEN')]) {
                    def openPRNumber = sh(
                        script: """
                            curl -sf \\
                                -H "Authorization: token \${GH_TOKEN}" \\
                                -H "Accept: application/vnd.github.v3+json" \\
                                "https://api.github.com/repos/${GITHUB_REPO}/pulls?state=open&head=${GITHUB_OWNER}:${rawBranchName}&per_page=1" \\
                            | python3 -c "import sys,json; prs=json.load(sys.stdin); print(prs[0]['number'] if prs else '')" 2>/dev/null || echo ''
                        """,
                        returnStdout: true,
                        label: 'check-open-pr-for-branch'
                    ).trim()

                    if (openPRNumber) {
                        echo "[INFO] 🔴 Branch '${env.BRANCH_NAME}' has open PR #${openPRNumber}."
                        echo "[INFO] Aborting Branch Build — PR Build #${openPRNumber} is the authoritative CI for this commit."
                        // ABORTED không gửi Slack failure, không block merge
                        currentBuild.result = 'ABORTED'
                        error("Branch build superseded: PR #${openPRNumber} is open. Only PR Build will run (industry best-practice).")
                    } else {
                        echo "[INFO] 🟢 No open PR found for branch '${env.BRANCH_NAME}'. Proceeding with Branch Build."
                    }
                }
            } catch (Exception e) {
                if (currentBuild.result == 'ABORTED') throw e
                // Nếu GitHub API lỗi (network, token hết hạn...) → tiếp tục build bình thường
                // Không nên block pipeline chỉ vì không check được PR status
                echo "[WARN] Could not query GitHub API to check for open PRs: ${e.message}. Proceeding with Branch Build as fallback."
            }
        }

        // Fetch target branch để có merge base chính xác
        sh "git fetch origin ${CHANGE_TARGET}:refs/remotes/origin/${CHANGE_TARGET} --depth=50"

        // (PUSH vs PR)
        def diffScript = ""
        if (isPR) {
            echo "[INFO] Smart Routing Mode: Pull Request (3-dot diff from origin/${CHANGE_TARGET})"
            diffScript = "git diff --name-only origin/${CHANGE_TARGET}...HEAD"
        } else {
            def prevCommit = env.GIT_PREVIOUS_COMMIT ?: 'HEAD~1'
            echo "[INFO] Smart Routing Mode: Feature Branch Push (diff from ${prevCommit} to HEAD)"
            diffScript = """
                # Kiểm tra xem prevCommit có tồn tại trong git tree không
                if git rev-parse --verify --quiet ${prevCommit} >/dev/null; then
                    git diff --name-only ${prevCommit}..HEAD
                else
                    echo "[WARN] Previous commit ${prevCommit} not found in tree, falling back to HEAD~1" >&2
                    git diff --name-only HEAD~1..HEAD
                fi
            """
        }

        def changedFiles = sh(
            script: """
                set -e
                ${diffScript}
            """,
            returnStdout: true,
            label: 'git-diff'
        ).trim().split('\n').findAll { it && !it.isEmpty() }

        echo "[DEBUG] Changed files: ${changedFiles.take(10)}${changedFiles.size() > 10 ? '...' : ''}"

        // Tìm danh sách modules (pom.xml hoặc package.json)
        def allModules = sh(
            script: '''find . -maxdepth 2 '(' -name 'pom.xml' -o -name 'package.json' ')' -printf '%h\n' 2>/dev/null | sed 's|^[.]/||' | grep -v '^[.]$' | sort -u''',
            returnStdout: true,
            label: 'find-modules'
        ).trim().split('\n').findAll { it && !it.isEmpty() }

        // Tính toán affected modules
        def affected = []
        def commonLibChanged = false

        changedFiles.each { file ->
            // Check common-library change (hiệu ứng domino)
            if (file.startsWith('common-library/')) {
                commonLibChanged = true
                echo "[INFO] common-library changed - will build all dependents"
            }

            // Find which module this file belongs to
            def module = allModules.find { m ->
                file.startsWith("${m}/") || file == "${m}/pom.xml" || file == "${m}/package.json"
            }

            if (module && !affected.contains(module)) {
                affected << module
                echo "[INFO] Affected module: ${module}"
            }
        }

        // Set environment variables cho các stage sau
        def affectedModules = affected.join(',')
        def shouldBuildFlag = (affectedModules?.trim() ? true : false) || commonLibChanged
        env.AFFECTED_MODULES = affectedModules
        env.COMMON_LIB_CHANGED = commonLibChanged.toString()
        env.SHOULD_BUILD = shouldBuildFlag ? 'true' : 'false'

        echo "[RESULT] Affected modules: ${affectedModules ?: 'none'}"
        echo "[RESULT] Common lib changed: ${env.COMMON_LIB_CHANGED}"
        echo "[RESULT] Should build: ${env.SHOULD_BUILD}"

    } catch (Exception e) {
        echo "[ERROR] Smart Routing failed: ${e.message}"
        // Vẫn cho phép pipeline tiếp tục với SHOULD_BUILD=true để không bỏ sót build
        env.SHOULD_BUILD = 'true'
        throw e
    }
}

def runGitleaksStage() {
    def gitleaksVersion = '8.18.0'
    def gitleaksBin = "${WORKSPACE}/gitleaks"

    sh """
        set -euo pipefail

        if command -v gitleaks &> /dev/null; then
            echo "[INFO] Gitleaks found in PATH: \$(command -v gitleaks)"
        fi

        if [ ! -f ${gitleaksBin} ]; then
            echo "[INFO] Downloading gitleaks ${gitleaksVersion}..."
            curl -fL "https://github.com/gitleaks/gitleaks/releases/download/v${gitleaksVersion}/gitleaks_${gitleaksVersion}_linux_x64.tar.gz" -o gitleaks.tar.gz
            tar -xzf gitleaks.tar.gz gitleaks
            chmod +x gitleaks
            rm -f gitleaks.tar.gz
        else
            echo "[INFO] Gitleaks binary already available at ${gitleaksBin}"
        fi

        if [ ! -x ${gitleaksBin} ]; then
            echo "[ERROR] Gitleaks binary is not executable: ${gitleaksBin}"
            exit 1
        fi

        echo "[INFO] Gitleaks version:"
        ${gitleaksBin} version
    """

    def reportPath = "${WORKSPACE}/gitleaks-report.json"
    def scanStatus = sh(
        script: """
            set -euo pipefail
            if [ -f .gitleaks.toml ]; then
                echo "[INFO] Using .gitleaks.toml"
                ${gitleaksBin} detect --source=. --no-git --redact --exit-code=1 \
                    --config=.gitleaks.toml \
                    --report-format=csv --report-path=${reportPath} \
                    --no-banner --no-color
            else
                echo "[WARN] .gitleaks.toml not found, running without config"
                ${gitleaksBin} detect --source=. --no-git --redact --exit-code=1 \
                    --report-format=csv --report-path=${reportPath} \
                    --no-banner --no-color
            fi
        """,
        returnStatus: true
    )
    if (scanStatus != 0) {
        sh """
            set -euo pipefail
            echo "[ERROR] Gitleaks found leaks. Files:"
            if [ -f ${reportPath} ]; then
                awk -F, 'NR>1{print \$3}' ${reportPath} | \
                    sed -E 's/^"//;s/"\$//' | \
                    sort -u | \
                    sed 's/^/- /'
            else
                echo "[ERROR] Report not found: ${reportPath}"
            fi
        """
        error "❌ Gitleaks scan failed: leaks found."
    }

    echo "[OK] Gitleaks passed."
}

def runCompileAndPackageStage() {
    def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []

    // Nếu common-library thay đổi, build tất cả dependents
    if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
        modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
        echo "[INFO] Common lib changed - will compile all modules"
    }

    if (modules.isEmpty()) {
        echo "[WARN] No modules to compile"
        return
    }

    // Tách frontend/backend
    def frontendModules = modules.findAll { it in ['backoffice', 'storefront'] }
    def backendModules = modules.findAll { !(it in ['backoffice', 'storefront']) }
    if (backendModules && !backendModules.contains('common-library') && fileExists('common-library/pom.xml')) {
        backendModules = (['common-library'] + backendModules).unique()
        echo "[INFO] Added common-library to backend build list"
    }

    // Build frontend modules (npm)
    frontendModules.each { module ->
        echo "[INFO] Installing dependencies for frontend: ${module}"
        dir(module) {
            sh """
                docker run --rm -v ${WORKSPACE}/${module}:/app -w /app node:20-alpine npm ci --prefer-offline --no-audit --loglevel=error
            """
        }
    }

    // Build backend modules (Maven reactor - single command)
    if (backendModules) {
        def usesJava21 = backendModules.any { it.contains('automation-ui') }
        def javaHome = usesJava21
            ? '/usr/lib/jvm/java-21-amazon-corretto'
            : '/usr/lib/jvm/java-25-amazon-corretto'

        def mvnCmd = "/opt/maven/bin/mvn clean install -DskipTests -T 1C -pl ${backendModules.join(',')} -am"
        if (env.COMMON_LIB_CHANGED == 'true') {
            mvnCmd += " -amd"
            echo "[INFO] Added -amd flag for common-library dependents"
        }

        echo "[CMD] ${mvnCmd}"

        sh """
            export JAVA_HOME=${javaHome}
            export PATH=${javaHome}/bin:\$PATH
            ${mvnCmd}
        """
    }

    echo "[OK] Compile & Package completed"
}

def runUnitTestsStage() {
    def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
    if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
        modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
    }
    if (modules.isEmpty()) {
        echo "[WARN] No modules to test"
        return
    }

    def frontendModules = modules.findAll { it in ['backoffice', 'storefront'] }
    def backendModules = modules.findAll { !(it in ['backoffice', 'storefront']) }
    if (backendModules && !backendModules.contains('common-library') && fileExists('common-library/pom.xml')) {
        backendModules = (['common-library'] + backendModules).unique()
        echo "[INFO] Added common-library to backend test list"
    }

    // Frontend unit tests
    frontendModules.each { module ->
        echo "[INFO] Running unit tests for frontend: ${module}"
        dir(module) {
            sh """
                if grep -q '"test":' package.json; then
                    docker run --rm -v ${WORKSPACE}/${module}:/app -w /app node:20-alpine npm test -- --coverage --watchAll=false --passWithNoTests
                else
                    echo "[INFO] No test script found in ${module}/package.json. Skipping."
                fi
            """
        }
    }

    // Backend unit tests (Maven Surefire)
    if (backendModules) {
        def usesJava21 = backendModules.any { it.contains('automation-ui') }
        def javaHome = usesJava21
            ? '/usr/lib/jvm/java-21-amazon-corretto'
            : '/usr/lib/jvm/java-25-amazon-corretto'

        // ⚠️ -T 1 (not 1C) for unit tests: Surefire forks 1 JVM per module already,
        // using -T 1C on top causes too many concurrent JVM processes → OOM → SIGTERM 143
        def mvnCmd = "/opt/maven/bin/mvn test -T 1 -pl ${backendModules.join(',')} -am"
        if (env.COMMON_LIB_CHANGED == 'true') mvnCmd += " -amd"

        sh """
            export JAVA_HOME=${javaHome}
            export PATH=${javaHome}/bin:\$PATH
            export MAVEN_OPTS="-Xmx2g -XX:+UseG1GC"
            ${mvnCmd}
        """
    }
}

def runIntegrationTestsStage() {
    def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
    if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
        modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
    }

    // Chỉ backend có integration tests
    def backendModules = modules.findAll {
        !(it in ['backoffice', 'storefront']) && fileExists("${it}/pom.xml")
    }

    if (!backendModules) {
        echo "[INFO] No backend modules for integration tests"
        return
    }

    def usesJava21 = backendModules.any { it.contains('automation-ui') }
    def javaHome = usesJava21
        ? '/usr/lib/jvm/java-21-amazon-corretto'
        : '/usr/lib/jvm/java-25-amazon-corretto'

    def mvnCmd = "/opt/maven/bin/mvn verify -DskipUnitTests -T 1 -DforkCount=1 -DreuseForks=true -pl ${backendModules.join(',')} -am"
    if (env.COMMON_LIB_CHANGED == 'true') mvnCmd += " -amd"
    if (env.CHANGE_ID) {
        mvnCmd += " -fae"  // Fail-At-End cho PR
        echo "[INFO] PR mode: -fae enabled"
    }

    echo "[CMD] ${mvnCmd}"

    // Integration tests có thể lâu, dùng timeout
    timeout(time: 30, unit: 'MINUTES') {
        sh """
            export JAVA_HOME=${javaHome}
            export PATH=${javaHome}/bin:\$PATH
            export MAVEN_OPTS="-Xmx3g -XX:+UseG1GC -Dtestcontainers.reuse.enable=true -Dtestcontainers.ryuk.disabled=true"
            ${mvnCmd}
        """
    }
}

def runSonarAnalysisStage() {
    def sonarRan = false
    try {
        def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
        if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
            modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
        }

        // Scan cả Java và Frontend modules
        modules.each { module ->
            def isFrontend = (module in ['backoffice', 'storefront'])

            if (!isFrontend && !fileExists("${module}/pom.xml")) {
                echo "[INFO] Skipping Sonar for non-Java and non-Frontend module: ${module}"
                return
            }

            def keyLine = PROJECT_KEYS.readLines()
                .collect { it.trim() }
                .find { it.startsWith("${module}:") }
            def projectKey = keyLine ? keyLine.substring(keyLine.indexOf(':') + 1).trim() : null
            if (!projectKey) {
                echo "[WARN] No Sonar project key for module: ${module}, skipping"
                return
            }

            echo "[INFO] Analyzing module: ${module} -> ${projectKey}"

            // Timeout cho Sonar scan
            timeout(time: 10, unit: 'MINUTES') {
                withSonarQubeEnv('sonarcloud') {
                    if (isFrontend) {
                        dir(module) {
                            sh """
                                mkdir -p ${WORKSPACE}/.scannerwork
                                docker run --rm \
                                    -v ${WORKSPACE}/${module}:/usr/src \
                                    -v ${WORKSPACE}/.scannerwork:/usr/src/.scannerwork \
                                    -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                                    -e SONAR_TOKEN=\${SONAR_AUTH_TOKEN} \
                                    sonarsource/sonar-scanner-cli \
                                    -Dsonar.projectKey=${projectKey} \
                                    -Dsonar.projectName=${module} \
                                    -Dsonar.organization=${SONAR_ORG} \
                                    -Dsonar.sources=. \
                                    -Dsonar.exclusions=node_modules/**,.next/**,out/**,coverage/**,public/** \
                                    -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                                    -Dsonar.scm.disabled=true \
                                    -Dsonar.working.directory=/usr/src/.scannerwork
                                
                                # Sửa quyền file để Jenkins có thể đọc và xoá (cleanWs)
                                docker run --rm -v ${WORKSPACE}:/app alpine chown -R \$(id -u):\$(id -g) /app/.scannerwork 2>/dev/null || true
                            """
                        }
                    } else {
                        sh """
                            /opt/maven/bin/mvn sonar:sonar \
                                -pl ${module} -am \
                                -Dsonar.projectKey=${projectKey} \
                                -Dsonar.projectName=${module} \
                                -Dsonar.organization=${SONAR_ORG} \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.sources=src/main/java \
                                -Dsonar.tests=src/test/java \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                                -Dsonar.scm.disabled=true \
                                -Dsonar.login=\${SONAR_AUTH_TOKEN}
                        """
                    }
                }
            }
            sonarRan = true
        }
    } finally {
        env.SONAR_RAN = sonarRan ? 'true' : 'false'
    }
}

def runSnykSecurityScanStage() {
    timeout(time: 10, unit: 'MINUTES') {
        def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
        if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
            modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
        }

        try {
            // --- CỰC KỲ QUAN TRỌNG ---
            // Phải install parent POM và common-library trước vì Snyk gọi mvn dependency:tree cục bộ
            // Nếu không có yas POM và common-library trong ~/.m2, mvn sẽ báo lỗi "Could not find artifact"
            // Thực hiện MỘT LẦN trước vòng lặp để tránh lỗi state do Jenkins giữ lại workspace
            sh '''
                echo "[INFO] Pre-building parent POM and common-library for Snyk resolution..."
                rm -rf ~/.m2/repository/com/yas || true
                export JAVA_HOME=/usr/lib/jvm/java-25-amazon-corretto
                export PATH=${JAVA_HOME}/bin:/opt/maven/bin:$PATH
                mvn install -N -DskipTests -q || true
                if [ -d common-library ]; then
                    mvn install -pl common-library -am -DskipTests -q || true
                fi
                if [ -d ~/.m2/repository/com/yas ]; then
                    find ~/.m2/repository/com/yas -name "*.pom" -exec sed -i 's/${revision}/1.0-SNAPSHOT/g' {} + || true
                fi
            '''

            withCredentials([string(credentialsId: 'snyk-api-token-yas', variable: 'SNYK_TOKEN')]) {
                // Track failure sau khi scan hết tất cả modules (để mọi report đều được tạo)
                def snykFailedModules = []

                modules.each { module ->
                    def scanDir = (module == 'common-library') ? '.' : module
                    def scanPath = (scanDir == '.') ? "${WORKSPACE}" : "${WORKSPACE}/${scanDir}"

                    echo "[INFO] Running Snyk scan for: ${module}"

                    def usesJava21 = module.contains('automation-ui')
                    def javaHome = usesJava21
                        ? '/usr/lib/jvm/java-21-amazon-corretto'
                        : '/usr/lib/jvm/java-25-amazon-corretto'

                    // Không dùng Docker image snyk:alpine vì image này không có Java/Maven
                    // Sử dụng Snyk CLI Native trực tiếp trên Agent để tận dụng Java/Maven có sẵn
                    def snykExitCode = sh(
                        script: """
                            export JAVA_HOME=${javaHome}
                            export PATH=\${JAVA_HOME}/bin:/opt/maven/bin:\$PATH
                            export MAVEN_OPTS="-Xmx2g"

                            if [ ! -f ./snyk-linux ]; then
                                curl -sLo ./snyk-linux https://static.snyk.io/cli/latest/snyk-linux
                                chmod +x ./snyk-linux
                            fi

                            # Cấp quyền thực thi cho mvnw để Snyk không bị lỗi EACCES (-13)
                            chmod +x ${scanPath}/mvnw 2>/dev/null || true

                            set +e
                            ./snyk-linux test --all-projects \\
                                --severity-threshold=high \\
                                --json-file-output=${scanPath}/snyk-report.json \\
                                ${scanPath} \\
                                -- -U -Drevision=1.0-SNAPSHOT
                            exit \$?
                        """,
                        returnStatus: true
                    )

                    // Archive Snyk report ngay lập tức trước khi throw error
                    if (fileExists("${scanDir}/snyk-report.json")) {
                        echo "📄 [REPORT] Snyk JSON Report saved at: ${scanDir}/snyk-report.json"
                        archiveArtifacts artifacts: "${scanDir}/snyk-report.json",
                                     allowEmptyArchive: true,
                                     fingerprint: true
                        
                        try {
                            echo "[INFO] Generating HTML report for ${module}..."
                            sh """
                                docker run --rm -v ${WORKSPACE}:/app -w /app node:20-alpine \\
                                npx -y snyk-to-html -i ${scanDir}/snyk-report.json -o ${scanDir}/snyk-report.html
                            """
                            
                            if (fileExists("${scanDir}/snyk-report.html")) {
                                publishHTML([
                                    allowMissing: true,
                                    alwaysLinkToLastBuild: true,
                                    keepAll: true,
                                    reportDir: "${scanDir}",
                                    reportFiles: 'snyk-report.html',
                                    reportName: "Snyk Report (${module})",
                                    reportTitles: "Security Scan Report - ${module}"
                                ])
                            }

                            // In log chi tiết các lỗi Critical
                            def report = readJSON file: "${scanPath}/snyk-report.json"
                            def criticalIssues = []
                            
                            // Snyk JSON có thể là Object hoặc Array tùy theo mode --all-projects
                            def vulnerabilities = (report instanceof List) 
                                ? report.collectMany { it.vulnerabilities ?: [] } 
                                : (report.vulnerabilities ?: [])

                            criticalIssues = vulnerabilities.findAll { it.severity?.toLowerCase() == 'critical' }

                            if (criticalIssues) {
                                echo "🔥 [CRITICAL] FOUND ${criticalIssues.size()} CRITICAL VULNERABILITIES IN ${module}:"
                                criticalIssues.each { issue ->
                                    echo "   - ID: ${issue.id}"
                                    echo "     Title: ${issue.title}"
                                    echo "     Package: ${issue.packageName} (Version: ${issue.version})"
                                    echo "     Path: ${issue.from?.join(' > ') ?: 'N/A'}"
                                    echo "     URL: ${issue.references?.getAt(0)?.url ?: 'N/A'}"
                                    echo "   ---------------------------------------"
                                }
                            }
                        } catch (Exception e) {
                            echo "[WARN] Error generating/publishing Snyk HTML report: ${e.message}"
                        }
                    }

                    // Xử lý Exit Code của Snyk theo Best-Practice 
                    switch(snykExitCode) {
                        case 0:
                            echo "✅ [SUCCESS] Snyk scan passed! No high-severity vulnerabilities found in module: ${module}."
                            break

                        case 1:
                            echo "⚠️ [SECURITY WARNING] Snyk found HIGH-SEVERITY VULNERABILITIES in module: ${module}!"

                            // Kiểm tra xem đích đến cuối cùng của code có phải là các nhánh Production không
                            def isTargetingProd = (isPR && (env.CHANGE_TARGET == 'main' || env.CHANGE_TARGET == 'master')) || 
                                                  (env.BRANCH_NAME == 'main' || env.BRANCH_NAME == 'master')

                            if (isTargetingProd) {
                                echo "❌ [POLICY] Tight security gate enforced for PR/Production branches."
                                echo "👉 ACTION REQUIRED: Please view the Snyk report artifact and upgrade your libraries immediately to fix the issues."
                                // Ghi nhận module bị fail, tiếp tục scan module còn lại để tạo đủ reports
                                snykFailedModules << module
                            } else {
                                echo "ℹ️ [POLICY] Feature branch detected. This warning is non-blocking (Continue build). However, please fix these vulnerabilities before opening a PR."
                            }
                            break

                        case 2:
                            echo "🔥 [SNYK SYSTEM ERROR] Snyk CLI failed due to environment/network issues (e.g., Token expired, 403 Forbidden, rate limit)."
                            echo "👉 TIP: Check your SNYK_TOKEN credentials or run the command locally with 'snyk test -d' to view debug logs."
                            // Lỗi hệ thống: không liên quan đến vulnerabilities, fail ngay
                            error("❌ Snyk system error in module: ${module} (exit code 2). Check SNYK_TOKEN and network.")

                        case 3:
                            echo "ℹ️ [INFO] Snyk did not find any supported package manager files (e.g., package.json, pom.xml) to scan in ${module}. Skipping gracefully."
                            break

                        default:
                            echo "❌ [UNKNOWN ERROR] Snyk scan terminated with an unexpected status code: ${snykExitCode}. Please contact DevOps team."
                            error("❌ Snyk unknown exit code ${snykExitCode} in module: ${module}.")
                    }
                }

                // Sau khi scan xong tất cả modules, fail pipeline nếu có module vi phạm
                if (snykFailedModules) {
                    error("❌ Snyk security gate FAILED for PR targeting production. Vulnerable modules: ${snykFailedModules.join(', ')}. Fix all HIGH/CRITICAL issues before merging.")
                }
            }
        } catch (Exception e) {
            def errMsg = e.message ?: ""
            if (errMsg.toLowerCase().contains("credential") || errMsg.toLowerCase().contains("could not find")) {
                error "❌ SNYK_TOKEN missing. Check credential 'snyk-api-token-yas' (Kind: Secret text). Error: ${errMsg}"
            } else {
                error "❌ Snyk scan failed (vulnerabilities found or other error): ${errMsg}"
            }
        }
    }
}

def resolveCommitInfo(Map scmVars = null, boolean failIfUnknown = false) {
    def fullCommit = scmVars?.GIT_COMMIT?.trim() ?: env.GIT_COMMIT?.trim()
    if (!fullCommit) {
        fullCommit = sh(
            script: "git rev-parse HEAD 2>/dev/null || true",
            returnStdout: true,
            label: 'resolve-commit-full'
        ).trim()
    }
    if (fullCommit) {
        env.GIT_COMMIT = fullCommit
    }

    def shortCommit = (fullCommit?.length() >= 7) ? fullCommit.take(7) : env.GIT_COMMIT_SHORT?.trim()
    if (!shortCommit || shortCommit == 'unknown' || !shortCommit.matches(/^[a-fA-F0-9]{7}$/)) {
        shortCommit = sh(
            script: "git rev-parse --short=7 HEAD 2>/dev/null || true",
            returnStdout: true,
            label: 'resolve-commit-short'
        ).trim()
    }
    env.GIT_COMMIT_SHORT = (shortCommit?.matches(/^[a-fA-F0-9]{7}$/)) ? shortCommit.toLowerCase() : 'unknown'

    if (failIfUnknown && env.GIT_COMMIT_SHORT == 'unknown') {
        error "Cannot determine valid commit SHA for image tagging. Ensure checkout completed and git metadata is available."
    }
}

def runBuildAndPushStage() {
    resolveCommitInfo(null, true)
    def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []

    modules.each { module ->
        def dockerfilePath = "${module}/Dockerfile"
        if (!fileExists(dockerfilePath)) {
            echo "[BUILD][${module}] SKIP: No Dockerfile"
            return
        }
        
        def commitSha = env.GIT_COMMIT_SHORT?.trim()?.toLowerCase()
        def imageRefs = validateAndBuildImageRef(module, commitSha, env.BRANCH_NAME)
        echo "[BUILD][${module}] START: Building ${imageRefs.immutable}"
        
        try {
            // ❌ KHÔNG retry docker build (lỗi code/Dockerfile → fail ngay)
            dir(module) {
                sh """
                    echo "[BUILD][${module}] STEP: docker build"
                    docker build -t ${imageRefs.immutable} \\
                      --label "org.opencontainers.image.revision=${env.GIT_COMMIT ?: ''}" \\
                      --label "org.opencontainers.image.created=\$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \\
                      --label "org.opencontainers.image.source=${env.BUILD_URL ?: ''}" \\
                      --progress=plain .
                """
            }
            
            // ✅ CHỈ retry docker push (network transient)
            echo "[PUSH][${module}] START: Pushing to ${env.REGISTRY}"
            retry(2) {
                sh """
                    echo "[PUSH][${module}] STEP: docker push attempt"
                    docker push ${imageRefs.immutable}
                """
            }
            
            def immutableDigest = sh(
                script: "docker inspect ${imageRefs.immutable} --format='{{index .RepoDigests 0}}'",
                returnStdout: true
            ).trim()
            
            // Mutable tag
            if (imageRefs.mutable) {
                try {
                    sh """
                        docker tag ${imageRefs.immutable} ${imageRefs.mutable}
                        docker push ${imageRefs.mutable}
                    """
                    echo "[BUILD][${module}] ✅ Mutable tag pushed: ${imageRefs.mutable}"
                } catch (Exception e) {
                    echo "[WARN][${module}] Mutable tag push failed: ${e.message}. Immutable tag is still valid."
                }
            }

            // Optional: verify remote digest if skopeo is available
            def hasSkopeo = sh(script: "command -v skopeo >/dev/null 2>&1", returnStatus: true) == 0
            if (hasSkopeo) {
                def remoteDigest = sh(
                    script: "skopeo inspect docker://${imageRefs.immutable} --format '{{.Digest}}'",
                    returnStdout: true,
                    label: 'verify-remote-digest'
                ).trim().replace('sha256:', '')
                def localDigest = immutableDigest.replace('sha256:', '').split('@')[1]
                if (remoteDigest != localDigest) {
                    error "Digest mismatch! Local: ${localDigest}, Remote: ${remoteDigest}. Possible registry corruption."
                }
            } else {
                echo "[WARN][${module}] skopeo not found; skipping remote digest verification."
            }

            def manifest = [
                module: module,
                commit: env.GIT_COMMIT,
                branch: env.BRANCH_NAME,
                immutableTag: imageRefs.immutable,
                mutableTag: imageRefs.mutable,
                digest: immutableDigest,
                buildUrl: env.BUILD_URL,
                timestamp: new Date().getTime()
            ]
            writeJSON file: "${module}/image-manifest.json", json: manifest
            archiveArtifacts artifacts: "${module}/image-manifest.json", allowEmptyArchive: true

            echo "[BUILD][${module}] ✅ SUCCESS: Pushed ${imageRefs.immutable}"
            
        } catch (Exception e) {
            echo "[BUILD][${module}] ❌ FAILED: ${e.message}"
            // Fail pipeline cho PR, collect error cho feature branch
            if (env.CHANGE_ID) error "Build/Push failed for ${module}"
            else currentBuild.result = 'UNSTABLE'
        }
    }

    // Dọn disk sau build
    sh "docker image prune -f --filter 'until=24h' 2>/dev/null || true"
}

def sanitizeRegistry(String raw) {
    def cleaned = raw?.trim()
    if (!cleaned?.matches(/^[\w\.-]+:\d+$/)) {
        error "Invalid registry format: ${raw}"
    }
    return cleaned
}

def validateAndBuildImageRef(String module, String commitSha, String branch = null) {
    if (!commitSha?.matches(/^[a-f0-9]{7}$/)) {
        error "Invalid commit SHA for tagging: ${commitSha}. Expected 7-char hex."
    }

    def registry = sanitizeRegistry(env.REGISTRY)
    def immutableTag = "yas-${module}:${commitSha}"
    def immutableRef = "${registry}/${immutableTag}"

    def mutableRef = null
    if (branch) {
        def sanitizedBranch = branch.replaceAll('[^\\w\\.-]', '-').toLowerCase()
        def mutableTag = "yas-${module}:${sanitizedBranch}-${commitSha}"
        mutableRef = "${registry}/${mutableTag}"
    }

    echo "[TAG-VALIDATED] Immutable: ${immutableRef}${mutableRef ? ', Mutable: ' + mutableRef : ''}"
    return [immutable: immutableRef, mutable: mutableRef]
}

def runSaveMavenCacheStage() {
    try {
        // Chỉ save nếu build thành công
        if (currentBuild.result in [null, 'SUCCESS', 'UNSTABLE']) {
            if (!env.BRANCH_NAME?.trim()) {
                env.BRANCH_NAME = env.CHANGE_BRANCH?.trim()
                    ?: (env.GIT_BRANCH ? env.GIT_BRANCH.replaceFirst(/^origin\//, '').trim() : null)
                    ?: 'detached'
                env.BRANCH_NAME = env.BRANCH_NAME.replaceAll('/', '-')
                echo "[WARN] BRANCH_NAME not set, using '${env.BRANCH_NAME}'"
            }
            if (!env.CACHE_KEY?.trim()) {
                env.CACHE_KEY = sh(
                    script: """
                        find . '(' -name 'pom.xml' -o -name 'package.json' ')' -exec md5sum {} + 2>/dev/null |
                        md5sum | awk '{print \$1}'
                    """,
                    returnStdout: true,
                    label: 'generate-cache-key-fallback'
                ).trim()
            }
            if (!env.CACHE_KEY?.trim()) {
                echo "[WARN] Cache key is empty, skipping cache save to S3"
                return
            }
            sh """
                echo "[INFO] Creating cache archive for branch ${BRANCH_NAME}"
                tar -czf cache.tar.gz -C ~/.m2 repository 2>/dev/null || true

                if [ -f cache.tar.gz ]; then
                    # Upload với retry
                    aws s3 cp cache.tar.gz \
                        s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz \
                        --storage-class STANDARD_IA

                    # Tạo và upload checksum
                    md5sum cache.tar.gz > cache.tar.gz.md5
                    aws s3 cp cache.tar.gz.md5 \
                        s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz.md5

                    echo "[OK] Cache saved to S3"
                else
                    echo "[WARN] Cache archive empty, skipping upload"
                fi

                rm -f cache.tar.gz cache.tar.gz.md5
            """
        } else {
            echo "[INFO] Build result: ${currentBuild.result}. Skipping cache save."
        }

    } catch (Exception e) {
        echo "[WARN] Failed to save cache: ${e.message}. Continuing."
        // Cache save fail không nên fail pipeline
    }
}

/**
 * Safe file existence check với error handling
 */
def safeFileExists(String path) {
    try {
        return fileExists(path)
    } catch (Exception e) {
        echo "[WARN] Could not check file ${path}: ${e.message}"
        return false
    }
}

/**
 * Safe sh execution với logging
 */
def safeSh(String script, String label = 'unnamed') {
    try {
        echo "[CMD:${label}] ${script.take(200)}${script.length() > 200 ? '...' : ''}"
        return sh(script: script, returnStdout: true, label: label)
    } catch (Exception e) {
        echo "[ERROR:${label}] Command failed: ${e.message}"
        throw e
    }
}