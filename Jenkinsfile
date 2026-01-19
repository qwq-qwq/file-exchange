pipeline {
    agent any

    environment {
        FILEBROWSER_ADMIN_USERNAME = credentials('filebrowser-admin-username')
        FILEBROWSER_ADMIN_PASSWORD = credentials('filebrowser-admin-password')
        APP_DIR = "/opt/projects/file-exchange"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Configuration') {
            steps {
                script {
                    // Создаём директории
                    sh "mkdir -p ${env.APP_DIR}/data ${env.APP_DIR}/config"

                    // Копируем файлы
                    sh "cp config/settings.json ${env.APP_DIR}/config/settings.json || true"
                    sh "cp docker-compose.yml ${env.APP_DIR}/docker-compose.yml || true"

                    // Даём права на запись в директории для FileBrowser контейнера
                    sh "chmod -R 777 ${env.APP_DIR}/config ${env.APP_DIR}/data"
                }
            }
        }

        stage('Initialize Database') {
            steps {
                dir("${env.APP_DIR}") {
                    script {
                        // Проверяем существует ли база данных
                        def dbExists = sh(
                            script: "test -f ${env.APP_DIR}/config/filebrowser.db && echo 'exists' || echo 'not_exists'",
                            returnStdout: true
                        ).trim()

                        if (dbExists == 'not_exists') {
                            echo "База данных не существует. Инициализируем с заданными credentials..."

                            // Запускаем временный контейнер для инициализации базы
                            sh """
                                docker run --rm \
                                    -v ${env.APP_DIR}/config:/config \
                                    -v ${env.APP_DIR}/data:/srv \
                                    filebrowser/filebrowser:latest \
                                    config init --database /config/filebrowser.db --config /config/settings.json
                            """

                            // Создаём пользователя admin с заданными credentials
                            sh """
                                docker run --rm \
                                    -v ${env.APP_DIR}/config:/config \
                                    -v ${env.APP_DIR}/data:/srv \
                                    filebrowser/filebrowser:latest \
                                    users add ${FILEBROWSER_ADMIN_USERNAME} ${FILEBROWSER_ADMIN_PASSWORD} \
                                    --database /config/filebrowser.db \
                                    --perm.admin
                            """

                            echo "База данных инициализирована с пользователем: ${FILEBROWSER_ADMIN_USERNAME}"
                        } else {
                            echo "База данных уже существует. Пропускаем инициализацию."
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                dir("${env.APP_DIR}") {
                    // Проверяем существование сети traefik_web-network
                    sh '''
                        if ! docker network ls | grep -q traefik_web-network; then
                            docker network create traefik_web-network
                        fi
                    '''

                    // Деплоим через Docker Compose
                    sh 'docker-compose down || true'
                    sh 'docker-compose up -d'
                }
            }
        }

        stage('Verify') {
            steps {
                dir("${env.APP_DIR}") {
                    // Проверяем что контейнер запущен
                    sh 'docker-compose ps'

                    // Ждём запуска сервиса
                    sh 'sleep 5'

                    // Проверяем что сервис отвечает
                    sh 'docker-compose logs'
                }
            }
        }
    }

    post {
        success {
            echo "==========================================="
            echo "FileBrowser успешно развёрнут!"
            echo "==========================================="
            echo "Информация для доступа:"
            echo "  URL: https://files.perek.rest"
            echo "  Логин: ${FILEBROWSER_ADMIN_USERNAME}"
            echo "  Пароль: настроен из Jenkins credentials"
            echo ""
            echo "Credentials настроены в jenkins-installer/.env"
            echo "==========================================="
        }

        failure {
            echo "Деплой не удался. Проверьте логи для деталей."
        }
    }
}