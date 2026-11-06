// Build the vale RPM in mock for every chroot listed below, then push the
// resulting SRPM and x86_64 RPM to the Nexus yum repository.
//
// Agent requirements: mock, rpmdevtools (spectool) and curl installed, and the
// Jenkins user in the "mock" group. The rhel-* chroots need the host registered
// with subscription-manager; swap in centos-stream-N-x86_64 or alma-N-x86_64 if
// that is not an option.

// mock chroot -> [platform, repo_name] path segments under the Nexus repo.
def CHROOTS = [
    'rhel-9-x86_64'   : ['rhel', '9'],
    'rhel-10-x86_64'  : ['rhel', '10'],
    'fedora-44-x86_64': ['fedora', '44'],
]

def SPEC    = 'vale.spec'
def SOURCES = 'sources'   // spectool downloads Source0 here
def OUT     = 'out'       // mock writes <OUT>/<chroot>/*.rpm and logs here

def rpm_upload_curl(rpm_file, platform, repo_name) {
    def sub_dir = rpm_file.name.endsWith('.src.rpm') ? 'SRPMS' : 'x86_64'
    def target  = "${env.NEXUS_URL}/repository/${env.NEXUS_REPO}/${platform}/${repo_name}/${sub_dir}/${rpm_file.name}"

    withCredentials([usernamePassword(credentialsId: 'jenkins-nexus-rpm-upload',
                                      usernameVariable: 'NEXUS_USER',
                                      passwordVariable: 'NEXUS_PASSWORD')]) {
        withEnv(["RPM_PATH=${rpm_file.path}", "TARGET_URL=${target}"]) {
            // Single-quoted on purpose: the shell expands the variables, so the
            // password never lands in the Groovy string or the console log.
            sh 'curl --fail --silent --show-error --user "$NEXUS_USER:$NEXUS_PASSWORD" --upload-file "$RPM_PATH" "$TARGET_URL"'
        }
    }
}

pipeline {
    agent { label 'master' }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    environment {
        NEXUS_URL  = "https://nexus-gls-common.apps.tools-na.prod.nextcle.com/"
        NEXUS_REPO = 'curriculum-repos'
    }

    stages {
        stage('Fetch source') {
            steps {
                sh "rm -rf ${SOURCES} ${OUT} && mkdir -p ${SOURCES} ${OUT}"
                sh "spectool -g -C ${SOURCES} ${SPEC}"
            }
        }

        stage('Build') {
            steps {
                script {
                    for (chroot in CHROOTS.keySet()) {
                        stage(chroot) {
                            sh "mock -r ${chroot} --spec ${SPEC} --sources ${SOURCES} --resultdir ${OUT}/${chroot}"
                        }
                    }
                }
            }
        }

        stage('Publish') {
            when {
                anyOf {
                    branch 'main'                                      // multibranch job
                    expression { env.GIT_BRANCH == 'origin/main' }     // plain pipeline job
                }
            }
            steps {
                script {
                    for (chroot in CHROOTS.keySet()) {
                        def (platform, repo_name) = CHROOTS[chroot]
                        for (rpm in findFiles(glob: "${OUT}/${chroot}/*.rpm")) {
                            rpm_upload_curl(rpm, platform, repo_name)
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${OUT}/*/*.rpm", fingerprint: true, allowEmptyArchive: true
            archiveArtifacts artifacts: "${OUT}/*/*.log", allowEmptyArchive: true
        }
    }
}
