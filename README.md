# myhome-ansible

## Описание
Этот репозиторий содержит Ansible-скрипты для развертывания инфраструктуры Docker Swarm и приложения myhome. Пожалуйста, убедитесь, что ваша версия Ansible равна или выше 2.9.

**Примечание:** Запуск из-под WSL/Cygwin не поддерживается. Рекомендуется использовать окружение без этих ограничений.

## Требования
- Ansible >= 2.9
- 5 машин

## Роли

1. **preconfig**
   - Устанавливает Docker, Docker Compose и другие зависимости.

2. **swarm_init**
   - Инициализация Docker Swarm на server1, сохранение токенов в факты.

3. **swarm_registry**
   - Поднятие локального registry.

4. **swarm_join**
   - Присоединение к Docker Swarm по сохраненным токенам.

5. **build**
   - Копирование бэкенда с гита, билд отдельных сервисов и добавление в registry.
   - Скачивание фронтенд-image с гита и добавление его в registry.

6. **deploy**
   - Поднятие всех сервисов, в т.ч. инфраструктурных - mysql, mongo, rmq.

**Примечание:** group_vars/all/vault.yml зашифрован. Вам нужен свой файл с переменными среды.

Полный запуск происходит по `ansible-playbook -i inventory all.yml -K`. Можно билдить (build), деплоить (deploy) отдельные сервисы по соответствующим тегам. `ansible-playbook -i inventory all.yml -K --tags "deploy"`

## Vagrantfile
Для удобного тестирования проекта предоставлен Vagrantfile, который позволяет быстро поднять виртуальные машины. Для этого выполните команду `vagrant up` для следующего Vagrantfile

```
ENV['VAGRANT_SERVER_URL'] = 'http://vagrant.elab.pro'
Vagrant.configure("2") do |config|
  (1..5).each do |i|
    config.vm.define "server#{i}" do |web|
      web.vm.box = "ubuntu/focal64"
      web.vm.network "forwarded_port", id: "ssh", host: 2222 + i, guest: 22
      web.vm.network "private_network", ip: "10.11.10.#{i}", virtualbox__intnet: true
      web.vm.hostname = "server#{i}"

      web.vm.provision "shell" do |s|
          ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
          s.inline = <<-SHELL
          echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
		      echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
          SHELL
      end

      web.vm.provider "virtualbox" do |v|
        v.name = "server#{i}"
        v.memory = 2048
        v.cpus = 1
      end
    end
  end
end

```