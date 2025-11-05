Установка NFS клиента:
```bash
sudo apt install nfs-common
```

Проверка что монтирование работает:
```bash
sudo mount -t nfs 172.10.10.10:/data /mnt
df -h /mnt
```

Команда `df` должна будет отобразить общий объем памяти равыный примонтированному диску, в данном случае 40GB.

Затем отмонтировать диск:
```bash
sudo umount /mnt
```

Установка NFS CSI, как по [гайду](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-master.md):
```bash
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/install-driver.sh | bash -s master --
```
Другие драйвера можно посмотреть в [документации](https://kubernetes.io/docs/concepts/storage/storage-classes/#nfs).

> Оговорюсь, что для установки драйвера требуется пулить образы с registry.k8s.io, который не работает в России, ну или, во всяком случае, не работает у меня. Я решил проблему тем, что пустил трафик с виртуалок через vpn. Но есть и другой вариант - развернуть прокси, например [Harbor proxy cache](https://goharbor.io/docs/2.1.0/administration/configure-proxy-cache/) и пустить только его трафик через vpn. Но к такому методу я прибегать не стал, хоть и начал, в связи с ограниченностью ресурсов.