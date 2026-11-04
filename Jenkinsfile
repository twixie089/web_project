pipeline {
    agent {
        label 'windows'
    }

    environment {
        DOCKER_BIN = 'C:\\Users\\pc\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        DOCKER_COMPOSE_BIN = 'C:\\Users\\pc\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker-compose.exe'
        DOCKER_HUB_USER = 'twixie089'
        IMAGE_NAME = "${DOCKER_HUB_USER}/my-web-app"
        TEST_CONTAINER = "jenkins_test_web"
        DOCKER_HUB_PASS = credentials('docker-hub-pass')
    }

    stages {
        stage('Checkout') {
            steps {
                powershell '''
                    Write-Output "Перевіряю наявність файлів сайту..."
                    Get-ChildItem
                    if (-Not (Test-Path html)) { Write-Error "Папка html відсутня!"; exit 1 }
                '''
            }
        }

        stage('Test') {
            steps {
                powershell '''
                    Write-Output "Перевіряю конфігураційні файли Docker..."
                    if (-Not (Test-Path Dockerfile)) { Write-Error "Dockerfile не знайдено!"; exit 1 }
                    if (-Not (Test-Path docker-compose.yml)) { Write-Error "docker-compose.yml не знайдено!"; exit 1 }
                '''
            }
        }

        stage('Build') {
            steps {
                powershell '''
                    Write-Output "Збираю Docker-образ для сайту..."
                    & "$env:DOCKER_BIN" build -t "${env:IMAGE_NAME}:${env:BUILD_NUMBER}" -t "${env:IMAGE_NAME}:latest" .
                '''
            }
        }

        stage('Docker') {
            steps {
                powershell '''
                    Write-Output "Зупиняю та видаляю старі контейнери..."
                    & "$env:DOCKER_COMPOSE_BIN" down -v --remove-orphans 2>$null; $LASTEXITCODE = 0
                    & "$env:DOCKER_BIN" stop ${env:TEST_CONTAINER} 2>$null; $LASTEXITCODE = 0
                    & "$env:DOCKER_BIN" rm ${env:TEST_CONTAINER} 2>$null; $LASTEXITCODE = 0

                    Write-Output "Тестую новий контейнер..."
                    & "$env:DOCKER_BIN" run -d --name ${env:TEST_CONTAINER} -p 8085:80 "${env:IMAGE_NAME}:latest"
                    Start-Sleep -Seconds 3
                    try { Invoke-WebRequest -Uri "http://localhost:8085" -UseBasicParsing } catch { Write-Error "Тестовий контейнер не відповідає!"; exit 1 }
                    & "$env:DOCKER_BIN" stop ${env:TEST_CONTAINER} 2>$null; & "$env:DOCKER_BIN" rm ${env:TEST_CONTAINER} 2>$null; $LASTEXITCODE = 0

                    Write-Output "Авторизуюсь та завантажую новий образ на Docker Hub..."
                    $env:DOCKER_HUB_PASS | & "$env:DOCKER_BIN" login -u "$env:DOCKER_HUB_USER" --password-stdin
                    & "$env:DOCKER_BIN" push "${env:IMAGE_NAME}:${env:BUILD_NUMBER}"
                    & "$env:DOCKER_BIN" push "${env:IMAGE_NAME}:latest"
                    & "$env:DOCKER_BIN" logout

                    Write-Output "Запускаю весь веб-сервер через Docker Compose..."
                    & "$env:DOCKER_COMPOSE_BIN" down -v --remove-orphans 2>$null; $LASTEXITCODE = 0
                    $env:DOCKER_HUB_USER="$env:DOCKER_HUB_USER"; & "$env:DOCKER_COMPOSE_BIN" up -d --force-recreate
                '''
            }
        }
    }
}
