---
layout: post
sitemap:
  lastmod: 2022-12-23
title: devops на минималках - terraform на примере libvirt
descr: terraform на примере libvirt
keywords: terraform, libvirt
---

#### Базовые настройки terraform

Предположим уже есть некий настроенный хост с libvirt и 2мя сетевыми интерфейсами.
Первая br0 смотрит в интернет, вторая br1 в локальную сеть.
На внутреннем интерфейсе статичный ип-адрес `10.0.0.1`.

По [мануалке](https://github.com/dmacvicar/terraform-provider-libvirt/blob/main/README.md)
создадим файлик `main.tf` и напишем в него так:

```
terraform {
  required_providers {
    libvirt = {
      source = "dmacvicar/libvirt"
      version = "~> 0.6.14"
    }
  }
}

provider "libvirt" {
  uri = "qemu+ssh://root@10.0.0.1/system"
}
```

Если во время `terraform init` будет "подвисать", то можно настроить 
[зеркало](https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-quickstart#configure-provider)
в `~/.terraformrc`.

#### Импорт имеющейся инфрастуктуры из libvirt

Далее нужно узнать `id` сетевых интерфейсов и хранилищ в libvirt. Пусть нам libvirt об этом расскажет.
Для сетевых интерфейсов:
```
virsh net-info br0;
virsh net-info br1;
```
Для пулов:
```
virsh pool-info default
```
Нужно запомнить строки `Name` и `UUID`.

Далее создадим файлик сетевых ресурсов `libvirt_network.tf` с содержимым:
```
resource "libvirt_network" "br0" {
}
resource "libvirt_network" "br1" {
}
```

И для ресурсов дискового пространства `libvirt_pool.tf`:
```
resource "libvirt_pool" "default" {
}
```

Теперь можно импортировать ресурсы. UUID берем из предыдущих virsh-команд:
```
terraform import libvirt_network.br0 <UUID>
terraform import libvirt_network.br1 <UUID>
terraform import libvirt_pool.default <UUID>
```

Чтож, теперь существующие ресурсы libvirt импортированы в статус terraform.
Теперь нужно записать их в конфигурационные файлы.
Команда `terraform state list` покажет существующие в terraform ресурсы.
Команда `terraform state show libvirt_network.br0` покажет содержимое ресурса.
Перенести можно примерно такой командой:
```
terraform state show libvirt_network.br0 | sed 's/id/#id/' > libvirt_network.tf
terraform state show libvirt_network.br1 | sed 's/id/#id/' >> libvirt_network.tf
```

id комментируется, так как оно не может хранится в конфиг-файле.
Аналгично перенесем pool: 
```
terraform state show libvirt_pool.default | sed 's/id/#id/' > libvirt_pool.tf
```

Ура. Ресурсы в конфиг-файлах.
Настало время объявить свои ресурсы, создать виртуальные машины в libvirt при помощи terraform.

###### Если хочется удалить импортированные ресурсы из terraform
То достаточно посмотреть список при помощи: 
```
terraform state list
libvirt_network.br0
libvirt_network.br1
libvirt_pool.default
``` 

и удалить ненужный
```
terraform state rm <name>
```

и затем удалить/переименовать/закоментировать соответсвующие файлы. Эта команда удалит именно из terraform.

#### Объявление своих ресурсов в terraform
Далее берется некий `cloud`-образ, образ диска с уже установленным дистрибутивом.
Запишем в файлик `upstream_debian_volume.tf` такое:
```
resource "libvirt_volume" "debian-11-upstream" {
  name = "debian-11-upstream.qcow2"
  pool = "default"
  source = "https://cloud.debian.org/images/cloud/bullseye/20221020-1174/debian-11-genericcloud-amd64-20221020-1174.qcow2"
  format = "qcow2"
}
```

Тут лучше использовать ссылку на конкретный образ нежели чем `latest`.
Со временем можно обновлять по необходимости. `terraform apply` выкачает образ.

Далее настроим виртуальную машину. Роутер для маршрутизации в докер.
В вашем случае такое может не понадобится. Всё нижеследующее просто для примера

Напишем файл `swarmgw.tf`. Ниже я разберу его по частям.
```
# resize hdd
resource "libvirt_volume" "swarmgw" {
  name = "swarmgw.qcow2"
  base_volume_id = libvirt_volume.debian-11-upstream.id
  pool = "default"
  size = 3221225472 # 3GB
}
```

Объявляется ресурс `libvirt_volume` `swarmgw` (это в контексте terraform).
Физически же он ляжет в пул `default` под именем `swarmgw.qcow2`.
Для этого апстримный образ и увеличится размер диска до 3GB (размер указывается в байтах).
Размер не может быть уменьшен.

Далее идут команды после старта системы

```
resource "libvirt_cloudinit_disk" "init-swarmgw" {
  name = "init-swarmgw.iso"
  pool = "iso"
  user_data = <<EOF
#cloud-config
growpart:
  mode: auto
  device: ['/']
users:
  - name: <user_name>
    lock_passwd: true
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
write_files:
  - path: /etc/systemd/network/ens3.network
    content: |
      [Match]
      Name=ens3
      [Network]
      DHCP=ipv4
  - path: /etc/systemd/network/ens4.network
    content: |
      [Match]
      Name=ens4
      [Network]
      Address=10.0.0.120/24
  - path: /home/alex/.ssh/authorized_keys
    content: |
      ssh-rsa <user_public_key>
package_update: true
hostname: swarmgw
runcmd:
  - DEBIAN_FRONTEND=noninteractive apt install -y qemu-guest-agent iptables-persistent
  - systemctl disable ifup@ens3 ifup@ens4
  - systemctl disable ifupdown-pre
  - DEBIAN_FRONTEND=noninteractive apt remove -y ifupdown resolvconf
  - systemctl disable networking
  - systemctl disable cloud-init-local cloud-init cloud-config cloud-final
  - systemctl enable systemd-networkd
  - systemctl enable systemd-resolved
  - ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
  - sed -i /etc/systemd/resolved.conf -e 's/#DNSSEC=no/DNSSEC=yes/' -e 's/#MulticastDNS=yes/MulticastDNS=no/' -e 's/#LLMNR=yes/LLMNR=no/'
  - sed -i /etc/ssh/sshd_config -e 's/#AddressFamily any/AddressFamily inet/' -e 's/#ListenAddress 0.0.0.0/ListenAddress 10.0.0.120/'
  - sed -i /etc/default/chrony -e 's/DAEMON_OPTS="-F 1"/DAEMON_OPTS="-4 -F 1"/'
  - echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
  - iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
  - iptables -A FORWARD -i ens3 -o ens4 -m state --state ESTABLISHED,RELATED -j ACCEPT
  - iptables-save > /etc/iptables/rules.v4
  - reboot
EOF
}

```

Для полного понимая нужно посмотреть на
[документацию](https://cloudinit.readthedocs.io/en/latest/topics/examples.html)
Основное - заменить всё что внутри `<>` (узернэйм и его публичный ключ),
`growpart` - расширяет файловую систему после увеличения диска,
`users` - создает пользователя,
а `lock_passwd` блокирует логин по паролю (у рут пользователя так же заблокирован пароль).

Далее я просто делаю минимальные настройки операционной системы и настраиваю простой nat.

Ниже объявление самой виртуальной машины 
(ну то что мы обычно кликаем в интерфейсе пользователя - память, цпу, ...).

```
resource "libvirt_domain" "swarmgw" {
  name   = "swarmgw"
  memory = "456"
  vcpu   = 1
  qemu_agent = true
  running    = true

  cloudinit = libvirt_cloudinit_disk.init-swarmgw.id

  cpu {
    mode = "host-passthrough"
  }

  video {
    type = "qxl"
  }

  network_interface {
    network_id = libvirt_network.br0.id
  }
  network_interface {
    network_id = libvirt_network.br1.id
  }

  boot_device {
    dev = [ "hd" ]
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  disk {
    volume_id = libvirt_volume.swarmgw.id
  }
}
```

После terraform apply создастся 3 ресурса, виртуальная машина, диск, и iso-образ.
Посмотреть их можно как обычно в `terraform state list`:
```
libvirt_cloudinit_disk.init-swarmgw
libvirt_domain.swarmgw
libvirt_volume.swarmgw
```

Удалить виртуальную машину можно через `terraform destroy --target <ресурс>`.
Эта команда удалит из терраформ и из libvirt.

На этом всё. Продолжение следует...
