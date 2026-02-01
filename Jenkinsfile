// Jenkinsfile - CI/CD Pipeline for GKE Deployment
// 此 Pipeline 实现: GitHub 拉取代码 -> 构建 Docker 镜像 -> 推送到 Artifact Registry -> 部署到 GKE

pipeline {
    agent {
        label 'docker-builder'  // 使用动态创建的 Kubernetes Pod Agent
    }

    // 环境变量配置
    environment {
        // Google Artifact Registry 配置
        GCP_PROJECT = 'project-gcp-bigdata'
        GCP_REGION = 'asia-northeast1'
        REGISTRY_URL = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/iws-docker"

        // GKE 配置
        GKE_CLUSTER_NAME = 'gke-sws-cluster'
        GKE_NAMESPACE = 'iws'  // 应用部署的命名空间

        // Docker 镜像配置
        IMAGE_NAME = "${env.JOB_NAME}"  // 使用 Jenkins Job 名称作为镜像名
        IMAGE_TAG = "${env.BUILD_NUMBER}"  // 使用构建号作为镜像标签
        FULL_IMAGE_NAME = "${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    // Pipeline 参数 (可在 Jenkins UI 中配置)
    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git 分支名称')
        string(name: 'REPLICAS', defaultValue: '2', description: '副本数量')
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging', 'production'], description: '部署环境')
    }

    // Pipeline 阶段
    stages {
        // 阶段 1: 从 GitHub 拉取代码
        stage('Checkout Code') {
            steps {
                script {
                    echo "正在从 GitHub 拉取代码..."
                    echo "分支: ${params.GIT_BRANCH}"

                    // 使用 scm 变量自动获取配置的仓库信息
                    checkout scm

                    echo "代码拉取完成: 分支 ${params.GIT_BRANCH}"
                }
            }
        }

        // 阶段 2: 配置 Docker 认证
        stage('Configure Docker Auth') {
            steps {
                container('docker') {
                    script {
                        echo "配置 Artifact Registry 认证..."

                        // 使用 GKE 节点的 Service Account 进行认证
                        // 通过 Workload Identity 或节点 SA 自动获取凭证
                        sh """
                            # 获取 access token 并配置 Docker
                            TOKEN=\$(wget -q -O - --header="Metadata-Flavor: Google" \
                                http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token \
                                | sed 's/.*"access_token":"\\([^"]*\\)".*/\\1/')

                            echo "\$TOKEN" | docker login -u oauth2accesstoken --password-stdin https://${GCP_REGION}-docker.pkg.dev
                        """

                        echo "Artifact Registry 认证成功"
                    }
                }
            }
        }

        // 阶段 3: 构建 Docker 镜像
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    script {
                        echo "正在构建 Docker 镜像..."
                        echo "镜像名称: ${FULL_IMAGE_NAME}"

                        // 构建 Docker 镜像
                        sh """
                            docker build \
                                --build-arg BUILD_DATE=\$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
                                --build-arg VCS_REF=${env.GIT_COMMIT} \
                                --build-arg VERSION=${IMAGE_TAG} \
                                -t ${FULL_IMAGE_NAME} \
                                -f Dockerfile .
                        """

                        echo "Docker 镜像构建完成"
                    }
                }
            }
        }

        // 阶段 4: 推送镜像到 Artifact Registry
        stage('Push to Artifact Registry') {
            steps {
                container('docker') {
                    script {
                        echo "正在推送镜像到 Artifact Registry..."

                        // 推送镜像
                        sh "docker push ${FULL_IMAGE_NAME}"

                        // 同时打标签为 latest
                        sh """
                            docker tag ${FULL_IMAGE_NAME} ${REGISTRY_URL}/${IMAGE_NAME}:latest
                            docker push ${REGISTRY_URL}/${IMAGE_NAME}:latest
                        """

                        echo "镜像推送完成: ${FULL_IMAGE_NAME}"
                    }
                }
            }
        }

        // 阶段 5: 部署到 GKE
        stage('Deploy to GKE') {
            steps {
                container('kubectl') {
                    script {
                        echo "正在部署到 GKE 集群..."
                        echo "命名空间: ${GKE_NAMESPACE}"
                        echo "副本数: ${params.REPLICAS}"

                        // 检查 Deployment 是否存在
                        def deploymentExists = sh(
                            script: "kubectl get deployment ${IMAGE_NAME} -n ${GKE_NAMESPACE} --ignore-not-found",
                            returnStdout: true
                        ).trim()

                        if (deploymentExists) {
                            // 更新现有 Deployment
                            echo "更新现有 Deployment..."
                            sh """
                                kubectl set image deployment/${IMAGE_NAME} \
                                    ${IMAGE_NAME}=${FULL_IMAGE_NAME} \
                                    -n ${GKE_NAMESPACE} \
                                    --record

                                kubectl scale deployment/${IMAGE_NAME} \
                                    --replicas=${params.REPLICAS} \
                                    -n ${GKE_NAMESPACE}
                            """
                        } else {
                            // 创建新 Deployment
                            echo "创建新 Deployment..."
                            sh """
                                kubectl create deployment ${IMAGE_NAME} \
                                    --image=${FULL_IMAGE_NAME} \
                                    --replicas=${params.REPLICAS} \
                                    -n ${GKE_NAMESPACE}

                                # 暴露服务 (如果需要)
                                kubectl expose deployment ${IMAGE_NAME} \
                                    --type=ClusterIP \
                                    --port=8080 \
                                    --target-port=8080 \
                                    -n ${GKE_NAMESPACE} || true
                            """
                        }

                        // 等待部署完成
                        echo "等待 Deployment 就绪..."
                        sh """
                            kubectl rollout status deployment/${IMAGE_NAME} \
                                -n ${GKE_NAMESPACE} \
                                --timeout=5m
                        """

                        echo "部署完成！"
                    }
                }
            }
        }

        // 阶段 6: 验证部署
        stage('Verify Deployment') {
            steps {
                container('kubectl') {
                    script {
                        echo "验证部署状态..."

                        // 查看 Pod 状态
                        sh """
                            kubectl get pods -n ${GKE_NAMESPACE} \
                                -l app=${IMAGE_NAME} \
                                -o wide
                        """

                        // 查看 Service 状态
                        sh """
                            kubectl get svc ${IMAGE_NAME} -n ${GKE_NAMESPACE} || true
                        """

                        // 获取部署详情
                        def podCount = sh(
                            script: "kubectl get pods -n ${GKE_NAMESPACE} -l app=${IMAGE_NAME} --field-selector=status.phase=Running --no-headers | wc -l",
                            returnStdout: true
                        ).trim()

                        echo "当前运行的 Pod 数量: ${podCount}"

                        if (podCount.toInteger() < params.REPLICAS.toInteger()) {
                            error "部署失败: 期望 ${params.REPLICAS} 个副本, 实际运行 ${podCount} 个"
                        }
                    }
                }
            }
        }
    }

    // Pipeline 执行后操作
    post {
        success {
            echo """
            ========================================
            Pipeline 执行成功！
            ========================================
            应用名称:   ${IMAGE_NAME}
            镜像版本:   ${IMAGE_TAG}
            镜像地址:   ${FULL_IMAGE_NAME}
            部署环境:   ${params.DEPLOY_ENV}
            命名空间:   ${GKE_NAMESPACE}
            副本数量:   ${params.REPLICAS}
            Git 分支:   ${params.GIT_BRANCH}
            ========================================
            """
        }

        failure {
            echo """
            ========================================
            Pipeline 执行失败！
            ========================================
            应用名称: ${IMAGE_NAME}
            构建编号: ${env.BUILD_NUMBER}
            请检查日志排查问题
            ========================================
            """
        }

        always {
            container('docker') {
                // 清理本地 Docker 镜像 (节省空间)
                sh """
                    docker rmi ${FULL_IMAGE_NAME} || true
                    docker rmi ${REGISTRY_URL}/${IMAGE_NAME}:latest || true
                """
            }

            // 清理工作空间
            cleanWs()
        }
    }
}
