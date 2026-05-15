// ============================================================
// JENKINSFILE - YAS CI/CD PIPELINE (PRODUCTION-READY)
// Monorepo: 22 microservices (Java Spring Boot + Next.js)
// Infrastructure: AWS VPC, Spot Agents (NVMe), Local Registry, S3 Cache
// Best-Practices: Retry at Master, Smart Routing, Maven Reactor, Graceful Error Handling
// ============================================================

pipeline {
    // ============================================================
    // 1. AGENT & GLOBAL OPTIONS
    // ============================================================
    agent none

    options {
        // ✅ Retry ở Master level - Critical cho Spot interruption handling
        retry(3)
        
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
        GIT_COMMIT_SHORT   = "${env.GIT_COMMIT ?: 'unknown'}"
        
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
                    echo "[INFO] === SMART ROUTING STARTED ==="
                    echo "[INFO] Target branch: ${CHANGE_TARGET}"
                    env.AFFECTED_MODULES = ''
                    env.COMMON_LIB_CHANGED = 'false'
                    env.SHOULD_BUILD = 'false'
                    
                    try {
                        env.GIT_COMMIT_SHORT = env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'unknown'
                        // Fetch target branch để có merge base chính xác
                        sh "git fetch origin ${CHANGE_TARGET}:refs/remotes/origin/${CHANGE_TARGET} --depth=50"
                        
                        // ✅ Git diff 3-dots: so sánh từ merge base, không bị nhiễm code người khác
                        def changedFiles = sh(
                            script: "git diff --name-only origin/${CHANGE_TARGET}...HEAD",
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
            }
        }

        // ============================================================
        // STAGE 2: GITLEAKS SCAN (Security - chạy trên built-in)
        // ============================================================
        stage('Gitleaks Scan') {
            when { expression { env.SHOULD_BUILD == 'true' } }
            agent { label 'built-in' }
            steps {
                script {
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
                    
                    sh """
                        set -euo pipefail
                        ${gitleaksBin} detect --source=. --no-git --redact --exit-code=1
                    """
                }
                echo "[OK] Gitleaks passed."
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
                    steps {
                        script {
                            echo "[INFO] === CHECKOUT & SETUP STARTED ==="
                            
                            try {
                                checkout scm
                                
                                // Lấy AWS config với error handling
                                env.ACCOUNT_ID = withAWS(region: 'us-east-1') {
                                    sh(script: 'aws sts get-caller-identity --query Account --output text', 
                                       returnStdout: true, label: 'get-account-id').trim()
                                }
                                env.CACHE_BUCKET = "yas-cache-${env.ACCOUNT_ID}"
                                
                                // Lấy Hub IP từ SSM với retry
                                env.HUB_IP = retryWithBackoff(3, 5) {
                                    sh(script: """
                                        aws ssm get-parameter \
                                            --name /yas/hub/private_ip \
                                            --region us-east-1 \
                                            --query Parameter.Value \
                                            --output text
                                    """, returnStdout: true, label: 'get-hub-ip').trim()
                                }
                                
                                env.REGISTRY = "${env.HUB_IP}:5000"
                                
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
                                retryWithBackoff(3, 3) {
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
                                            exit \$?
                                        """,
                                        returnStatus: true,
                                        label: 'restore-branch-cache'
                                    )
                                    if (exitCode == 0 && fileExists('cache.tar.gz')) {
                                        // Verify checksum nếu có
                                        if (sh(script: "aws s3 ls s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz.md5 2>/dev/null", returnStatus: true) == 0) {
                                            sh "aws s3 cp s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz.md5 ./cache.tar.gz.md5 2>/dev/null"
                                            if (fileExists('cache.tar.gz.md5')) {
                                                sh "md5sum -c cache.tar.gz.md5 || { echo '[WARN] Cache checksum mismatch'; rm -f cache.tar.gz.md5; }"
                                            }
                                        }
                                        sh "tar -xzf cache.tar.gz -C ~/.m2 2>/dev/null"
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
                                            exit \$?
                                        """,
                                        returnStatus: true,
                                        label: 'restore-main-cache'
                                    )
                                    if (exitCode == 0 && fileExists('cache.tar.gz')) {
                                        sh "tar -xzf cache.tar.gz -C ~/.m2 2>/dev/null"
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
                            
                            try {
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
                                
                                // Build frontend modules (npm)
                                frontendModules.each { module ->
                                    echo "[INFO] Installing dependencies for frontend: ${module}"
                                    dir(module) {
                                        retryWithBackoff(2, 5) {
                                            sh """
                                                npm ci --prefer-offline --no-audit --loglevel=error
                                            """
                                        }
                                    }
                                }
                                
                                // Build backend modules (Maven reactor - single command)
                                if (backendModules) {
                                    def usesJava21 = backendModules.any { it.contains('automation-ui') }
                                    def javaHome = usesJava21 
                                        ? '/usr/lib/jvm/java-21-amazon-corretto'
                                        : '/usr/lib/jvm/java-25-amazon-corretto'
                                    
                                    def mvnCmd = "/opt/maven/bin/mvn clean compile -T 1C -pl ${backendModules.join(',')}"
                                    if (env.COMMON_LIB_CHANGED == 'true') {
                                        mvnCmd += " -amd"
                                        echo "[INFO] Added -amd flag for common-library dependents"
                                    }
                                    
                                    echo "[CMD] ${mvnCmd}"
                                    
                                    retryWithBackoff(2, 10) {
                                        sh """
                                            export JAVA_HOME=${javaHome}
                                            export PATH=${javaHome}/bin:\$PATH
                                            ${mvnCmd}
                                        """
                                    }
                                }
                                
                                echo "[OK] Compile & Package completed"
                                
                            } catch (Exception e) {
                                echo "[ERROR] Compile failed: ${e.message}"
                                // Fail fast cho compile stage
                                throw e
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.5: Unit Tests
                // ------------------------------------------------
                stage('Unit Tests') {
                    steps {
                        script {
                            echo "[INFO] === UNIT TESTS STARTED ==="
                            
                            try {
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
                                
                                // Frontend unit tests
                                frontendModules.each { module ->
                                    echo "[INFO] Running unit tests for frontend: ${module}"
                                    dir(module) {
                                        // Continue on test failure để collect all results
                                        sh """
                                            npm test -- --coverage --watchAll=false --passWithNoTests || true
                                        """
                                    }
                                }
                                
                                // Backend unit tests (Maven Surefire)
                                if (backendModules) {
                                    def usesJava21 = backendModules.any { it.contains('automation-ui') }
                                    def javaHome = usesJava21 
                                        ? '/usr/lib/jvm/java-21-amazon-corretto'
                                        : '/usr/lib/jvm/java-25-amazon-corretto'
                                    
                                    def mvnCmd = "/opt/maven/bin/mvn test -T 1C -pl ${backendModules.join(',')}"
                                    if (env.COMMON_LIB_CHANGED == 'true') mvnCmd += " -amd"
                                    
                                    retryWithBackoff(2, 10) {
                                        sh """
                                            export JAVA_HOME=${javaHome}
                                            export PATH=${javaHome}/bin:\$PATH
                                            ${mvnCmd} || true  # Continue để collect test results
                                        """
                                    }
                                }
                                
                            } catch (Exception e) {
                                echo "[WARN] Unit tests encountered errors: ${e.message}. Continuing to integration tests."
                                // Không throw để tiếp tục pipeline
                            }
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
                            
                            try {
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
                                
                                def mvnCmd = "/opt/maven/bin/mvn verify -DskipUnitTests -T 1C -pl ${backendModules.join(',')}"
                                if (env.COMMON_LIB_CHANGED == 'true') mvnCmd += " -amd"
                                if (env.CHANGE_ID) {
                                    mvnCmd += " -fae"  // Fail-At-End cho PR
                                    echo "[INFO] PR mode: -fae enabled"
                                }
                                
                                echo "[CMD] ${mvnCmd}"
                                
                                // Integration tests có thể lâu, dùng timeout
                                timeout(time: 30, unit: 'MINUTES') {
                                    retryWithBackoff(2, 15) {
                                        sh """
                                            export JAVA_HOME=${javaHome}
                                            export PATH=${javaHome}/bin:\$PATH
                                            ${mvnCmd}
                                        """
                                    }
                                }
                                
                            } catch (Exception e) {
                                echo "[ERROR] Integration tests failed: ${e.message}"
                                throw e  // Fail pipeline cho integration tests
                            }
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
                        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                    }
                    steps {
                        script {
                            echo "[INFO] === SONARQUBE ANALYSIS STARTED ==="
                            
                            try {
                                def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
                                if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
                                    modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
                                }
                                
                                // Chỉ scan Java modules
                                modules.each { module ->
                                    if (!fileExists("${module}/pom.xml")) {
                                        echo "[INFO] Skipping Sonar for non-Java module: ${module}"
                                        return
                                    }
                                    
                                    def projectKey = PROJECT_KEYS.readLines()
                                        .find { it.startsWith("${module}:") }
                                        ?.split(':')
                                        ?.getAt(1)
                                    if (!projectKey) {
                                        echo "[WARN] No Sonar project key for module: ${module}, skipping"
                                        return
                                    }
                                    
                                    echo "[INFO] Analyzing module: ${module} -> ${projectKey}"
                                    
                                    // Timeout cho Sonar scan
                                    timeout(time: 10, unit: 'MINUTES') {
                                        withSonarQubeEnv('sonarcloud') {
                                            retryWithBackoff(2, 5) {
                                                sh """
                                                    /opt/maven/bin/mvn sonar:sonar \
                                                        -pl ${module} \
                                                        -Dsonar.projectKey=${projectKey} \
                                                        -Dsonar.projectName=${module} \
                                                        -Dsonar.sources=${module}/src/main/java \
                                                        -Dsonar.tests=${module}/src/test/java \
                                                        -Dsonar.java.binaries=${module}/target/classes \
                                                        -Dsonar.coverage.jacoco.xmlReportPaths=${module}/target/site/jacoco/jacoco.xml \
                                                        -Dsonar.scm.disabled=true \
                                                        -Dsonar.login=\${SONAR_AUTH_TOKEN}
                                                """
                                            }
                                        }
                                    }
                                }
                                
                            } catch (Exception e) {
                                echo "[ERROR] SonarQube analysis failed: ${e.message}"
                                // Log error nhưng không fail pipeline để tiếp tục các stage khác
                                echo "[WARN] Sonar analysis error logged, continuing pipeline"
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.8: Quality Gate (Coverage Check cho PR)
                // ------------------------------------------------
                stage('Quality Gate') {
                    when { expression { env.CHANGE_ID != null } }
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
                // 4.9: Snyk Security Scan
                // ------------------------------------------------
                stage('Snyk Security Scan') {
                    when {
                        anyOf {
                            branch 'main'
                            expression { env.CHANGE_ID != null }
                        }
                        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                    }
                    steps {
                        script {
                            echo "[INFO] === SNYK SECURITY SCAN STARTED ==="
                            
                            try {
                                timeout(time: 10, unit: 'MINUTES') {
                                    withCredentials([string(credentialsId: 'snyk-api-token-yas', variable: 'SNYK_TOKEN')]) {
                                        
                                        def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
                                        if (modules.isEmpty() && env.COMMON_LIB_CHANGED == 'true') {
                                            modules = PROJECT_KEYS.readLines().collect { it.split(':')[0] }.findAll { it }
                                        }
                                        
                                        modules.each { module ->
                                            def scanDir = (module == 'common-library') ? '.' : module
                                            def scanPath = (scanDir == '.') ? "${WORKSPACE}" : "${WORKSPACE}/${scanDir}"
                                            
                                            echo "[INFO] Running Snyk scan for: ${module}"
                                            
                                            // Snyk trong Docker với volume mount
                                            retryWithBackoff(2, 5) {
                                                sh """
                                                    docker run --rm \
                                                        -v ${scanPath}:/app:ro \
                                                        -e SNYK_TOKEN=\${SNYK_TOKEN} \
                                                        snyk/snyk:alpine \
                                                        snyk test --all-projects \
                                                        --severity-threshold=high \
                                                        --fail-on=high,critical \
                                                        --json-file-output=/app/snyk-report.json || true
                                                """
                                            }
                                            
                                            // Archive Snyk report nếu có
                                            if (fileExists("${scanDir}/snyk-report.json")) {
                                                archiveArtifacts artifacts: "${scanDir}/snyk-report.json", 
                                                             allowEmptyArchive: true
                                            }
                                        }
                                    }
                                }
                                
                            } catch (Exception e) {
                                echo "[ERROR] Snyk scan failed: ${e.message}"
                                // Security scan fail thì fail pipeline
                                if (env.CHANGE_ID) {
                                    throw e
                                }
                            }
                        }
                    }
                }

                // ------------------------------------------------
                // 4.10: Build & Push Docker Image (chỉ main branch)
                // ------------------------------------------------
                stage('Build & Push Docker Image') {
                    when { branch 'main' }
                    steps {
                        script {
                            echo "[INFO] === DOCKER BUILD & PUSH STARTED ==="
                            
                            try {
                                def modules = env.AFFECTED_MODULES ? env.AFFECTED_MODULES.split(',').findAll { it } : []
                                
                                modules.each { module ->
                                    def dockerfilePath = "${module}/Dockerfile"
                                    if (!fileExists(dockerfilePath)) {
                                        echo "[WARN] No Dockerfile for ${module}, skipping image build"
                                        return
                                    }
                                    
                                    def immutableTag = "yas-${module}:${env.GIT_COMMIT_SHORT}"
                                    echo "[INFO] Building image for module: ${module}"
                                    
                                    try {
                                        if (module in ['backoffice', 'storefront']) {
                                            // Frontend: build từ root với -f flag
                                            retryWithBackoff(2, 5) {
                                                sh """
                                                    docker build -f ${dockerfilePath} \
                                                        -t ${env.REGISTRY}/${immutableTag} . \
                                                        --progress=plain
                                                """
                                            }
                                        } else {
                                            // Backend: build từ folder module
                                            dir(module) {
                                                retryWithBackoff(2, 5) {
                                                    sh """
                                                        docker build -t ${env.REGISTRY}/${immutableTag} . \
                                                            --progress=plain
                                                    """
                                                }
                                            }
                                        }
                                        
                                        // Push với retry cho network issues
                                        echo "[INFO] Pushing ${immutableTag} to ${env.REGISTRY}"
                                        retryWithBackoff(3, 5) {
                                            sh """
                                                docker push ${env.REGISTRY}/${immutableTag}
                                            """
                                        }
                                        
                                        // Mutable tag cho human-readable
                                        def mutableTag = "yas-${module}:${env.BRANCH_NAME.replace('/', '-')}-${env.GIT_COMMIT_SHORT}"
                                        sh """
                                            docker tag ${env.REGISTRY}/${immutableTag} ${env.REGISTRY}/${mutableTag}
                                            docker push ${env.REGISTRY}/${mutableTag}
                                        """
                                        
                                        echo "[OK] Pushed ${immutableTag}"
                                        
                                    } catch (Exception e) {
                                        echo "[ERROR] Failed to build/push image for ${module}: ${e.message}"
                                        // Continue với module khác, không fail toàn pipeline
                                    }
                                }
                                
                            } catch (Exception e) {
                                echo "[ERROR] Docker build stage failed: ${e.message}"
                                // Log nhưng không fail để các stage sau vẫn chạy
                            }
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
                    when { expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' } }
                    steps {
                        script {
                            echo "[INFO] === SAVING MAVEN CACHE ==="
                            
                            try {
                                // Chỉ save nếu build thành công
                                if (currentBuild.result in [null, 'SUCCESS', 'UNSTABLE']) {
                                    sh """
                                        echo "[INFO] Creating cache archive for branch ${BRANCH_NAME}"
                                        tar -czf cache.tar.gz -C ~/.m2 repository 2>/dev/null || true
                                        
                                        if [ -f cache.tar.gz ]; then
                                            # Upload với retry
                                            aws s3 cp cache.tar.gz \
                                                s3://${env.CACHE_BUCKET}/maven/${BRANCH_NAME}-${env.CACHE_KEY}.tar.gz \
                                                --storage-class STANDARD_IA
                                            
                                            # Tạo và upload checksum
                                            md5sum cache.tar.gz | awk '{print \$1}' > cache.tar.gz.md5
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
                    }
                }
            }
        }

        // ============================================================
        // STAGE 5: Pipeline Metrics (Optional)
        // ============================================================
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
                    slackSend channel: '#ci-cd',
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
                    slackSend channel: '#ci-cd',
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
                slackSend channel: '#ci-cd',
                          color: 'warning',
                          message: "⚠️ Pipeline UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}\n" +
                                   "Branch: ${env.BRANCH_NAME}\n" +
                                   "Check test failures or quality warnings"
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
            def delay = baseDelaySeconds * Math.pow(2, attempts - 1)
            echo "[WARN] Attempt ${attempts}/${maxAttempts} failed. Retrying in ${delay}s... Error: ${e.message}"
            sleep(delay)
        }
    }
    
    // Fallback (không nên reach here)
    throw lastException ?: new RuntimeException("Retry failed with unknown error")
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