pipeline {
    agent any

    environment {
        BUILD_CONFIGURATION = 'Release'
        ANDROID_HOME = '/path/to/android/sdk'  // Android SDK location
        DOTNET_VERSION = '6.0'
        SONARQUBE_SCANNER_HOME = tool 'SonarQubeScanner' // SonarQube scanner configuration
    }

    tools {
        // Ensure .NET SDK is installed on the build agent
        dotnet '6.0'
    }

    stages {
        // Stage 1: Checkout source code from the repository
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-repo/your-mobile-app.git'
            }
        }

        // Stage 2: Restore dependencies (NuGet packages)
        stage('Restore Dependencies') {
            steps {
                // Restore NuGet packages for the project
                script {
                    if (isUnix()) {
                        sh 'dotnet restore'
                    } else {
                        bat 'dotnet restore'
                    }
                }
            }
        }

        // Stage 3: SonarQube Code Quality Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQubeServer') { // Assuming "SonarQubeServer" is configured in Jenkins
                    script {
                        if (isUnix()) {
                            sh '''
                                ${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=your-mobile-app \
                                -Dsonar.sources=. \
                                -Dsonar.cs.opencover.reportsPaths=coverage.opencover.xml \
                                -Dsonar.host.url=https://your-sonarqube-server \
                                -Dsonar.login=your-sonar-token
                            '''
                        } else {
                            bat '''
                                ${SONARQUBE_SCANNER_HOME}\\bin\\sonar-scanner.bat ^
                                -Dsonar.projectKey=your-mobile-app ^
                                -Dsonar.sources=. ^
                                -Dsonar.cs.opencover.reportsPaths=coverage.opencover.xml ^
                                -Dsonar.host.url=https://your-sonarqube-server ^
                                -Dsonar.login=your-sonar-token
                            '''
                        }
                    }
                }
            }
        }

        // Stage 4: Build for Android
        stage('Build Android') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'dotnet build --configuration ${BUILD_CONFIGURATION} --framework net6.0-android'
                    } else {
                        bat 'dotnet build --configuration ${BUILD_CONFIGURATION} --framework net6.0-android'
                    }
                }
            }
        }

        // Stage 5: Build for iOS (only on macOS)
        stage('Build iOS') {
            when {
                expression {
                    isUnix() && fileExists('/usr/bin/xcodebuild') // Ensure it's a macOS agent
                }
            }
            steps {
                sh 'dotnet build --configuration ${BUILD_CONFIGURATION} --framework net6.0-ios'
            }
        }

        // Stage 6: Run Unit Tests
        stage('Run Unit Tests') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'dotnet test --configuration ${BUILD_CONFIGURATION} --logger trx --results-directory ./TestResults'
                    } else {
                        bat 'dotnet test --configuration ${BUILD_CONFIGURATION} --logger trx --results-directory ./TestResults'
                    }
                }
            }
            post {
                always {
                    // Publish test results, also useful for SonarQube coverage reports
                    junit '**/TestResults/*.trx'
                }
            }
        }

        // Stage 7: Package Android APK
        stage('Package Android APK') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'msbuild /t:PackageForAndroid /p:Configuration=${BUILD_CONFIGURATION}'
                    } else {
                        bat 'msbuild /t:PackageForAndroid /p:Configuration=${BUILD_CONFIGURATION}'
                    }
                }
            }
        }

        // Stage 8: Package iOS IPA (on macOS only)
        stage('Package iOS IPA') {
            when {
                expression {
                    isUnix() && fileExists('/usr/bin/xcodebuild')
                }
            }
            steps {
                sh 'msbuild /t:Build /p:Platform=iPhone /p:Configuration=Release /p:BuildIpa=true'
            }
        }

        // Stage 9: Archive Artifacts (APK and IPA files)
        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: '**/bin/**/*.apk, **/bin/**/*.ipa', allowEmptyArchive: true
            }
        }

        // Stage 10: Deploy to Test Environment (Optional)
        stage('Deploy to Test Environment') {
            steps {
                script {
                    // Optionally deploy to App Center for testing
                    sh '''
                        appcenter distribute release --app "YourApp/android" --file ./bin/Release/android.apk --group "Testers"
                        appcenter distribute release --app "YourApp/ios" --file ./bin/Release/ios.ipa --group "Testers"
                    '''
                }
            }
        }
    }

    post {
        // Always notify and clean up resources after the pipeline run
        always {
            echo 'Pipeline execution completed.'
        }

        // Perform a quality gate check after SonarQube analysis
        success {
            script {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}

