Vagrant.configure("2") do |config|
#  config.vm.provider "virtualbox" do |vb|
#    vb.check_guest_additions = false
#    vb.functional_vboxsf = false
#  end

# MASTER 
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
      echo "Jenkins container already exists. Starting..."
      docker start jenkins
  else
      echo "Creating Jenkins container..."
      docker run -d --name jenkins \
        -p 8080:8080 -p 50000:50000 \
        -v jenkins_home:/var/jenkins_home \
        --restart=always \
        jenkins/jenkins:lts
  fi
SHELL
  end

# WORKER
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
