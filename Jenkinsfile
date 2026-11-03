pipeline {
    agent { label 'docker-node' }

    environment {
        DOCKER_CREDS   = credentials('docker-hub-credentials')
        IMAGE_NAME     = "${DOCKER_CREDS_USR}/my-web-app"
        TEST_CONTAINER = "jenkins_test_web"
    }

    triggers {
        pollSCM('H/5 * * * *') 
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Перевіряю структуру файлів та папку з сайтом..."
                sh '''
                    pwd
                    ls -la
                    test -d html || (echo "Помилка: папка html відсутня!" && exit 1)
                    echo "Файли сайту на місці."
                '''
            }
        }

        stage('Test') {
            steps {
                echo "Перевіряю наявність конфігураційних файлів Docker..."
                sh '''
                    test -f Dockerfile || (echo "Помилка: Dockerfile відсутній!" && exit 1)
                    test -f docker-compose.yml || (echo "Помилка: docker-compose.yml відсутній!" && exit 1)
                    echo "Конфігураційні файли на місці."
                '''
            }
        }

        stage('Build') {
            steps {
                echo "Збираю новий Docker-образ Nginx..."
                sh """
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                    echo "Образ успішно зібрано."
                """
            }
        }

        stage('Docker') {
            steps {
                echo "Починаю тестування, пуш та розгортання..."
                sh """
                    echo "1. Зупиняю та видаляю старі контейнери..."
                    docker-compose down -v --remove-orphans || true
                    docker stop ${TEST_CONTAINER} || true
                    docker rm ${TEST_CONTAINER} || true

                    echo "2. Запускаю тестовий контейнер..."
                    docker run -d --name ${TEST_CONTAINER} -p 8086:80 ${IMAGE_NAME}:latest
                    sleep 3
                    curl -f http://localhost:8086 || (echo "Помилка: тестовий контейнер не відповідає!" && exit 1)
                    docker stop ${TEST_CONTAINER} && docker rm ${TEST_CONTAINER}

                    echo "3. Авторизуюсь та завантажую новий образ на Docker Hub..."
                    echo "${DOCKER_CREDS_PSW}" | docker login -u "${DOCKER_CREDS_USR}" --password-stdin
                    docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                    docker push ${IMAGE_NAME}:latest
                    docker logout

                    echo "4. Піднімаю весь веб-сервер через Docker Compose..."
                    DOCKER_HUB_USER=${DOCKER_CREDS_USR} docker-compose up -d --force-recreate
                    echo "Веб-сервер успішно запущено!"
                """
            }
        }
    }
}