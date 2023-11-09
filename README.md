# funbiscuit_platform
funbiscuit Platform repository

# ДЗ 1
В kubernetes-intro добавил Dockerfile и nginx.conf для сборки образа веб-сервера, выдающего
файлы из директории /app.

Для запуска веб-сервера сделан манифест пода web-pod.yaml.

Для проекта Hipster Shop сделан манифест пода frontend: frontend-pod.yaml. Для запуска
потребовались дополнительные переменные среды (адреса других микросервисов), их добавил
в манифест frontend-pod-healthy.yaml.
