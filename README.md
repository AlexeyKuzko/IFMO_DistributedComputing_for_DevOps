# Распределенное веб-приложение с мониторингом
Этот проект представляет Ansible‑плейбук для пошагового развертывания WordPress веб-приложения с помощью Docker Compose и подключением мониторинга.
Сначала запускается однобазовая конфигурация. (ДЗ1)
Затем выполняется резервное копирование MySQL и приложение перезапускается в режиме кластера MySQL с включённой групповой репликацией данных. (ДЗ2)
В завершении к существующей системе подключается мониторинг Grafana + Prometheus c автоматическим импортированием шаблона дашборда для мониторинга базы данных. (ДЗ3)

## Структура проекта
```
IFMO_DistributedComputing_for_DevOps
├── group_vars
│   └── all.yml
├── roles
│   ├── homework1
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── docker-compose.yml.j2
│   ├── homework2
│   │   ├── files
│   │   │   └── primary_bootstrap.sql
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── docker-compose.yml.j2
│   ├── homework3
│   │   ├── files
│   │   │   └── dashboard.sql
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   │       └── compose-monitoring.yml.j2
│   │       └── grafana-dashboard.yml.j2
│   │       └── grafana-datasource.yml.j2
│   │       └── prometheus.yml.j2
├── inventory
│   └── hosts.ini
├── playbook.yml
└── README.md
```

## Требования
- Управляющий сервер (локальный компьютер):
  - Ansible [core 2.18.3] или новее
  - Community-плагины для Ansible:
  - * ansible-galaxy collection install community.docker
  - * ansible-galaxy collection install community.mysql
  - SSH-ключ для доступа к удаленному серверу
- Удаленный сервер:
  - Ubuntu/Debian Linux
  - Минимум 4 ГБ оперативной памяти для работы кластера базы данных
  - Docker и Docker Compose будут установлены автоматически

## Архитектура распределенного приложения
Система состоит из следующих компонентов:
- **WordPress**: Веб-приложение, работающее в отдельном контейнере
- **Кластер MySQL**: Три узла MySQL (1 первичный, 2 вторичных) с групповой репликацией
  - **Первичный узел**: Обрабатывает запросы на запись и чтение
  - **Вторичные узлы**: Обрабатывают запросы на чтение и служат резервными копиями
- **Сети Docker**: 
  - `my_app_wordpress-network`: Сеть для WordPress
  - `db_cluster_network`: Сеть для кластера базы данных
- **Система мониторинга**: Grafana с Prometheus в отдельных контейнерах и скраппингом в контейнере mysqld-exporter

# Инструкции по настройке
0. Убедитесь, что выполнены все требования (см выше)
1. Обновите файлы `inventory/hosts.ini` и `group_vars/all.yml` используя свои IP и учетные данные.
2. Клонируйте этот репозиторий на ваш локальный компьютер:
   ```bash
   git clone https://github.com/AlexeyKuzko/IFMO_DistributedComputing_for_DevOps.git
   ```
3. Перейдите в каталог проекта:
   ```bash
   cd IFMO_DistributedComputing_for_DevOps
   ```
4. Убедитесь, что у вас есть SSH-ключ для доступа к удаленному серверу:
   ```bash
   ls ~/.ssh/distributed_computing/private_key
   ```
5. Запустите плейбук для развертывания на удаленном сервере:
   ```bash
   ansible-playbook -i inventory/hosts.ini playbook.yml
   ```

## Использование
После запуска плейбука, будет развернуто и запущено WordPress веб-приложение, доступное по URI, указанному в inventory/hosts.ini (например, http://158.160.46.212:8080).
Мониторинг будет доступен по порту, указанному в inventory/hosts.ini (например, http://158.160.46.212:3000 / http://158.160.46.212:9090 / http://158.160.46.212:9104).

### Проверка состояния кластера
Вы можете управлять контейнерами с помощью команд Docker на удаленном сервере.
Для проверки состояния кластера MySQL вы можете выполнить следующие команды на удаленном сервере:
```bash
# Подключение ко вторичному узлу
docker exec -it mysql_secondary mysql -uroot -proot_password

# Внутри MySQL выполните:
SHOW REPLICA STATUS;
```

## Завершение работы
Для остановки веб-приложения, выполните:
```bash
sudo docker rm -f $(sudo docker ps -aq) ; sudo docker system prune -a -f; sudo docker volume prune -f
```
Для очистки вольюмов контейнеров, выполните:
```bash
sudo docker docker volume ls -q | xargs -r docker volume rm

```
Для очистки конфигурационных файлов, выполните:
```bash
sudo rm -rf /opt/my_app/
```