# Step project 2

---

# 1. Vagrantfile 

Vagrantfile створює **дві VM**:

### jenkins-master
- Ubuntu 22.04
- Встановлює Docker
- Запускає Jenkins у Docker
- Налаштовує автозапуск Jenkins

### jenkins-worker
- Ubuntu 22.04
- Встановлює Docker
- Створює користувача jenkins
- Дозволяє SSH-підключення для Jenkins master

Запускаємо:

```
vagrant up
```
Зміст файлу:

```ruby
Vagrant.configure("2") do |config|

# master (server) 
  config.vm.define "jenkins-master" do |master|
    master.vm.box = "bento/ubuntu-22.04"
    master.vm.hostname = "jenkins-master"
    master.vm.network "private_network", ip: "192.168.56.10"

    master.vm.provider "virtualbox" do |vb|
      vb.cpus = 4
      vb.memory = 2048
      vb.name = "jenkins-master"
    end

    master.vm.provision "shell", inline: <<-SHELL
  apt update -y
  apt install -y docker.io openjdk-17-jdk
  systemctl enable docker
  systemctl start docker

  if [ "$(docker ps -aq -f name=jenkins)" ]; then
      echo "container Jenkins exists, starting"
      docker start jenkins
  else
      echo "creating Jenkins container"
      docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home \
        --restart=always jenkins/jenkins:lts
  fi
SHELL
  end

# worker (agent)
  config.vm.define "jenkins-worker" do |worker|
    worker.vm.box = "bento/ubuntu-22.04"
    worker.vm.hostname = "jenkins-worker"
    worker.vm.network "private_network", ip: "192.168.56.11"

    worker.vm.provider "virtualbox" do |vb|
      vb.cpus = 4
      vb.memory = 2048
      vb.name = "jenkins-worker"
    end

    worker.vm.provision "shell", inline: <<-SHELL
      apt update -y
      apt install -y docker.io openjdk-17-jdk ssh
      systemctl enable docker
      systemctl start docker

      useradd -m -s /bin/bash jenkins
      echo "jenkins:jenkins" | chpasswd
      usermod -aG docker jenkins
    SHELL
  end

end

```

---

# 2. Dockerfile

Створюємо Docker образ застосунку на js та встановлюємо його залежності

Зміст файлу:
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

# 3. Jenkinsfile 

Порядок робити Pipline:

1. Клонує код із GitHub
2. Збирає Docker образ
3. Запускає тести у контейнері
4. Якщо все проходить успішно - відправляє образ у Docker Hub

Зміст файлу:

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

# 4. Підключаємось до Jenkins (master)

Після запуску в браузері перейти за адресою:

```
http://192.168.56.10:8080
```

Для отримання паролю підключаємось до vm:

```
vagrant ssh jenkins-master
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
копіюємо пароль та налаштовуюмо Jenkins

---

# 5. Налаштування Jenkins

1. Install suggested plugins  
2. Створити адміністратора  
3. Вказати Jenkins URL  
4. Додати Jenkins worker: Manage Jenkins - Nodes - New Node

---

# 6. Додавання Jenkins Worker

Параметри:

| Поле | Значення |
|------|----------|
| Host | 192.168.56.11 |
| Credentials | jenkins / jenkins |
| Remote root | /home/jenkins |

---

# 7. Додавання DockerHub credentials

Manage Jenkins - Credentials - Global - Add credentials

| Поле | Значення |
|------|----------|
| Kind | Username with password |
| ID | dockerhub |
| Username | ваш DockerHub логін |
| Password | ваш DockerHub пароль |

---

# 8. Як запустити pipeline

1. Створити новий Pipeline job  
2. Вказати, що Jenkinsfile лежить у репозиторії  
3. Натиснути **Build Now**

---
# 9. DockerHub

Після успішного завершення pipeline образ буде доступний за адресою:

*https://hub.docker.com/r/alexdevops05/step2/tags*

---