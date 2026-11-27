pipeline {
    agent any

    environment {
        DEPLOY_DIR = 'C:\\ProgramData\\Jenkins\\deploy\\esports-devops'
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Получение исходного кода'
                checkout scm
                echo "Текущая ветка: ${env.GIT_BRANCH}"
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Установка зависимостей backend'

                bat 'python -m venv .venv'
                bat '.venv\\Scripts\\python.exe -m pip install -r requirements.txt'

                echo 'Установка зависимостей frontend'

                dir('client') {
                    bat 'npm ci'
                }
            }
        }

        stage('Tests') {
            steps {
                echo 'Запуск 24 автотестов'
                bat '.venv\\Scripts\\python.exe manage.py test'
            }
        }

        stage('Build') {
            steps {
                echo 'Сборка frontend'

                dir('client') {
                    bat 'npm run build'
                }
            }
        }

        stage('Deploy') {
            when {
                expression {
                    env.GIT_BRANCH == 'origin/main' ||
                    env.GIT_BRANCH == 'main'
                }
            }

            steps {
                echo 'Остановка старой версии приложения'

                powershell '''
                    $ErrorActionPreference = 'Stop'

                    $connections = Get-NetTCPConnection -LocalPort 8000,5173 -State Listen -ErrorAction SilentlyContinue

                    $processIds = $connections |
                        Select-Object -ExpandProperty OwningProcess -Unique

                    foreach ($processId in $processIds) {
                        $process = Get-CimInstance Win32_Process -Filter "ProcessId=$processId"

                        if ($process.CommandLine -notlike "*$env:DEPLOY_DIR*") {
                            throw "Port is occupied by another application: PID $processId"
                        }
                    }

                    foreach ($processId in $processIds) {
                        Stop-Process -Id $processId -Force
                    }
                '''

                echo 'Обновление рабочей версии приложения'

                bat '''
                    if not exist "%DEPLOY_DIR%" mkdir "%DEPLOY_DIR%"

                    robocopy "%WORKSPACE%" "%DEPLOY_DIR%" /MIR /R:1 /W:1 ^
                    /XD .git .venv __pycache__ .pytest_cache media logs ^
                    /XF db.sqlite3 db.sqlite3-journal db.sqlite3-wal db.sqlite3-shm *.pyc .env .env.*

                    if %ERRORLEVEL% GEQ 8 exit /b 1
                    exit /b 0
                '''

                echo 'Установка зависимостей рабочей версии'

                bat '''
                    cd /d "%DEPLOY_DIR%"
                    if not exist ".venv\\Scripts\\python.exe" (
                        python -m venv .venv || exit /b 1
                    )
                    .venv\\Scripts\\python.exe -m pip install -r requirements.txt
                '''

                echo 'Применение миграций базы данных'

                bat '''
                    cd /d "%DEPLOY_DIR%"
                    .venv\\Scripts\\python.exe manage.py migrate
                '''

                echo 'Запуск backend и frontend'

                powershell '''
                    $ErrorActionPreference = 'Stop'
                    $env:JENKINS_NODE_COOKIE = 'dontKillMe'

                    $dir = $env:DEPLOY_DIR

                    New-Item -ItemType Directory -Force "$dir\\logs" | Out-Null

                    Start-Process `
                        -FilePath "$dir\\.venv\\Scripts\\python.exe" `
                        -ArgumentList "$dir\\manage.py runserver 127.0.0.1:8000 --noreload" `
                        -WorkingDirectory $dir `
                        -WindowStyle Hidden `
                        -RedirectStandardOutput "$dir\\logs\\backend.log" `
                        -RedirectStandardError "$dir\\logs\\backend-error.log"

                    Start-Process `
                        -FilePath 'node.exe' `
                        -ArgumentList "$dir\\client\\node_modules\\vite\\bin\\vite.js --host 127.0.0.1 --port 5173 --strictPort" `
                        -WorkingDirectory "$dir\\client" `
                        -WindowStyle Hidden `
                        -RedirectStandardOutput "$dir\\logs\\frontend.log" `
                        -RedirectStandardError "$dir\\logs\\frontend-error.log"
                '''

                echo 'Проверка доступности сайта'

                powershell '''
                    for ($i = 0; $i -lt 30; $i++) {
                        try {
                            $frontend = Invoke-WebRequest -Uri 'http://localhost:5173/' -UseBasicParsing -TimeoutSec 3

                            $backend = Invoke-WebRequest -Uri 'http://localhost:5173/admin/login/' -UseBasicParsing -TimeoutSec 3

                            if ($frontend.StatusCode -eq 200 -and $backend.StatusCode -eq 200) {
                                Write-Output 'Application is running'
                                exit 0
                            }
                        }
                        catch {
                            Start-Sleep -Seconds 2
                        }
                    }

                    throw 'Application startup failed'
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline для ${env.GIT_BRANCH} успешно завершен"
        }

        failure {
            echo "Pipeline для ${env.GIT_BRANCH} завершен с ошибкой"
        }
    }
}