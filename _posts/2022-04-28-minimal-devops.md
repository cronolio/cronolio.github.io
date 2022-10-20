---
layout: default
title: devops на минималках - часть 1
descr: terraform на примере libvirt
---

Предположим уже есть настроенный некий хост с libvirt и 2мя сетевка. Первая смотрит в интернет,
вторая в локальную сеть. На внутреннем интерфейсе статичный ип-адрес `10.0.0.1`.

По [мануалке](https://github.com/dmacvicar/terraform-provider-libvirt/blob/main/README.md#getting-started) создадим файлик `main.tf`
и напишем в него так:

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

Далее нужно узнать `id` сетевых интерфейсов и хранилищь в libvirt.
