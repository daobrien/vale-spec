// Build the vale RPM in mock for every chroot listed below (in parallel), then push the
// resulting SRPM and x86_64 RPM to the Nexus yum repository.
//
// Agent requirements: mock, rpmdevtools (spectool) and curl installed, and the
// Jenkins user in the "mock" group. The rhel-* chroots need the host registered
// with subscription-manager; the centos-stream-* and fedora-* chroots do not.
// Every chroot needs a matching /etc/mock/<chroot>.cfg from mock-core-configs
// and its signing key from distribution-gpg-keys. Fedora releases past EOL are
// moved to /etc/mock/eol/ by mock-core-configs; drop them from CHROOTS then.
//
// RPMs already present in Nexus are skipped rather than re-uploaded, so
// re-running the pipeline for a published version is safe.

// mock chroot -> [platform, repo_name] path segments under the Nexus repo.
def CHROOTS = [
    'rhel-9-x86_64'           : ['rhel', '9'],
    'rhel-10-x86_64'          : ['rhel', '10'],
    'centos-stream-9-x86_64'  : ['centos', '9'],
    'centos-stream-10-x86_64' : ['centos', '10'],
    'fedora-43-x86_64'        : ['fedora', '43'],
    'fedora-44-x86_64'        : ['fedora', '44'],
    'fedora-45-x86_64'        : ['fedora', '45'],
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
            //
            // Nexus rejects re-uploads of an existing asset (deployment policy
            // "disable redeploy"), so skip anything that is already published.
            // Rebuilding a version that is already in the repo is a no-op, not
            // a failure; bump the Release in the spec to publish a new build.
            sh '''
                if curl --fail --silent --head --output /dev/null --user "$NEXUS_USER:$NEXUS_PASSWORD" "$TARGET_URL"; then
                    echo "Already published, skipping: $TARGET_URL"
                else
                    curl --fail --silent --show-error --user "$NEXUS_USER:$NEXUS_PASSWORD" --upload-file "$RPM_PATH" "$TARGET_URL"
                fi
            '''
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
        NEXUS_URL  = "https://nexus-gls-common.apps.tools-na.prod.nextcle.com"
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
                    // One parallel branch per chroot. Mock keeps a separate root,
                    // cache and lock under /var/lib/mock/<chroot>, so distinct
                    // chroots are safe to build concurrently on one host.
                    def builds = [:]
                    for (c in CHROOTS.keySet()) {
                        def chroot = c   // fresh binding per closure
                        builds[chroot] = {
                            sh "mock -r ${chroot} --spec ${SPEC} --sources ${SOURCES} --resultdir ${OUT}/${chroot}"
                        }
                    }
                    parallel builds
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
