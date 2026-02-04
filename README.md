# Ansible Deployment Task

Этот проект автоматизирует развертывание Docker контейнеров (Nginx, PostgreSQL, pg_cron) на целевом сервере с использованием Ansible.

## Структура проекта

- **inventory/hosts**: Файл инвентаря с адресом целевого сервера.
- **roles/**:
  - `docker_install`: Установка Docker CE и зависимостей.
  - `nginx_container`: Запуск Nginx с кастомным конфигом.
  - `postgres_container`: Запуск основного PostgreSQL.
  - `pg_cron_container`: Запуск PostgreSQL с pg_cron и его настройка.
- **vars/**:
  - `encrypted_vars.yml`: Зашифрованные Ansible Vault переменные (SSH ключи, пользователи, пароли БД).
  - `.vault_pass`: Файл с паролем от хранилища (для демо целей содержит "vault_password").
- **site.yml**: Основной плейбук.

## Как работать с секретами

Все чувствительные данные (пароли, SSH ключи) хранятся в `vars/encrypted_vars.yml`.

Для редактирования зашифрованных переменных используйте команду:
```bash
ansible-vault edit vars/encrypted_vars.yml --vault-password-file .vault_pass
```

Для просмотра содержимого:
```bash
ansible-vault view vars/encrypted_vars.yml --vault-password-file .vault_pass
```

В данном проекте зашифрованы:
- `ansible_user`: Пользователь для подключения (ubuntu).
- `ansible_ssh_private_key_file`: Путь к приватному ключу.
- `postgres_password`: Пароль от основной БД.
- `pg_cron_password`: Пароль от БД pg_cron.

## Запуск деплоя

Чтобы раскатать конфигурацию на сервер, выполните:

```bash
ansible-playbook -i inventory/hosts site.yml --vault-password-file .vault_pass
```

## Удаление (Uninstall)

Чтобы удалить все установленные компоненты (контейнеры, данные, сам Docker), запустите плейбук с переменными состояния `absent`:

```bash
ansible-playbook -i inventory/hosts site.yml --vault-password-file .vault_pass -e "docker_install_state=absent nginx_container_state=absent postgres_container_state=absent pg_cron_container_state=absent"
```

## Проверка работоспособности

После деплоя можно проверить:

1. **Nginx**: `curl http://34.205.71.187:8080` (должен вернуть "Hello from Ansible-managed Nginx!").

2. **Docker**: `docker ps` на сервере.

3. **pg_cron**: 

   **Важно**: В проекте есть два контейнера PostgreSQL:
   - `main_postgres` - основной PostgreSQL (пользователь: `myuser`, БД: `mydb`, порт: 5432)
   - `cron_postgres` - PostgreSQL с pg_cron (пользователь: `cron_user`, БД: `cron_db`, порт: 5433)



   **Ручная проверка:**
   ```bash
   # Подключение к контейнеру cron_postgres
   sudo docker exec -it cron_postgres psql -U cron_user -d cron_db
   
   # Проверка наличия расширения pg_cron
   sudo docker exec -it cron_postgres psql -U cron_user -d cron_db -c "SELECT * FROM pg_extension WHERE extname='pg_cron';"
   
   # Просмотр всех задач cron
   sudo docker exec -it cron_postgres psql -U cron_user -d cron_db -c "SELECT * FROM cron.job;"
   
   # Просмотр истории выполнения задач
   sudo docker exec -it cron_postgres psql -U cron_user -d cron_db -c "SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;"
   ```

   **Пример создания тестовой задачи:**
   ```bash
   sudo docker exec -it cron_postgres psql -U cron_user -d cron_db -c "SELECT cron.schedule('test-job', '*/5 * * * *', 'SELECT NOW();');"
   ```
