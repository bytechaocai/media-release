def checkoutRepo(String targetDir, String repoUrl, String branch) {
    def remote = [url: repoUrl]
    def credentialsId = params.CREDENTIALS_ID?.trim()
    if (credentialsId) {
        remote.credentialsId = credentialsId
    }
    dir(targetDir) {
        checkout([
            $class: 'GitSCM',
            branches: [[name: branch]],
            userRemoteConfigs: [remote],
            extensions: [[$class: 'CloneOption', shallow: true, depth: 1, noTags: false, timeout: 30]]
        ])
    }
}

def prepareVersion() {
    def buildVersion = params.RELEASE_VERSION?.trim()
    if (!buildVersion) {
        buildVersion = sh(script: 'git describe --tags HEAD 2>/dev/null || true', returnStdout: true).trim()
    }
    if (!buildVersion) {
        error('无法通过 git describe 获取发行版号，请在 RELEASE_VERSION 参数中指定')
    }

    def serverRevision = sh(script: 'git -C media rev-parse --short HEAD', returnStdout: true).trim()
    def uiRevision = sh(script: 'git -C media-ui rev-parse --short HEAD', returnStdout: true).trim()
    if (!serverRevision || !uiRevision) {
        error('无法获取前后端代码版本')
    }
    def serverVersion = sh(script: "git -C media tag --contains ${serverRevision} 2>/dev/null | head -n 1", returnStdout: true).trim()
    def uiVersion = sh(script: "git -C media-ui tag --contains ${uiRevision} 2>/dev/null | head -n 1", returnStdout: true).trim()

    env.BUILD_VERSION = buildVersion
    env.SERVER_REVISION = serverRevision
    env.UI_REVISION = uiRevision
    env.SERVER_VERSION = serverVersion
    env.UI_VERSION = uiVersion

    echo "发行版: ${buildVersion}"
    echo "后端版本: ${serverRevision}（tag: ${serverVersion ?: '无'}）"
    echo "前端版本: ${uiRevision}（tag: ${uiVersion ?: '无'}）"

    sh 'mkdir -p media/application/src/main/resources/static'
    writeVersionProperties(buildVersion, serverVersion, uiVersion)
    writeVersionMarkers(serverRevision, uiRevision)

    sh 'cp media/releasenote.html media/application/src/main/resources/static/'
    sh 'cp media-ui/releasenote-ui.html media/application/src/main/resources/static/'

    currentBuild.description = "${params.BRANCH} | ${buildVersion} | server=${serverRevision} ui=${uiRevision}"
}

def writeVersionProperties(String buildVersion, String serverVersion, String uiVersion) {
    def propertiesPath = 'media/config/src/main/resources/version.properties'
    def properties = [:]
    if (fileExists(propertiesPath)) {
        readFile(propertiesPath).readLines().each { line ->
            def position = line.indexOf('=')
            if (position > 0) {
                properties[line.substring(0, position).trim()] = line.substring(position + 1).trim()
            }
        }
    }
    properties['main'] = "v${buildVersion}"
    properties['server'] = serverVersion
    properties['client'] = uiVersion
    writeFile file: propertiesPath, text: properties.collect { key, value -> "${key}=${value}" }.join('\n') + '\n'
}

def writeVersionMarkers(String serverRevision, String uiRevision) {
    dir('media/application/src/main/resources/static') {
        writeFile file: "server-${serverRevision}", text: ''
        writeFile file: "ui-${uiRevision}", text: ''
    }
    dir('media/cli/src/main/resources') {
        writeFile file: "cli-${serverRevision}", text: ''
    }
}

def buildUi() {
    dir('media-ui') {
        if (!fileExists('src/pages/demo/index.tsx.backup')) {
            error('缺少 src/pages/demo/index.tsx.backup，无法复制 demo 页面')
        }
        sh 'cp src/pages/demo/index.tsx.backup src/pages/demo/index.tsx'
        def npmRegistry = params.NPM_REGISTRY?.trim() ?: 'https://registry.npmjs.org/'
        withEnv(["NPM_CONFIG_REGISTRY=${npmRegistry}"]) {
            sh 'yarn install'
            sh 'yarn run build'
        }
        if (!fileExists('dist')) {
            error('前端构建未生成 dist 目录')
        }
        dir('dist') {
            stash name: 'ui-dist', includes: '**'
        }
    }
}

def assemble() {
    def modules = ['common', 'service', 'web', 'dataaccess', 'config']
    def appJar = "media-${params.BRANCH}-${env.BUILD_VERSION}-${env.SERVER_REVISION}-${env.UI_REVISION}.jar"
    def cliJar = "cli-${params.BRANCH}-${env.BUILD_VERSION}-${env.SERVER_REVISION}.jar"
    env.APP_JAR = appJar
    env.CLI_JAR = cliJar
    echo "web 程序 jar 包: ${appJar}"
    echo "命令行程序 jar 包: ${cliJar}"

    parallel(
        '应用 jar': {
            sh "mv media/application/target/*.jar app/${appJar}"
        },
        '命令行 jar': {
            sh "mv media/cli/target/*.jar app/${cliJar}"
        },
        'Maven 依赖': {
            sh 'mv media/application/target/dependency/* app/lib/'
        },
        '模块 jar': {
            modules.each { module ->
                sh "mv media/${module}/target/*.jar app/lib/"
            }
        }
    )
}

def createStartScripts() {
    def cliMainClasses = ['LoadSingleFile', 'LoadDirectory']
    def javaCommand = 'java -Dspring.profiles.active=prd,cli'
    dir('app') {
        writeFile file: 'media.bat', text: "${javaCommand} -cp ${env.APP_JAR} com.sheepfly.media.application.MediaApplication\r\n"
        cliMainClasses.each { mainClass ->
            writeFile file: "${mainClass}.bat", text: "${javaCommand} -Dmodule=${mainClass} -cp ${env.CLI_JAR} com.sheepfly.media.cli.${mainClass} %*\r\n"
        }
    }
}


pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        timeout(time: 3, unit: 'HOURS')
    }

    parameters {
        choice(name: 'BRANCH', choices: ['feature', 'master'], description: '打包分支')
        string(name: 'SERVER_REPO', defaultValue: 'git@github.com:bytechaocai/media.git', description: '后端代码仓库')
        string(name: 'UI_REPO', defaultValue: 'git@github.com:bytechaocai/media-ui.git', description: '前端代码仓库')
        string(name: 'RELEASE_VERSION', defaultValue: '', description: '发行版号。留空时使用 git describe --tags HEAD')
        string(name: 'CREDENTIALS_ID', defaultValue: '', description: '拉取代码的凭据 ID。留空时使用构建节点上的 SSH key')
        string(name: 'AGENT_LABEL', defaultValue: 'linux', description: '构建节点标签')
        string(name: 'NPM_REGISTRY', defaultValue: 'https://registry.npmjs.org/', description: 'npm 镜像地址，默认使用 npm 官方源')
    }

    stages {
        stage('清理工作目录') {
            steps {
                script {
                    ['media', 'media-ui', 'app'].each { name ->
                        echo "清理目录: ${name}"
                        dir(name) {
                            deleteDir()
                        }
                    }
                    sh 'mkdir -p app/lib'
                }
            }
        }

        stage('拉取代码') {
            steps {
                script {
                    parallel(
                        '后端代码': {
                            checkoutRepo('media', params.SERVER_REPO, params.BRANCH)
                        },
                        '前端代码': {
                            checkoutRepo('media-ui', params.UI_REPO, params.BRANCH)
                        }
                    )
                }
            }
        }

        stage('设置版本信息') {
            steps {
                script {
                    prepareVersion()
                }
            }
        }

        stage('并行构建') {
            steps {
                script {
                    parallel(
                        '前端构建': {
                            buildUi()
                        },
                        '后端编译': {
                            dir('media') {
                                sh 'mvn -B -Dfile.encoding=UTF-8 -Dmaven.test.skip=true compile'
                            }
                        },
                        'Maven 依赖复制': {
                            dir('media/application') {
                                sh 'mvn -B dependency:copy-dependencies'
                            }
                        }
                    )
                }
            }
        }

        stage('复制前端产物') {
            steps {
                script {
                    dir('media/application/src/main/resources/static') {
                        unstash 'ui-dist'
                    }
                }
            }
        }

        stage('后端打包') {
            steps {
                script {
                    dir('media') {
                        sh 'mvn -B -Dfile.encoding=UTF-8 -Dmaven.test.skip=true package'
                    }
                }
            }
        }

        stage('组装发行包') {
            steps {
                script {
                    assemble()
                }
            }
        }

        stage('创建启动脚本') {
            steps {
                script {
                    createStartScripts()
                }
            }
        }

        stage('归档产物') {
            steps {
                archiveArtifacts artifacts: 'app/**', fingerprint: true, allowEmptyArchive: false
            }
        }
    }

    post {
        success {
            echo "打包完成: ${env.BUILD_VERSION}，前端 ${env.UI_REVISION}，后端 ${env.SERVER_REVISION}"
        }
        failure {
            echo '打包失败，请查看上方日志'
        }
    }
}
