// The recipient is a Jenkins credential so this public repo never carries an address.
def notifyTeam(String status) {
    withCredentials([string(credentialsId: 'notify-email', variable: 'NOTIFY_TO')]) {
        mail to: env.NOTIFY_TO,
             subject: "[${status}] ${env.JOB_NAME} #${env.BUILD_NUMBER} (${env.BRANCH_NAME})",
             body: """Pipeline: ${env.JOB_NAME}
Branch:   ${env.BRANCH_NAME}
Build:    #${env.BUILD_NUMBER}
Result:   ${status}
URL:      ${env.BUILD_URL}
"""
    }
}

pipeline {
    agent {
        kubernetes {
            cloud 'kind'
            yamlFile 'ci/flutter-pod.yaml'
            defaultContainer 'flutter'
        }
    }

    environment {
        APP_NAME = 'taskflow-mobile'
        // Caches live on the flutter-cache PVC so each fresh pod skips re-downloading Gradle and pub.
        GRADLE_USER_HOME = '/cache/gradle'
        PUB_CACHE = '/cache/pub'
        // A long-lived Gradle daemon outlives the build step and was enough to OOM-kill the pod.
        GRADLE_OPTS = '-Dorg.gradle.daemon=false'
        ANDROID_KEY_ALIAS = 'upload'
    }

    options {
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '5'))
    }

    stages {
        stage('Setup') {
            steps {
                sh 'flutter --version --suppress-analytics && flutter pub get --suppress-analytics'
            }
        }

        stage('Parallel Checks') {
            failFast true
            parallel {
                stage('flutter analyze') {
                    steps { sh 'flutter analyze --suppress-analytics' }
                }
                stage('flutter test --coverage') {
                    steps { sh 'flutter test --coverage --suppress-analytics' }
                    post {
                        always {
                            archiveArtifacts artifacts: 'coverage/lcov.info', allowEmptyArchive: true
                        }
                    }
                }
                stage('SCA - osv-scanner') {
                    steps {
                        container('osv') {
                            // Exits 1 when any dependency in the lockfile has a known vulnerability.
                            sh '/osv-scanner scan source --lockfile=pubspec.lock'
                        }
                    }
                }
            }
        }

        stage('Build Debug APK') {
            steps {
                sh 'flutter build apk --debug --suppress-analytics'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build/app/outputs/flutter-apk/app-debug.apk', fingerprint: true
                }
            }
        }

        stage('Build Signed Release AAB') {
            when { branch 'main' }
            steps {
                withCredentials([
                    file(credentialsId: 'android-keystore', variable: 'ANDROID_KEYSTORE_PATH'),
                    string(credentialsId: 'android-keystore-password', variable: 'ANDROID_KEYSTORE_PASSWORD')
                ]) {
                    // PKCS12 keystores use one password for both the store and the key.
                    sh 'ANDROID_KEY_PASSWORD="$ANDROID_KEYSTORE_PASSWORD" flutter build appbundle --release --suppress-analytics'
                }
                sh '''
                    aab=build/app/outputs/bundle/release/app-release.aab
                    jarsigner -verify "$aab"
                    keytool -printcert -jarfile "$aab" | tee signer.txt
                    # Gradle silently falls back to debug keys when no keystore is bound; never ship that.
                    if grep -q "CN=Android Debug" signer.txt; then
                        echo "Release AAB is signed with the debug key"
                        exit 1
                    fi
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build/app/outputs/bundle/release/app-release.aab, signer.txt', fingerprint: true
                }
            }
        }
    }

    post {
        success { notifyTeam('SUCCESS') }
        failure { notifyTeam('FAILURE') }
    }
}
