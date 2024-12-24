# cvat-yolov8-tutorial

## Первые шаги

* Установка WSL2 (подсистема Windows для Linux) см. в [этом официальном руководстве](https://docs.microsoft.com/windows/wsl/install-win10). WSL2 требует Windows 10 версии 2004 или выше. После установки WSL2 установите дистрибутив Linux по вашему выбору.

* Загрузите и установите [Docker Desktop для Windows](https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe). Дважды щелкните `Docker for Windows Installer`, чтобы запустить установщик. Дополнительные инструкции можно найти [здесь](https://docs.docker.com/docker-for-windows/install/). Официальное руководство по бэкэнду Docker WSL2 можно найти [здесь](https://docs.docker.com/docker-for-windows/wsl/). Примечание: убедитесь, что вы используете бэкэнд WSL2 для Docker.

* В Docker Desktop перейдите в «Настройки >> Ресурсы >> Интеграция с WSL» и включите интеграцию с выбранным вами дистрибутивом Linux.

* Загрузите и установите [Git для Windows](https://github.com/git-for-windows/git/releases/download/v2.21.0.windows.1/Git-2.21.0-64-bit.exe). При установке пакета оставьте все параметры по умолчанию. Более подробную информацию о пакете можно найти [здесь](https://gitforwindows.org).

* Загрузите и установите [Google Chrome](https://www.google.com/chrome/). Это единственный браузер, поддерживаемый CVAT.

* Перейдите в меню Windows, найдите установленный вами дистрибутив Linux и запустите его. Вы должны увидеть окно терминала.

* Клонируйте исходный код _CVAT_ из [репозитория GitHub](https://github.com/cvat-ai/cvat).

Следующая команда клонирует последнюю ветку develop:

```shell
git clone https://github.com/cvat-ai/cvat
cd cvat
```

См. [альтернативы](#how-to-get-cvat-source-code), если вы хотите загрузить одну из версий релиза.

Выберите имя пользователя и пароль для вашей учетной записи администратора. Для получения дополнительной информации см. [документацию Django](https://docs.djangoproject.com/en/2.2/ref/django-admin/#createsuperuser).

* Откройте установленный браузер Google Chrome и перейдите по адресу [localhost:8080](http://localhost:8080). Введите логин/пароль для суперпользователя на странице входа и нажмите кнопку _Войти_. Теперь вы сможете создать новую задачу аннотации. Более подробную информацию можно найти в [руководстве по CVAT](../../../../docs/manual/).

## Установка serverless версии

В каталоге CVAT запустите:

* Сначала остановите все контейнеры, если таковые имеются.

```
docker compose down
```

* Запустите CVAT вместе с плагином, используемым для помощника по автоматической аннотации AI.

```
docker compose -f docker-compose.yml -f components/serverless/docker-compose.serverless.yml up -d
```
* Создайте учетную запись
```
docker exec -it cvat_server bash -ic 'python3 ~/manage.py createsuperuser'
```

* Установите `nuctl`*

```
wget https://github.com/nuclio/nuclio/releases/download/<version>/nuctl-<version>-linux-amd64
```

* После загрузки nuclio дайте ему соответствующие права и сделайте мягкую ссылку.* (не знаю как это будет работать в WSL2)

```
sudo chmod +x nuctl-<version>-linux-amd64
sudo ln -sf $(pwd)/nuctl-<version>-linux-amd64 /usr/local/bin/nuctl
```

## Установка расширения для YOLOv8

* Клонируйте репозиторий с расширением куда-нибудь
```
git clone https://github.com/kurkurzz/custom-yolov8-auto-annotation-cvat-blueprint.git
```

### Конфигурация

* замените `custom-yolov8n.pt` на вашу модель

* в файле `funciton.yaml` измените
```yaml
spec: |
      [
        {"id": 0, "name": "tench"},
        {"id": 1, "name": "goldfish"},
        .
        .
        .
        .
        {"id": 998, "name": "ear"},
        {"id": 999, "name": "toilet paper"}
      ]
```

на свои id и name. Например:
```yaml
spec: |
      [
        {"id": 0, "name": "bus", "type": "rectangle"},
        {"id": 1, "name": "car", "type": "rectangle"},
        {"id": 2, "name": "motorbike", "type": "rectangle"},
        {"id": 3, "name": "road_train", "type": "rectangle"},
        {"id": 4, "name": "truck", "type": "rectangle"}
      ]
```

Получить отображение id в name:
```python
from ultralytics import YOLO
model = YOLO("function.yaml")
model.names
```

* Создайте образ docker и запустите контейнер. После этого вы можете сразу использовать модель в CVAT.
```
./serverless/deploy_cpu.sh путь/к/папке/с/расширением
```

* Устранение неполадок
- Проверьте, что ваша функция `nuclio` работает правильно
`docker ps --filter NAME=custom-model-yolov8`:
```
     CONTAINER ID   IMAGE                        COMMAND       CREATED          STATUS                    PORTS                                         NAMES
     3dc54494bbb8   custom-model-yolov8:latest   "processor"   52 minutes ago   Up 52 minutes (healthy)   0.0.0.0:32896->8080/tcp, :::32896->8080/tcp   nuclio-nuclio-custom-model-yolov8
```
- Если он не работает, проверьте журнал контейнера, он может показать, что идет не так:
`docker logs 3dc54494bbb8` (CONTAINER ID)
```
     ...
     24.01.31 12:28:35.522                 processor (D) Processor started
     24.01.31 12:29:27.171 sor.http.w0.python.logger (I) Run custom-model-yolov8 model {"worker_id": "0"}
```
- При установке на WSL2 следует обновить файл ".wslconfig" WLS2, чтобы разрешить WSL не менее 6 ГБ ОЗУ. Поместите файл .wslconfig в папку C:\/Users\/\<имя_пользователя\>. Если DOCKER часто вылетает или автоматическая аннотация не работает сразу после запуска, это, вероятно, из-за нехватки ОЗУ. Используйте альтернативную функцию, если установка не удалась (сработало для установки CVAT-2.16.2 в WSL2, с 6 ГБ, разрешенными для WSL2).

Примечание: * — это одноразовый шаг.

## Конфигурация
