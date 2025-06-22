# Ansible_lab

## Настройка nginx в VM на сервере с помощью Ansible. 

# Теория 

Ansible — это инструмент для автоматизации:

- установки ПО

- настройки серверов

- управления конфигурациями

Особенности:

- Работает по SSH, не требует установки на удалённую машину

- Использует YAML-файлы — playbooks

- Поддерживает идемпотентность (повторный запуск не сломает систему)

# Начало

В лабе по Terraform я уже создал готовую ВМ, которая хостится в Yandex Cloud. Теперь мы сможем настроить её с помощью Ansible. 

Для начала продублирую файл main.tf из лабы по tf (есть незначительные изменения в виде названий): 

```
resource "yandex_compute_instance" "web" {
  name = "ansible-web"
  zone = "ru-central1-a"

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = "fd8e2c31dqpv3dcu1bqg" 
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.default.id
    nat       = true
  }

  metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_rsa.pub")}"
  }
}

```

Здесь мы создавали ВМ, выделяли ей ресурсы и для подключения создавали shh публичный ключ, теперь мы сможем его использовать.

После скачаем Ansible `sudo apt install ansible -y` и создадим структуру проекта:

```
mkdir ansible-lab && cd ansible-lab
touch inventory.ini
touch playbook.yml
mkdir files && echo "<h1>Hello from Ansible!</h1>" > files/index.html
```

Html файл простенький - мы все таки тут девопсом занимаемся)

# inventory.ini

inventory - это список хостов (серверов), с которыми будет работать Ansible.

inventory.ini

```
[web]
<external ip> ansible_user=debian ansible_ssh_private_key_file=~/.ssh/id_rsa

```

Указываем название ресурсы tf сверху, потом указываем публичный адрес, который мы можем найти либо в выводе файла output.tf из лабы по tf, либо прямо в облаке, где развернута наша ВМ:

![image](https://github.com/user-attachments/assets/9250510e-e2b2-4ec1-928a-473bbfd05ae9)

а также юзера и shh ключ для подключения из файла `~/.ssh/id_rsa.pub`. 

Проверить подключения можно командой `ansible -i inventory.ini all -m ping`:

![image](https://github.com/user-attachments/assets/4f7d5e74-cd38-4d72-97eb-d2af3fc764c6)


# playbook.yml

playbook - файл с задачами, которые нужно выполнить. Написан на YAML.

playbook.yml

```
- name: Установить nginx и развернуть сайт
  hosts: web
  become: true
  tasks:
    - name: Установить nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Скопировать index.html
      copy:
        src: ./index.html
        dest: /var/www/html/index.html

    - name: Убедиться, что nginx запущен
      service:
        name: nginx
        state: started
        enabled: yes

```

Элементы:

play - имя сценария: Указывается имя задачи

Целевые хосты:  Указывается, к каким серверам будет применяться сценарий. В данном случае — к группе web, которая должна быть определена в  в inventory.ini

Повышение привилегий (bceome): Указывается, что все действия должны выполняться с правами администратора, потому что установка программ и управление службами требует таких прав.

Список задач (tasks)

- Первая задача: установить Nginx на удалённой машине через системный пакетный менеджер.

- Вторая задача: скопировать HTML-файл (страницу сайта) с локальной машины на сервер в директорию, откуда Nginx будет его отдавать.

- Третья задача: убедиться, что служба Nginx запущена и настроена на автоматический запуск при перезагрузке системы.

# Запуск 

`ansible-playbook -i inventory.ini playbook.yml`

Запускаем playbook Ansible. 

![image](https://github.com/user-attachments/assets/cea7244a-fd05-4495-b665-be87868d9c75)

Кажется, что все работает. Проверим. Переходим по external ip:

![image](https://github.com/user-attachments/assets/e14b6805-e810-43c0-8b11-3fda68fd54b0)

Видим html-страничку, настроенную ранее. Победа!

# Заключение

В данной работе была успешно выполнена интеграция Terraform и Ansible для автоматизированного развертывания инфраструктуры и настройки веб-сервера. С помощью Terraform был создан виртуальный сервер в Yandex Cloud, а с помощью Ansible — установлен веб-сервер Nginx и размещена HTML-страница. Такой подход показывает удобство IaC, что позволяет быстро и повторяемо развертывать рабочие среды.

Стоит скачать лишь, что необходимо менять external ip в файле invetory, так как при запуске он может меняться, но на реальных серверах это реже. Можно написать скрипт. 

Спасибо за прочтение!



