# funbiscuit_platform
funbiscuit Platform repository

# ДЗ 1
В kubernetes-intro добавил Dockerfile и nginx.conf для сборки образа веб-сервера, выдающего
файлы из директории /app.

Для запуска веб-сервера сделан манифест пода web-pod.yaml.

Для проекта Hipster Shop сделан манифест пода frontend: frontend-pod.yaml. Для запуска
потребовались дополнительные переменные среды (адреса других микросервисов), их добавил
в манифест frontend-pod-healthy.yaml.

# ДЗ 2
В kubernetes-controllers добавил конфиг для локального кластера kind из 4 нод.
Запустил кластер локально, убедился, что есть 3 worker ноды и одна с ролью control-plane.

Попробовал создать replicaset для запуска frontend микросервиса. Манифест из задания содержал
ошибку - отсутствовал селектор, из-за чего ReplicaSet нельзя создать. Также отсутствовали
настройки переменных среды. После добавления селектора и переменных среды ReplicaSet создался,
а под перешел в состояние Ready.

Увеличил количество реплик до 3 командой kubectl scale, число подов увеличилось до трёх.

После удаления всех подов, они были автоматически созданы вновь благодаря контроллеру ReplicaSet.

Повторно применил манифест с 1 репликой, число подов уменьшилось обратно до одного.

Увеличил число реплик в манифесте до трех и применил его, число подов увеличилось обратно до трех.

Изменил версию образа в манифесте и повторно применил его, но ничего не произошло. Поды остались
работать на прежней версии образа. После удаления всех подов, они были созданы повторно, уже с новой
версией образа. Так происходит, потому что ReplicaSet не сравнивает фактическую спецификацию подов
и свой шаблон. Смотрит только на количество подов, возвращаемых по указанному селектору и создает новые,
когда их не хватает (либо убивает лишние).

Создал манифест replicaset для запуска v0.0.1 версии payment service. Применил и убедился, что сервис
поднялся на трех подах. Удалил replicaset, создал манифест deployment на его основе. Применил deployment.
Убедился, что создались три пода, которыми управляет созданный replicaset.

Обновил в манифесте версию образа на v0.0.2 и применил его. Убедился, что при обновлении применялась
стратегия RollingUpdate. Новые поды создавались по одному, а старые удалялись после каждого запущенного пода.

Выполнил kubectl rollout undo для deployment и откатил к предыдущей версии. На изначальном replicaset произошло
масштабирование до трех реплик, а на новом - до 0.

Создал deployment для сценария развертывания blue-green указанием maxSurge 3 и maxUnavailable 0. Таким образом
контроллер deployment сначала создаст replicaset для новой версии и запустит в нем все три реплики (благодаря maxSurge)
и только после запуска новых реплик начнет удалять старые поды.

Создал deployment для сценария развертывания reverse rolling update указанием maxSurge 0 и maxUnavailable 1. Таким образом
контроллер deployment сначала в существующем replicaset уменьшит число реплик на 1, затем увеличит в новом на 1.
И так до полного перехода на новую версию.

Создал deployment для запуска сервиса frontend и добавил в него readinessProbe. Применил deployment и убедился, что
все поды запустили и перешли в Ready.

Изменил в deployment readinessProbe на некорректный и запустил обновление на версию v0.0.2. Убедился, что при обновлении
запустился один под и дальше обновление не шло, т.к. этот под не мог перейти в Ready (readinessProbe возвращал ошибку)

Создал daemon set для запуска node exporter на всех четырех нодах. Для запуска в control plane указад в блоке
tolerations те taints, которыми отмечены ноды control-plane и master, чтобы node-exporter в том числе запустился там.
Через kubectl describe pod убедился, что поды запущенны на разных нодах (т.е. на всех четырех, включая control plane).
Пробросил порт через kubectl port-forward и убедился, что метрики выводятся.

# ДЗ 3
Скопировал манифест web-pod из первого задания и добавил в readinessProbe, что файл index.html отдается с порта 80

Создал под с используемым манифестом. Под перешел в состояние Running, но Ready не перешел, т.к. веб-сервер слушает
порт 8000, а проверка идет на 80.

Добавил в манифест проверку livenessProbe на доступность tcp сокета на порту 8000. И пересоздал под.

Вопросы для самопроверки
1. Потому что код возврата всегда будет 0, т.к. grep всегда будет находить одну строку с самой командой.
2. Кажется, что нет (по крайней мере в таком виде)

На основе манифеста пода сделал манифест deployment с одной репликой. Создал deployment, в kubectl describe убедился,
что Available установлен в False, т.к. реплики не переходят в Ready.

Исправил ошибку в readinessProbe в манифесте (порт изменил с 80 на 8000) и увеличил число реплик до 3. Применил
новый манифест и со временем нужное количество подов запустилось и перешло в Ready. И в Deployment Available перешел
в True.

Попробовал обновление образа в deployment с разными вариантами maxSurge и maxUnavailable:
оба 0 - падает ошибка, т.к. так делать нельзя
оба 100% - одновременно создаются три новых пода (с новым образом) и удаляются все старые
maxSurge 0, maxUnavailable 100% - сначала удаляются все существующие поды, параллельно с удалением создаются новые,
но без превышения общего количества подов в 3 (т.к. 3 реплики)
maxSurge 100%, maxUnavailable 0 - сначала создаются три новых пода (с новым образом), ппараллельно с их переходом в
Ready удаляются старые поды, но поддерживая все время количество готовых подов в 3 (т.к. 3 реплики)

Создал сервис типа ClusterIP для отправки запросов, приходящих на 80 порт сервиса, на 8000 порт подов с меткой app: web.
Убедился, что сервис создался и ему назначился IP (в моем случае 10.102.164.104).

Подключился по ssh к ноде minikube под root.
curl http://10.102.164.104/index.html - работает
Но пинга до этого IP нет и в вывод ip addr show его тоже нет.
В выводе iptables --list -nv -t nat видно добавленное правило для перенаправления трафика.

В конфигмапе kube-proxy изменил режим на ipvs и включил strictARP для ipvs. После чего удалил под kube-proxy, чтобы
применить новую конфигурацию.

При повторном ssh в ноду в выводе iptables --list -nv -t nat видно, что цепочка не удалилась, но ссылок на нее больше нет.
Выполнил очистку iptables конфигом из задания и через некоторое время kube-proxy восстановил остальное.

Попробовал опять сделать пинг ClusterIP. Но пинг все равно не делался, на сей раз с ошибкой port unreachable. Но по
крайней мере сам IP уже доступен. Убедился, что он есть в интерфейсе kube-ipvs0.

Установил MetalLB балансировщик. Убедился, что его компоненты запущены. Добавил конфигурацию MetalLB (из методички
не работает, взял из документации).

Создал и применил манифест сервиса LoadBalancer, настроенного на поды с меткой app=web.

Добавил статический маршрут на IP адрес minikube, после чего страница в браузере открылась по IP LoadBalancer.

Задание со * по dns пропустил.

Установил ingress nginx из манифеста. Добавил и применил манифест LoadBalancer для доступа к nginx снаружи.
Убедился, что ему назначился external ip. Переход на этот адрес снаружи выдает 404 от nginx, т.е. все работает.

Добавил headless сервис для объединения подов под ингресс. Убедился, что ни cluster, ни external ip такому сервису
не назначены.

Создал манифест ингресса для отправки запросов на поды. В методичке он устаревший и уже не работал. После этого
перейдя на IP_NGINX/web/index.html открылась нужная страничка.

Задания со * по ingress не делал.

# ДЗ 4
Создал манифест StatefulSet для MinIO из примера в ДЗ. Применил, запустился появился один под minio-0 использующий
PVC data-minio-0, который связан с PV pvc-dbb83c00-1371-44c1-8017-826cb3e7cdd4.

Добавил для MinIO headless service, который будет выбирать наше поды селектором app=minio.

Проверил работу MinIO. Для этого пробросил у сервиса порт 9000 наружу. Настроил minio cli для работы с этим minio,
создал бакет test и скопировал туда тестовый файл. Через mcli ls убедился, что файл загрузился. Удалил под minio,
заново запустил port forward (он отвалился из-за удаления пода) и еще раз убедился, что файл по-прежнему на месте.

Задание со * про секреты пропустил.

Создал и применил манифест для PV с хранилищем hostPath на 1Gi и режимом ReadWriteOnce.

Создал и применил манифест для PVC, запрашивающий 500Mi с режимом ReadWriteOnce.

Создал и применил манифест для пода, который монтирует том по пути /app/data и использует для этого созданный ранее PVC.

Подключился к поду, создал файл /app/data/data.txt и записал в него строку. После чего удалил под.

Создал еще один под, который использует тот же PVC. Подключился и убедился, что файл /app/data/data.txt на месте.

# ДЗ 5
Создал манифест для service account bob. Добавил role binding, чтобы выдать роль admin в рамках всего кластера.

Создал манифест с service account dave, без каких-либо ролей.

Создал манифест для создания namespace prometheus. Добавил манифест для создания в данном namespace service account carol.

Добавил манифест для создания кластерной роли pod-reader с правами на get list и watch подов во всем кластере.
И добавил манифест на rolebinding с выдачей всем service account в namespace prometheus роль pod-reader.

Добавил манифест для создания namespace dev. Добавил манифесты для создания в этом namespace service account jane и ken.
Добавил манифесты для выдачи ролей в рамках namespace dev: jane - роль admin, ken - роль view.
