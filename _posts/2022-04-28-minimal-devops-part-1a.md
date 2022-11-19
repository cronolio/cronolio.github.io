---
layout: post
sitemap:
  lastmod: 2022-11-19
title: devops на минималках - terraform на примере libvirt
descr: terraform на примере libvirt
keywords: terraform, libvirt
---

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

Далее нужно узнать `id` сетевых интерфейсов и хранилищь в libvirt. Пусть нам libvirt об этом расскажет.
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
