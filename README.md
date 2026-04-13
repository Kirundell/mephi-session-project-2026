## Среда выполнения

Задание выполнено в **WSL2** (Windows Subsystem for Linux) с дистрибутивом **Ubuntu 22.04 LTS** вместо Fedora 43.
Ядро: `Linux 6.6.87.2-microsoft-standard-WSL2`.

## Адаптации под Ubuntu/WSL

### Пакетный менеджер

| Fedora | Ubuntu |
|---|---|
| `dnf install` | `apt install` |
| `dnf download` (RPM) | `apt download` (DEB) |
| `rpm -i` | `dpkg -i` |
| `libcap-ng-utils` | `libcap2-bin` (содержит `getcap`, `setcap`) |

Файл `tcpdump.rpm` в репозитории - это переименованный `.deb`-пакет для соответствия таблице артефактов.

### Файловая система (Раздел 3.1)

В WSL отсутствует физический диск `/dev/sdb`. Вместо него использован файл-образ через loop-устройство:

```
dd if=/dev/zero of=/opt/sdb.img bs=1M count=100
losetup /dev/loop0 /opt/sdb.img
parted /dev/loop0 mklabel gpt
parted /dev/loop0 mkpart primary ext4 0% 100%
mkfs.ext4 -L MEPHI_DATA /dev/loop0p1
```

Раздел смонтирован в `/data/mephi-web` через `/etc/fstab` по метке `LABEL=MEPHI_DATA`.

### Сеть (Раздел 1)

Статический IP `192.168.1.100/24` настроен через `nmcli`. В WSL сеть управляется Hyper-V, поэтому:

- `ping 192.168.1.1` - **Destination Host Unreachable** (шлюз физически не существует)
- `ping 8.8.8.8` - **работает** (через реальный маршрут WSL)
- `curl http://192.168.1.100` - **работает** (nginx слушает на этом IP)

### SELinux (Раздел 4.2)

WSL использует собственное ядро Microsoft без поддержки SELinux. Результат:

- `getenforce` -> `Disabled`
- `semanage fcontext` -> `ValueError: SELinux policy is not managed or store cannot be accessed.`
- `restorecon -Rv` - выполняется без ошибок, но не применяет контекст
- `ls -Zd /data/mephi-web` -> `? /data/mephi-web` (контекст отсутствует)

Команды выполнены для демонстрации знания процедуры.

### NetworkManager

По умолчанию в WSL/Ubuntu NetworkManager не управляет интерфейсами (плагин `ifupdown` + системный конфиг `unmanaged-devices=*`). Для работы `nmcli` потребовалось:

1. Установить `network-manager`
2. Создать `/etc/NetworkManager/conf.d/10-globally-managed-devices.conf` с `unmanaged-devices=none`

## Что работает полностью

- Настройка статического IP и hostname через `nmcli` / `hostnamectl`
- Установка пакетов (`nginx`, `tcpdump`, `libcap2-bin`)
- Создание файловой системы ext4, монтирование, `/etc/fstab`
- Управление сервисами (`systemctl enable/start nginx`)
- Журналирование (`journalctl -u nginx`)
- DAC: пользователь, группа, права 2775 (setgid)
- Capabilities (`setcap cap_net_raw,cap_net_admin+ep`)
- PAM-ограничение входа root (`pam_listfile.so`)
- nginx отдаёт страницу `Hello from Student: 368860`
