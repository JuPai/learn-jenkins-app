pipeline {
    agent any

    options {
        // 避免同一 job 併發時，兩個 run 互相踩同一個 workspace
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        // CRA / react-scripts 在 CI 需要這個，否則 npm test 會進 watch mode 卡住
        CI = 'true'
        NETLIFY_SITE_ID = '402308f3-0387-4a82-8a2a-bfff6095c34d'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {

        stage('Prepare') {
            steps {
                // 開頭就清掉上一輪的產物：
                // 不清的話，Build 失敗或被跳過時，後面仍會拿到舊的 build/ 而全綠
                // 注意：故意不清 node_modules（npm ci 自己會處理）
                //       也故意不清 .npm-cache-*（那是我們要保留的加速資產）
                sh 'rm -rf build playwright-report test-results junit-results'
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            environment {
                // npm 會讀取任何 npm_config_* 前綴的環境變數當作設定。
                // 指到 workspace 內：workspace 跨 run 持久、owner 正確，
                // 不需要在 host 上預先建目錄或處理權限。
                npm_config_cache = "${WORKSPACE}/.npm-cache-alpine"
            }
            steps {
                sh '''
                    echo "--- cache check ---"
                    npm config get cache
                    du -sh "$npm_config_cache" 2>/dev/null || echo "cache EMPTY (first run)"

                    node --version
                    npm --version

                    npm ci
                    npm run build

                    # 讓「沒建出來」直接讓 stage 紅掉，而不是印個字繼續往下跑
                    test -f build/index.html
                '''
            }
        }

        stage('Test') {
            parallel {

                stage('Unit Test') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    environment {
                        npm_config_cache = "${WORKSPACE}/.npm-cache-alpine"
                    }
                    steps {
                        sh 'npm test'
                    }
                    post {
                        always {
                            junit 'junit-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.55.0-noble'
                            reuseNode true
                        }
                    }
                    environment {
                        // 不同 libc（noble 是 glibc，alpine 是 musl），cache 分開放
                        npm_config_cache = "${WORKSPACE}/.npm-cache-noble"
                    }
                    steps {
                        sh '''
                            # npx --yes：跑完即走，不寫進 node_modules、
                            # 不改 package.json / package-lock.json，
                            # 因此不會跟同時執行的 Unit Test 互相干擾。
                            # 版本釘住，避免 latest 造成不可重現。
                            npx --yes serve@14 -s build -l 3000 &

                            # 取代 sleep 10：等到伺服器真的可連線，最多 60 秒
                            npx --yes wait-on@7 http://localhost:3000 -t 60000

                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: false,
                                icon: '',
                                keepAll: false,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'playwright HTML Report',
                                reportTitles: '',
                                useWrapperFileDirectly: true
                            ])
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                 
                    echo "Deploy site id $NETLIFY_SITE_ID"

                    node_modules/.bin/netlify status
                '''
            }
        }
    }
}
