def checkoutRepo(String targetDir, String repoUrl, String branch) {
    def remote = [url: repoUrl]
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
    def buildVersion = params.releaseVersion?.trim()
    if (!buildVersion) {
        buildVersion = sh(script: 'git describe --tags HEAD 2>/dev/null || true', returnStdout: true).trim()
    }
    if (!buildVersion) {
        error('无法通过 git describe 获取发行版号，请在 releaseVersion 参数中指定')
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

    currentBuild.description = "${params.branch} | ${buildVersion} | server=${serverRevision} ui=${uiRevision}"
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
    def nodeModulesDir = params.nodeModuleDir?.trim() ?: env.NODE_MODULE_DIR?.trim()
    if (!nodeModulesDir) {
        error('nodeModuleDir 参数或 NODE_MODULE_DIR 环境变量不能为空')
    }
    if (fileExists('media-ui/node_modules')) {
        error('前端工程中已存在 node_modules，无法安全切换依赖目录')
    }

    withEnv(["NODE_MODULES_DIR=${nodeModulesDir}"]) {
        sh(
            label: '移入前端依赖目录',
            script: '''
                set -e
                if [ -e "$NODE_MODULES_DIR" ]; then
                    if [ ! -d "$NODE_MODULES_DIR" ]; then
                        echo "nodeModuleDir 不是目录: $NODE_MODULES_DIR" >&2
                        exit 1
                    fi
                    mv "$NODE_MODULES_DIR" media-ui/node_modules
                fi
            '''
        )

        dir('media-ui') {
            if (!fileExists('src/pages/demo/index.tsx.backup')) {
                error('缺少 src/pages/demo/index.tsx.backup，无法复制 demo 页面')
            }
            sh 'cp src/pages/demo/index.tsx.backup src/pages/demo/index.tsx'
            def npmRegistry = params.npmRegistry?.trim() ?: 'https://registry.npmjs.org/'
            withEnv(["NPM_CONFIG_REGISTRY=${npmRegistry}"]) {
                if (!fileExists('node_modules')) {
                    sh 'yarn install'
                } else {
                    echo '已存在 node_modules，跳过 yarn install'
                }
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
}

def assemble() {
    def modules = ['common', 'service', 'web', 'dataaccess', 'config']
    def appJar = "media-${params.branch}-${env.BUILD_VERSION}-${env.SERVER_REVISION}-${env.UI_REVISION}.jar"
    def cliJar = "cli-${params.branch}-${env.BUILD_VERSION}-${env.SERVER_REVISION}.jar"
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

def preserveRootNodeModules() {
    def nodeModulesDir = params.nodeModuleDir?.trim() ?: env.NODE_MODULE_DIR?.trim()
    if (!nodeModulesDir) {
        error('nodeModuleDir 参数或 NODE_MODULE_DIR 环境变量不能为空')
    }

    withEnv(["NODE_MODULES_DIR=${nodeModulesDir}"]) {
        sh(
            label: '保护现有前端依赖',
            script: '''
                set -e
                if [ -d node_modules ]; then
                    source_dir=$(cd node_modules && pwd -P)
                    target_parent=$(dirname "$NODE_MODULES_DIR")
                    mkdir -p "$target_parent"
                    target_dir="$(cd "$target_parent" && pwd -P)/$(basename "$NODE_MODULES_DIR")"
                    if [ "$source_dir" = "$target_dir" ]; then
                        echo "依赖目录已在目标位置: $NODE_MODULES_DIR"
                    elif [ -e "$NODE_MODULES_DIR" ]; then
                        echo "依赖目录目标已存在，不覆盖: $NODE_MODULES_DIR" >&2
                        exit 1
                    else
                        mv -n node_modules "$NODE_MODULES_DIR"
                        if [ -e node_modules ]; then
                            echo "依赖目录未能移动到目标位置: $NODE_MODULES_DIR" >&2
                            exit 1
                        fi
                    fi
                fi
            '''
        )
    }
}

def workspaceCleanupExclusions() {
    def nodeModuleDir = params.nodeModuleDir?.trim() ?: env.NODE_MODULE_DIR?.trim()
    if (!nodeModuleDir) {
        error('nodeModuleDir 参数或 NODE_MODULE_DIR 环境变量不能为空')
    }

    def workspacePath = pwd().replace('\\', '/').replaceAll('/+$', '')
    def configuredPath = nodeModuleDir.replace('\\', '/')
    def relativePath = null
    def isAbsolute = configuredPath.startsWith('/') || (configuredPath ==~ /^[A-Za-z]:\/.*/)

    if (isAbsolute) {
        def normalizedWorkspace = workspacePath.toLowerCase()
        def normalizedConfigured = configuredPath.toLowerCase()
        if (normalizedConfigured == normalizedWorkspace) {
            error('nodeModuleDir 不能指向整个工作目录')
        }
        if (normalizedConfigured.startsWith("${normalizedWorkspace}/")) {
            relativePath = configuredPath.substring(workspacePath.length() + 1)
        }
    } else if (configuredPath != '..' && !configuredPath.startsWith('../')) {
        relativePath = configuredPath
    }

    if (!relativePath) {
        return []
    }

    return [
        [pattern: relativePath, type: 'EXCLUDE'],
        [pattern: "${relativePath}/**", type: 'EXCLUDE']
    ]
}


pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '5'))
        timeout(time: 3, unit: 'HOURS')
    }

    parameters {
        choice(name: 'branch', choices: ['feature', 'master'], description: '打包分支')
        string(name: 'serverRepo', defaultValue: 'git@github.com:bytechaocai/media.git', description: '后端代码仓库')
        string(name: 'uiRepo', defaultValue: 'git@github.com:bytechaocai/media-ui.git', description: '前端代码仓库')
        string(name: 'releaseVersion', defaultValue: '', description: '发行版号。留空时使用 git describe --tags HEAD')
        string(name: 'npmRegistry', defaultValue: 'https://registry.npmjs.org/', description: 'Yarn registry 地址，默认使用官方源')
        string(name: 'nodeModuleDir', defaultValue: "${env.NODE_MODULE_DIR ?: 'node_modules'}", description: '依赖缓存目录；工作区相对路径或构建节点上的绝对路径，默认读取 NODE_MODULE_DIR 环境变量')
        booleanParam(name: 'copyDependencies', defaultValue: true, description: '是否将后端第三方依赖复制到发行包')
        booleanParam(name: 'fingerprint', defaultValue: false, description: '是否为归档制品生成指纹')
    }

    stages {
        stage('清理工作目录') {
            steps {
                script {
                    preserveRootNodeModules()
                    cleanWs(deleteDirs: true, patterns: workspaceCleanupExclusions())
                    sh 'mkdir -p app/lib'
                }
            }
        }

        stage('拉取代码') {
            steps {
                script {
                    parallel(
                        '后端代码': {
                            checkoutRepo('media', params.serverRepo, params.branch)
                        },
                        '前端代码': {
                            checkoutRepo('media-ui', params.uiRepo, params.branch)
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
                            if (params.copyDependencies) {
                                dir('media/application') {
                                    sh 'mvn -B dependency:copy-dependencies -DoutputDirectory=../../app/lib'
                                }
                            } else {
                                echo '已跳过后端第三方依赖复制'
                            }
                        }
                    )
                }
            }
        }

        stage('复制前端产物') {
            steps {
                script {
                    parallel(
                        '复制前端产物': {
                            dir('media/application/src/main/resources/static') {
                                unstash 'ui-dist'
                            }
                        },
                        '移回前端依赖': {
                            def nodeModulesDir = params.nodeModuleDir?.trim() ?: env.NODE_MODULE_DIR?.trim()
                            if (!nodeModulesDir) {
                                error('nodeModuleDir 参数或 NODE_MODULE_DIR 环境变量不能为空')
                            }
                            withEnv(["NODE_MODULES_DIR=${nodeModulesDir}"]) {
                                sh(
                                    label: '移回前端依赖目录',
                                    script: '''
                                        set -e
                                        if [ -d media-ui/node_modules ]; then
                                            mkdir -p "$(dirname "$NODE_MODULES_DIR")"
                                            if [ -e "$NODE_MODULES_DIR" ]; then
                                                echo "恢复依赖目录时目标位置已存在: $NODE_MODULES_DIR" >&2
                                                exit 1
                                            fi
                                            mv media-ui/node_modules "$NODE_MODULES_DIR"
                                        fi
                                    '''
                                )
                            }
                        }
                    )
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
                archiveArtifacts artifacts: 'app/**', fingerprint: params.fingerprint, allowEmptyArchive: false
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
