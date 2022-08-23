def getBranchName() {
   return "${GIT_BRANCH.split('/').size() > 1 ? GIT_BRANCH.split('/')[1..-1].join('/') : GIT_BRANCH}"
}

def getSimplifiedBranchName() {
   return "${getBranchName().replace('/', '-')}"
}

pipeline {
    environment {
        BRANCH_NAME = getSimplifiedBranchName() // getBranchName()
        DOCKER_IMAGE_NAME = "gounthar/jenkinsci-docker-android-base:$BRANCH_NAME"
    }
    agent any
    options {
        timestamps()
    }
    stages {
//        stage('Checkout') {
//            agent any
//            steps {
//                echo 'Checkout if needed'
//            }
//        }
        stage('Static Analysis') {
            parallel {
                stage('Static Analysis with gradlew check') {
                    agent {
                        label 'android'
                    }
                    steps {
                        echo 'Run the static analysis to the code'
                        sh 'chmod +x ./gradlew'
                        sh './gradlew detekt --auto-correct'
                        sh 'git diff'
                        sh './gradlew check'
                    }
                }
                stage('Qodana') {
                    agent {
                        docker {
                            label 'docker'
                            image 'jetbrains/qodana-jvm-android'
                            args '-v .:/data/project/'
                            args '-v ./app/build/reports/qodana:/data/results/'
                            args '-v /home/jenkins/:/data/cache/'
                            args '--entrypoint=""'
                        }
                    }
                    steps {
                        sh "qodana --save-report"
                    }
                }
            }
        }
        stage('Compile and Code Coverage') {
            parallel {
                stage('Compile') {
                    environment {
                        ANDROID_PUBLISHER_CREDENTIALS = credentials('android-publisher-credentials')
                    }
                    agent {
                        label 'android'
                    }
                    steps {
                        script {
                            sh 'echo "Compile the source code"'
                            sh 'env | grep $HOME'
                            sh 'chmod +x ./gradlew'
                            sh './gradlew build'
                            sh 'ls -artl /home/jenkins/.gradle/wrapper/dists'
                            sh 'find /home/jenkins/ -name "gradle-7.3.3-bin.zip" -exec ls {} \\;'
                            sh './gradlew :app:bundleDebug :app:bundleRelease'
                        }
                    }
                }
                stage('Code Coverage') {
                    environment {
                        ANDROID_PUBLISHER_CREDENTIALS = credentials('android-publisher-credentials')
                    }
                    agent {
                        label 'android'
                    }
                    steps {
                        script {
                            sh 'echo "Ensure that the code coverage is not just wishful thinking"'
                            sh 'chmod +x ./gradlew'
                            sh 'chmod +x ./gradlew && ./gradlew jacocoTestReport'
                            sh 'find build -name "*eports*"'
                        }
                    }
                }
            }
        }
        stage('Security Check') {
            agent any
            steps {
                echo 'Run the security check against the application'
                echo 'Something like dependency check or dependabot'
            }
        }
        stage('Run Unit Tests') {
            agent {
                label 'android'
            }
            steps {
                echo 'Run unit tests from the source code'
                sh 'chmod +x gradlew && ./gradlew test'
            }
        }
        stage('Run Instrumented Tests') {
            agent {
                label 'android'
            }
            steps {
                echo 'Run only instrumented tests from the source code'
                // We don't have any device connected yet
                sh 'adb connect emulator:5555'
                sh 'adb devices'
                sh 'adb wait-for-device shell \'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 1; done;\''
                sh 'adb shell am start -n "io.jenkins.mobile.example.myfirstbuiltbyjenkinsapplication/io.jenkins.mobile.example.myfirstbuiltbyjenkinsapplication.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER'
                sh 'chmod +x ./gradlew &&./gradlew connectedAndroidTest'
            }
        }
        stage('Publish Artifacts') {
            agent any
            steps {
                echo 'Save the assemblies generated from the compilation'
            }
        }
        stage('Release on GitHub') {
            environment {
                GITHUB_CREDENTIALS = credentials('github-app-android')
                ANDROID_PUBLISHER_CREDENTIALS = credentials('android-publisher-credentials')
            }
            agent {
                label 'android'
            }
            steps {
                script {
                // Later on, move everything into functions and call them here.
                     releaseAlreadyExists = sh (
                            script: 'chmod +x ./jenkins/release-already-exists.sh && bash -x ./jenkins/release-already-exists.sh',
                            returnStdout: true
                        )
                        echo "Release already exists: $releaseAlreadyExists."
                        if (releaseAlreadyExists == 'false') {
                            echo "The release does not exist yet, so we can create it."
                            createRelease()
                        } else {
                            echo "The release already exists, so we won't create it."
                        }
                }
            }
        }
        stage('Release on Google Play Store') {
            environment {
                GITHUB_CREDENTIALS = credentials('github-app-android')
                ANDROID_PUBLISHER_CREDENTIALS = credentials('android-publisher-credentials')
            }
            agent {
                label 'android'
            }
            steps {
                echo 'Publishes the bundle on the Google Play Store'
                createGooglePlayStoreRelease()
            }
        }
    }
}

void releaseAlreadyExist(config) {
    GITHUB_CREDENTIALS_PSW = credentials('github-app-android').toString()
    echo $GITHUB_CREDENTIALS_PSW
}

void createRelease() {
    sh (
        script: 'chmod +x ./jenkins/create-release.sh && ./jenkins/create-release.sh',
        returnStdout: true
    )
}

void createGooglePlayStoreRelease() {
    sh (
        script: 'chmod +x ./jenkins/create-gps-release.sh && ./jenkins/create-gps-release.sh',
        returnStdout: true
    )
}
