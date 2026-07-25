pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 45, unit: 'MINUTES')
    }

    parameters {
        string(
            name: 'REPO_URL',
            defaultValue: '',
            description: 'Optional repository override; leave empty to build the repository containing this Jenkinsfile'
        )

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch to build'
        )

        string(
            name: 'SONAR_PROJECT_KEY',
            defaultValue: 'python-fastapi-boilerplate',
            description: 'SonarQube project key'
        )

        booleanParam(
            name: 'FAIL_ON_IMAGE_VULNERABILITIES',
            defaultValue: true,
            description: 'Fail when Trivy finds a fixed HIGH or CRITICAL vulnerability'
        )

        string(
            name: 'DOCKERHUB_REPOSITORY',
            defaultValue: 'python-fastapi-boilerplate',
            description: 'Docker Hub repository name or namespace/name'
        )

        string(
            name: 'EMAIL_TO',
            defaultValue: '',
            description: 'Optional notification address; leave empty to disable email'
        )
    }

    environment {
        LOCAL_IMAGE_NAME = "python-fastapi-boilerplate:${BUILD_NUMBER}"
        SMOKE_CONTAINER = "fastapi-ci-${BUILD_TAG}"
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        PYTHONDONTWRITEBYTECODE = '1'
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
    }

    stages {
        stage('Validate Parameters') {
            agent none

            steps {
                script {
                    if (!params.GIT_BRANCH?.trim()) {
                        error('GIT_BRANCH must not be empty')
                    }

                    if (!params.SONAR_PROJECT_KEY.matches('[A-Za-z0-9_.:-]+')) {
                        error('SONAR_PROJECT_KEY contains unsupported characters')
                    }

                    if (!params.DOCKERHUB_REPOSITORY.matches(
                        '[a-z0-9]+(?:[._-][a-z0-9]+)*(?:/[a-z0-9]+(?:[._-][a-z0-9]+)*)*'
                    )) {
                        error(
                            'DOCKERHUB_REPOSITORY must be a lowercase repository name ' +
                            'or namespace/repository'
                        )
                    }

                    if (params.EMAIL_TO?.trim() &&
                        !params.EMAIL_TO.trim().matches('[^\\s@]+@[^\\s@]+\\.[^\\s@]+')) {
                        error('EMAIL_TO must be empty or contain one valid email address')
                    }
                }
            }
        }

        /*
         * One agent allocation is retained for all source-code CI stages.
         * Required on exec_node_1: Git, Python 3.11+, venv and the configured
         * Jenkins SonarScanner tool.
         */
        stage('Python CI') {
            agent {
                label 'exec_node_1'
            }

            stages {
                stage('Checkout') {
                    steps {
                        deleteDir()

                        script {
                            if (params.REPO_URL?.trim()) {
                                git(
                                    url: params.REPO_URL.trim(),
                                    branch: params.GIT_BRANCH.trim()
                                )
                            } else {
                                checkout scm
                            }

                            env.BUILT_REPOSITORY = sh(
                                script: 'git config --get remote.origin.url',
                                returnStdout: true
                            ).trim()
                        }

                        sh '''
                            git --no-pager log -1 \
                              --pretty=format:'Commit: %H%nAuthor: %an%nMessage: %s%n'
                        '''

                        /*
                         * Stash before creating .venv and reports so only
                         * repository source is transferred to exec_node_2.
                         */
                        stash(
                            name: 'application-source',
                            includes: '**',
                            useDefaultExcludes: true
                        )
                    }
                }

                stage('Install Dependencies') {
                    steps {
                        sh '''#!/usr/bin/env bash
set -euo pipefail

python3 --version
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip install pytest-cov ruff

mkdir -p reports
.venv/bin/python -m pip freeze > reports/installed-packages.txt
'''
                    }
                }

                stage('Python Syntax Check') {
                    steps {
                        sh '''
                            .venv/bin/python -m compileall \
                              -q \
                              -x '(^|/)(\\.venv|\\.git|reports)/' \
                              .
                        '''
                    }
                }

                stage('Lint') {
                    steps {
                        sh '''#!/usr/bin/env bash
set -o pipefail

.venv/bin/ruff check main.py test_main.py \
  --output-format=full | tee reports/ruff-results.txt
'''
                    }
                }

                stage('Unit Tests and Coverage') {
                    steps {
                        sh '''
                            .venv/bin/pytest \
                              --verbose \
                              --junitxml=reports/pytest-results.xml \
                              --cov=main \
                              --cov-report=term-missing \
                              --cov-report=xml:reports/coverage.xml \
                              test_main.py
                        '''
                    }

                    post {
                        always {
                            junit(
                                testResults: 'reports/pytest-results.xml',
                                allowEmptyResults: true
                            )
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        script {
                            def scannerHome = tool 'SonarScanner'

                            withEnv(["SCANNER_HOME=${scannerHome}"]) {
                                withSonarQubeEnv('SonarQube') {
                                    sh '''
                                        "$SCANNER_HOME/bin/sonar-scanner" \
                                          -Dsonar.projectKey="$SONAR_PROJECT_KEY" \
                                          -Dsonar.projectName="$SONAR_PROJECT_KEY" \
                                          -Dsonar.sources=main.py \
                                          -Dsonar.tests=test_main.py \
                                          -Dsonar.python.version=3.11 \
                                          -Dsonar.python.coverage.reportPaths=reports/coverage.xml \
                                          -Dsonar.sourceEncoding=UTF-8
                                    '''
                                }
                            }
                        }
                    }
                }
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'reports/**',
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
            }
        }

        stage('SonarQube Quality Gate') {
            agent none

            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /*
         * Required on exec_node_2: Docker CLI and access to a Docker daemon.
         * Hadolint and Trivy run as containers, so they do not need to be
         * installed directly on the Jenkins agent.
         */
        stage('Container CI') {
            agent {
                label 'exec_node_2'
            }

            stages {
                stage('Prepare Docker Workspace') {
                    steps {
                        deleteDir()
                        unstash 'application-source'
                        sh 'mkdir -p reports'
                    }
                }

                stage('Lint Dockerfile') {
                    steps {
                        sh '''#!/usr/bin/env bash
set -o pipefail

docker run --rm -i \
  hadolint/hadolint:v2.12.0-debian \
  --failure-threshold error - \
  < Dockerfile | tee reports/hadolint-results.txt
'''
                    }
                }

                stage('Build Docker Image') {
                    steps {
                        sh '''
                            docker build \
                              --pull \
                              --label "org.opencontainers.image.revision=${GIT_COMMIT}" \
                              --label "ci.jenkins.build=${BUILD_URL}" \
                              --tag "$LOCAL_IMAGE_NAME" \
                              .

                            docker image inspect "$LOCAL_IMAGE_NAME" \
                              > reports/docker-image-inspect.json
                        '''
                    }
                }

                stage('Scan Docker Image') {
                    steps {
                        script {
                            int scanStatus = sh(
                                label: 'Trivy package vulnerability scan',
                                returnStatus: true,
                                script: '''#!/usr/bin/env bash
set -o pipefail

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache/trivy \
  aquasec/trivy:latest \
  image \
    --scanners vuln \
    --severity HIGH,CRITICAL \
    --ignore-unfixed \
    --exit-code 1 \
    "$LOCAL_IMAGE_NAME" | tee reports/trivy-image-results.txt
'''
                            )

                            if (scanStatus != 0) {
                                if (params.FAIL_ON_IMAGE_VULNERABILITIES) {
                                    error(
                                        'Trivy found fixed HIGH or CRITICAL vulnerabilities. ' +
                                        'See reports/trivy-image-results.txt.'
                                    )
                                }

                                unstable(
                                    'Trivy found fixed HIGH or CRITICAL vulnerabilities, ' +
                                    'but FAIL_ON_IMAGE_VULNERABILITIES is disabled.'
                                )
                            }
                        }
                    }
                }

                stage('Container Smoke Test') {
                    steps {
                        sh '''#!/usr/bin/env bash
set -euo pipefail

docker rm -f "$SMOKE_CONTAINER" >/dev/null 2>&1 || true
docker run -d --name "$SMOKE_CONTAINER" "$LOCAL_IMAGE_NAME"

for attempt in $(seq 1 20); do
  if docker exec "$SMOKE_CONTAINER" \
       python -c "import urllib.request; r=urllib.request.urlopen('http://127.0.0.1:8000/', timeout=2); assert r.status == 200"; then
    echo "Container smoke test passed."
    exit 0
  fi

  if [ "$(docker inspect -f '{{.State.Running}}' "$SMOKE_CONTAINER" 2>/dev/null || true)" != "true" ]; then
    echo "Container stopped before becoming healthy."
    docker logs "$SMOKE_CONTAINER" || true
    exit 1
  fi

  sleep 1
done

echo "Container did not become healthy within 20 seconds."
docker logs "$SMOKE_CONTAINER" || true
exit 1
'''
                    }
                }

                stage('Push to Docker Hub') {
                    steps {
                        script {
                            withCredentials([
                                usernamePassword(
                                    credentialsId: env.DOCKERHUB_CREDENTIALS_ID,
                                    usernameVariable: 'DOCKERHUB_USER',
                                    passwordVariable: 'DOCKERHUB_TOKEN'
                                )
                            ]) {
                                def targetRepository = params.DOCKERHUB_REPOSITORY.trim()

                                /*
                                 * If only a repository name was supplied,
                                 * publish below the authenticated user's
                                 * Docker Hub namespace.
                                 */
                                if (!targetRepository.contains('/')) {
                                    targetRepository =
                                        "${env.DOCKERHUB_USER}/${targetRepository}"
                                }

                                env.PUBLISHED_IMAGE =
                                    "${targetRepository}:${env.BUILD_NUMBER}"

                                try {
                                    sh '''#!/usr/bin/env bash
set -euo pipefail

printf '%s' "$DOCKERHUB_TOKEN" |
  docker login \
    --username "$DOCKERHUB_USER" \
    --password-stdin

docker tag "$LOCAL_IMAGE_NAME" "$PUBLISHED_IMAGE"
docker push "$PUBLISHED_IMAGE"

printf '%s\n' "$PUBLISHED_IMAGE" > reports/pushed-image.txt
'''
                                } finally {
                                    sh 'docker logout >/dev/null 2>&1 || true'
                                }
                            }
                        }
                    }
                }
            }

            post {
                always {
                    sh '''
                        docker logs "$SMOKE_CONTAINER" \
                          > reports/container.log 2>&1 || true

                        docker rm -f "$SMOKE_CONTAINER" \
                          >/dev/null 2>&1 || true

                        docker image rm -f "$LOCAL_IMAGE_NAME" \
                          >/dev/null 2>&1 || true

                        if [ -n "${PUBLISHED_IMAGE:-}" ]; then
                          docker image rm -f "$PUBLISHED_IMAGE" \
                            >/dev/null 2>&1 || true
                        fi
                    '''

                    archiveArtifacts(
                        artifacts: 'reports/**',
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                }
            }
        }
    }

    post {
        success {
            echo "CI completed successfully for ${env.BUILT_REPOSITORY ?: 'the configured SCM'}."
        }

        unsuccessful {
            echo "CI did not complete successfully. Review the failed stage and archived reports."
        }

        always {
            script {
                if (params.EMAIL_TO?.trim()) {
                    emailext(
                        to: params.EMAIL_TO.trim(),
                        subject: "${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        mimeType: 'text/html',
                        body: """
                            <h2>Jenkins CI result</h2>
                            <p><b>Job:</b> ${env.JOB_NAME}</p>
                            <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                            <p><b>Status:</b> ${currentBuild.currentResult}</p>
                            <p><b>Repository:</b> ${env.BUILT_REPOSITORY ?: params.REPO_URL ?: 'Configured SCM'}</p>
                            <p><b>Branch:</b> ${params.GIT_BRANCH}</p>
                            <p><b>Docker image:</b> ${env.PUBLISHED_IMAGE ?: 'Not published'}</p>
                            <p><a href="${env.BUILD_URL}">Open the Jenkins build</a></p>
                        """,
                        attachLog: true,
                        compressLog: true
                    )
                }
            }
        }
    }
}
