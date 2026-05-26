pipeline {
  agent { label 'linux' }

  options {
    timeout(time: 30, unit: 'MINUTES')
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
  }

  triggers {
    pollSCM('H/5 * * * *')
  }

  environment {
    SHA_SHORT = "${env.GIT_COMMIT?.take(12) ?: 'manual'}"
    VERSION   = "rellince-${env.BUILD_NUMBER}-${SHA_SHORT}"
  }

  stages {
    stage('checkout') {
      steps { checkout scm }
    }

    stage('wire keystore') {
      steps {
        withCredentials([
          file(credentialsId: 'seadroid-keystore', variable: 'KEYSTORE_FILE'),
          string(credentialsId: 'seadroid-keystore-password', variable: 'KEYSTORE_PASSWORD'),
          string(credentialsId: 'seadroid-key-alias', variable: 'KEY_ALIAS'),
          string(credentialsId: 'seadroid-key-password', variable: 'KEY_PASSWORD')
        ]) {
          sh '''
            set -e
            cp "$KEYSTORE_FILE" app/release.keystore
            chmod 600 app/release.keystore
            cat > app/key.properties <<EOF
keyStore=release.keystore
keyStorePassword=$KEYSTORE_PASSWORD
keyAlias=$KEY_ALIAS
keyAliasPassword=$KEY_PASSWORD
EOF
            chmod 600 app/key.properties
          '''
        }
      }
    }

    stage('build') {
      steps {
        // DooD sibling container — cimg/android has JDK17 + Android SDK + Gradle.
        // Workspace bind-mount, run as Jenkins UID so produced files are readable
        // outside.
        sh '''
          set -e
          docker run --rm \\
            -v $WORKSPACE:/work -w /work \\
            -e GRADLE_USER_HOME=/work/.gradle \\
            --user $(id -u):$(id -g) \\
            cimg/android:2024.10.1 \\
            bash -c "./gradlew --no-daemon clean assembleRelease"
        '''
      }
    }

    stage('locate + rename apk') {
      steps {
        sh '''
          set -e
          APK_PATH=$(find app/build/outputs/apk/release -name "*.apk" | head -1)
          if [ -z "$APK_PATH" ]; then
            echo "::error::No release APK produced"
            exit 1
          fi
          cp "$APK_PATH" "seadroid-${VERSION}.apk"
          ls -la "seadroid-${VERSION}.apk"
        '''
      }
    }

    stage('archive') {
      steps {
        archiveArtifacts artifacts: "seadroid-${VERSION}.apk", fingerprint: true
      }
    }
  }

  post {
    always {
      // Wipe keystore + properties from workspace BEFORE cleanWs (defensive)
      sh 'rm -f app/release.keystore app/key.properties || true'
      cleanWs()
    }
    failure { echo "Build ${env.BUILD_NUMBER} failed: ${env.BUILD_URL}" }
  }
}
