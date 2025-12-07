# Step2 - CI/CD (Jenkins + Docker + Node.js)

Step2 проєкт реалізує процес CI/CD для Node.js застосунку, включаючи:

- створення інфраструктури через **Vagrant**
- підняття **Jenkins Master (сервер) + Jenkins Worker (агент)**
- складання Docker-образу
- запуск Jest-тестів у контейнері
- завантаження в Docker Hub при успішному проходженні тестів
- автоматизований Jenkins Pipeline

---

## 1. Структура Step2

```
step2/
 ┣ index.js
 ┣ package.json
 ┣ app.test.js
 ┣ Dockerfile
 ┣ Jenkinsfile
 ┣ Vagrantfile
 ┣ README.md
 ┗ .vagrant/
```

---

# 2. Node.js застосунок

Express-сервер, який повертає відповідь:

```
GET /
→ "Hello Alex DevOps!"
```

Тести реалізовано через: jest та supertest

---

# 3. Dockerfile

```dockerfile
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

---

# 4. Запускаємо локально

```bash
npm install
npm start
npm test
```

---

# 5. Vagrant 

Створюємо **дві віртуальні машини**:

| Машина | IP | Призначення |
|--------|---------|-------------|
| jenkins-master | 192.168.56.10 | Сервер Jenkins |
| jenkins-worker | 192.168.56.11 | Jenkins SSH агент |

---

# 6. Налаштування Jenkins

1. Відкрийте панель:
   ```
   http://192.168.56.10:8080
   ```
2. Отримання паролю:
   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Створити **SSH agent node**
4. Створити **DockerHub credentials**:
   - ID: <ваш id_dockerhub>
   - Username: <ваш логин>
   - Password: <ваш пароль>
5. Створити *Pipeline Job*

---

# 7. Jenkins (Pipeline)

```groovy
pipeline {
    agent { label 'worker' }

    environment {
        DOCKERHUB = credentials('dockerhub')
        IMAGE_NAME = "alexdevops05/step2"
    }

    stages {

        stage('checkout code') {
            steps {
                echo "cloning github repository"
                git branch: 'main', url: 'https://github.com/allex-git/step2.git'
            }
        }

        stage('build image') {
            steps {
                echo "build image"
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('run tests') {
            steps {
                echo "run tests"
                sh "docker run --rm ${IMAGE_NAME}:latest npm test"
            }
        }

        stage('push to dockerHub') {
            when {
                expression { currentBuild.currentResult == "SUCCESS" }
            }
            steps {
                echo "push image to docker hub"
                sh '''
                    echo ${DOCKERHUB_PSW} | docker login -u ${DOCKERHUB_USR} --password-stdin
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }
    }

    post {
        success {
            echo "build and tests successfully"
        }
        failure {
            echo "tests failed"
        }
    }
}
```

---

# 8. DockerHub

Після успішного завершення pipeline образ буде доступний за шляхом:

*https://hub.docker.com/r/alexdevops05/step2/tags*

---

