def TOOL_CHECKS = [
    'git' : [cmd: 'git --version', min: [2, 0]],
    'java': [cmd: 'java -version', min: [8, 0]],
    'mvn' : [cmd: 'mvn -v', min: [3, 6]],
    'node': [cmd: 'node -v', min: [14, 0]],
    'npm' : [cmd: 'npm -v', min: [6, 0]]
]

def MODULES = ['common', 'service', 'web', 'dataaccess', 'config']

def CLI_MAIN_CLASSES = ['LoadSingleFile', 'LoadDirectory']


def checkEnvironment() {
    echo "构建节点: ${env.NODE_NAME}"
    echo "工作目录: ${env.WORKSPACE}"
    echo "执行平台: ${isUnix() ? 'Unix' : 'Windows'}"

    try {
        sh(script: 'sh -c "echo shell-ok"', returnStdout: true)
    } catch (ignored) {
        error('未找到 sh，Windows 节点必须安装 Git Bash 并将 sh 加入 PATH')
    }

    def versionChecks = [:]
    TOOL_CHECKS.each { name, spec ->
        versionChecks[name] = {
            checkToolVersion(name, spec.cmd, spec.min)
        }
    }
    parallel versionChecks

    def writable = sh(script: 'test -w . && echo yes || echo no', returnStdout: true).trim()
    if (writable != 'yes') {
        error("工作目录不可写: ${env.WORKSPACE}")
    }

    if (env.JAVA_HOME) {
        def javaHome = shellPath(env.JAVA_HOME)
        def exists = sh(script: "test -d '${javaHome}' && echo yes || echo no", returnStdout: true).trim()
        if (exists != 'yes') {
            error("JAVA_HOME 指向的目录不存在: ${env.JAVA_HOME}")
        }
        echo "JAVA_HOME: ${env.JAVA_HOME}"
    } else {
        echo 'JAVA_HOME 未设置，使用 PATH 中的 java'
    }

    def m2Exists = sh(script: 'test -d "$HOME/.m2/repository" && echo yes || echo no', returnStdout: true).trim()
    if (m2Exists == 'yes') {
        echo 'Maven 本地仓库: $HOME/.m2/repository'
    } else {
        echo '警告: 未找到 Maven 本地仓库 $HOME/.m2/repository，构建时会重新下载依赖'
    }

    ['media', 'media-ui', 'app'].each { name ->
        echo "工作目录 ${name}: ${fileExists(name) ? '已存在，构建时将重建' : '不存在，构建时创建'}"
    }
}

def checkToolVersion(String name, String command, List minVersion) {
    def output = sh(script: "${command} 2>&1", returnStdout: true).trim()
    if (!output) {
        error("无法获取 ${name} 版本，请确认 ${name} 已安装并在 PATH 中")
    }
    def matcher = (output =~ /(\d+)\.(\d+)/)
    if (!matcher.find()) {
        error("无法解析 ${name} 版本: ${output}")
    }
    def major = Integer.parseInt(matcher.group(1))
    def minor = Integer.parseInt(matcher.group(2))
    if (name == 'java' && major == 1) {
        major = minor
        minor = 0
    }
    def actual = output.readLines()[0].trim()
    echo "${name}: ${actual}（要求 >= ${minVersion[0]}.${minVersion[1]}）"
    if (major < minVersion[0] || (major == minVersion[0] && minor < minVersion[1])) {
        error("${name} 版本过低，当前 ${actual}，要求 >= ${minVersion[0]}.${minVersion[1]}")
    }
}

def shellPath(String path) {
    if (!path) {
        return path
    }
    if (!isUnix()) {
        def converted = sh(script: "cygpath -u '${path}' 2>/dev/null || true", returnStdout: true).trim()
        if (converted) {
            return converted
        }
    }
    return path
}

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
        sh 'npm install --no-audit --no-fund'
        sh 'npm run build'
        if (!fileExists('dist')) {
            error('前端构建未生成 dist 目录')
        }
        dir('dist') {
            stash name: 'ui-dist', includes: '**'
        }
    }
}

def assemble() {
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
            dir('media/application') {
                sh 'mvn -B dependency:copy-dependencies'
            }
            sh 'mv media/application/target/dependency/* app/lib/'
        },
        '模块 jar': {
            MODULES.each { module ->
                sh "mv media/${module}/target/*.jar app/lib/"
            }
        }
    )
}

def createStartScripts() {
    def javaCommand = 'java -Dspring.profiles.active=prd,cli'
    dir('app') {
        writeFile file: 'media.bat', text: "${javaCommand} -cp ${env.APP_JAR} com.sheepfly.media.application.MediaApplication\r\n"
        CLI_MAIN_CLASSES.each { mainClass ->
            writeFile file: "${mainClass}.bat", text: "${javaCommand} -Dmodule=${mainClass} -cp ${env.CLI_JAR} com.sheepfly.media.cli.${mainClass} %*\r\n"
        }
    }
}


pipeline {
    agent { label "${params.AGENT_LABEL ?: 'windows'}" }

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
        string(name: 'AGENT_LABEL', defaultValue: 'windows', description: '构建节点标签，节点需安装 Git Bash 并将 sh 加入 PATH')
    }

    stages {
        stage('环境检查') {
            steps {
                script {
                    checkEnvironment()
                }
            }
        }

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
                        '后端依赖': {
                            dir('media/application') {
                                sh 'mvn -B dependency:copy-dependencies'
                            }
                        }
                    )
                }
            }
        }

        stage('后端打包') {
            steps {
                script {
                    dir('media/application/src/main/resources/static') {
                        unstash 'ui-dist'
                    }
                    dir('media') {
                        sh 'mvn -B -Dfile.encoding=UTF-8 -Dmaven.test.skip=true install'
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
