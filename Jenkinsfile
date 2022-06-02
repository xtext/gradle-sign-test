pipeline {
  agent {
    kubernetes {
        inheritFrom 'centos-8'
    }
  }

  triggers {
    // only on master branch / 30 minutes before nightly sign-and-deploy
    cron(env.BRANCH_NAME == 'master' ? 'H 15 * * *' : '')
    githubPush()
  }

  environment {
    GRADLE_USER_HOME = "$WORKSPACE/.gradle" // workaround for https://bugs.eclipse.org/bugs/show_bug.cgi?id=564559
    DOWNLOAD_AREA = '/home/data/httpd/download.eclipse.org/modeling/tmf/xtext/downloads/drops'
    KEYRING = credentials('secret-subkeys.asc')
    SCRIPTS = "$WORKSPACE/umbrella/releng/jenkins/sign-and-deploy/scripts"
    GITHUB_API_CREDENTIALS_ID = 'github-bot-token'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr:'15'))
    disableConcurrentBuilds()
    timeout(time: 90, unit: 'MINUTES')
  }

  // Build stages
  stages {
    stage('Build') {
      tools {
        maven "apache-maven-3.8.5"
        jdk "temurin-jdk11-latest"
      }
      environment {
        EXTRA_ARGS = "-Dmaven.repo.local=.m2 -Dtycho.localArtifacts=ignore"
        MAVEN_OPTS="-Xmx512m"
      }
      steps {
        checkout scm
        wrap([$class: 'Xvnc', takeScreenshot: false, useXauthority: true]) {
          withCredentials([string(credentialsId: 'gpg_passphrase', variable: 'GPG_PASSPHRASE')]) {
            sh '''
            env
            echo "$(pwd)"
            export GPG_TTY=$(tty)
            gpg --version
            gpg --batch --import "${KEYRING}"
            for fpr in $(gpg --list-keys --with-colons  | awk -F: '/fpr:/ {print $10}' | sort -u);
            do
              echo -e "5\ny\n" | gpg --batch --command-fd 0 --expert --edit-key $fpr trust;
            done
            ls -la /home/jenkins/.gradle/gradle.properties
            ./gradlew clean build --info -Psigning.gnupg.passphrase=${GPG_PASSPHRASE}
            '''
          }
        }
      }
    }
  }

  post {
    success {
      archiveArtifacts artifacts: 'build/libs/**'
    }
    cleanup {
      script {
        def curResult = currentBuild.currentResult
        def lastResult = 'NEW'
        if (currentBuild.previousBuild != null) {
          lastResult = currentBuild.previousBuild.result
        }

        if (curResult != 'SUCCESS' || lastResult != 'SUCCESS') {
          def color = ''
          switch (curResult) {
            case 'SUCCESS':
              color = '#00FF00'
              break
            case 'UNSTABLE':
              color = '#FFFF00'
              break
            case 'FAILURE':
              color = '#FF0000'
              break
            default: // e.g. ABORTED
              color = '#666666'
          }

          slackSend (
            message: "${lastResult} => ${curResult}: <${env.BUILD_URL}|${env.JOB_NAME}#${env.BUILD_NUMBER}>",
            botUser: true,
            channel: 'xtext-builds',
            color: "${color}"
          )
        }
      }
    }
  }
}