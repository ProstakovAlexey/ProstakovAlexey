Установка яндекс танк
=====================
Удобный способ установить - через докер. Возможно он у вас не установлен.

1. Компоненты системы, скорее всего напишет что все уже стоит, кроме apt-transport-https
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common

2.Добавить ключ для репозитария 
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

3. sudo apt-key fingerprint 0EBFCD88

pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22

4. Добавляю репозитарий
udo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

5. Проверить какие версии пакетов есть apt-cache madison docker-ce

вероятно ставить надо самый последний sudo apt-get install docker-ce

6. проверка что все работает:

sudo docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.


sudo groupadd docker
sudo usermod -aG docker $USER




1. https://docs.docker.com/install/linux/docker-ce/ubuntu/#set-up-the-repository








https://yandextank.readthedocs.io/en/latest/install.html
