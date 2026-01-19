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

        stage('Configure Admin') {
            steps {
                dir("${env.APP_DIR}") {
                    // Ждём запуска контейнера
                    sh 'sleep 10'

                    // Обновляем пароль администратора
                    sh """
                        docker exec file-exchange filebrowser users update admin \
                            --username ${FILEBROWSER_ADMIN_USERNAME} \
                            --password ${FILEBROWSER_ADMIN_PASSWORD} \
                            --perm.admin || echo "Admin user configuration skipped"
                    """
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
            echo "FileBrowser успешно развёрнут!"
            echo "Информация для доступа:"
            echo "URL: https://files.perek.rest"
            echo "Логин администратора: ${FILEBROWSER_ADMIN_USERNAME}"
            echo "Пароль администратора настроен из credentials"
            echo "ВАЖНО: Для безопасности смените пароль после первого входа!"
        }

        failure {
            echo "Деплой не удался. Проверьте логи для деталей."
        }
    }
}