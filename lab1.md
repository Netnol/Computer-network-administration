# Лабораторная работа №1

### Развёртывание и настройка высокодоступного кластера PostgreSQL

## Часть 1

1. Подготовка и создание всех файлов
Для развёртывания кластера были подготовлены необходимые конфигурационные файлы: Dockerfile, docker-compose.yml, postgres0.yml и postgres1.yml
<img width="746" height="167" alt="image" src="https://github.com/user-attachments/assets/757f2e8b-7879-46db-84ef-f32c6a770bb6" />

2. Сборка Docker-образ
Для создания Docker-образа была выполнена команда: docker compose build
<img width="740" height="723" alt="image" src="https://github.com/user-attachments/assets/65d9b8ce-6caf-41ba-a621-ae8e11362403" />

В результате был успешно собран образ.
   
3. Запуск кластера
Для запуска всех сервисов, описанных в docker-compose.yml, была выполнена команда: docker compose up -d
<img width="793" height="122" alt="image" src="https://github.com/user-attachments/assets/fabe195f-c953-467b-9c49-c1b2cb325fe5" />

После запуска были созданы и запущены контейнеры pg-master, pg-slave и zoo
3.1 При изменении файла postgres0.yml или postgres1.yml необходимо выполнить пересборку Docker-образа, так как эти файлы копируются в образ во время выполнения docker compose build. Поэтому после изменения postgresX.yml необходимо выполнить:
docker compose up -d --build. При изменении Dockerfile также необходимо пересобрать Docker-образ, так как изменяется инструкция его сборки.

4.Проверяем состояние контейнеров
Для проверки состояния контейнеров была выполнена команда: docker compose ps
<img width="1393" height="172" alt="image" src="https://github.com/user-attachments/assets/9b5394ba-5966-4b71-abf2-e109daab6280" />
В результате проверки все три контейнера успешно запущены и находятся в состоянии Up.

5.Проверяем логи Zookeeper
Для проверки работы Zookeeper были просмотрены его логи.
<img width="1384" height="808" alt="image" src="https://github.com/user-attachments/assets/5be380de-3274-4087-ae6e-2d539f066c23" />

По выводу логов видно, что Zookeeper успешно запустился и начал принимать подключения на порту 2181

6.Проверяем Patroni на первой ноде
Для проверки состояния Patroni на узле pg-master была выполнена команда: docker logs pg-master --tail 100
<img width="1231" height="652" alt="image" src="https://github.com/user-attachments/assets/9fb71153-11ad-4cd6-89e9-904355723946" />

видно: pg-master не лидер. Он пишет: Lock owner: postgresql1; I am postgresql0
и I am (postgresql0), a secondary, and following a leader (postgresql1)
=> pg-master → Replica и pg-slave →  Leader

7.Проверяем pg-slave
Для подтверждения состояния второго узла были просмотрены логи pg-slave
<img width="1227" height="633" alt="image" src="https://github.com/user-attachments/assets/28ec47ea-6762-415f-8676-17f4d620bd50" />

По логам однозначно видно, что 
pg-slave → leader (главный узел)
pg-master → secondary/replica (реплика)

8. Для подключения к PostgreSQL на узле pg-slave была выполнена команда: docker exec -it pg-slave psql -h 127.0.0.1 -U postgres
<img width="917" height="185" alt="image" src="https://github.com/user-attachments/assets/648699dd-a879-4961-a8d4-6974e6eacc5c" />

9. Для определения роли узла была выполнена команда: SELECT pg_is_in_recovery()
<img width="378" height="119" alt="image" src="https://github.com/user-attachments/assets/9c41d9ce-64ff-42cc-b59f-87b4da404f45" />

В результате получено значение: f
Значение f (false) означает, что PostgreSQL не находится в режиме восстановления.

10. Проверка режима восстановления на pg-master
<img width="923" height="185" alt="image" src="https://github.com/user-attachments/assets/9890caaa-9dd3-4c96-91ec-c379d7cd740f" />
t (true) —  pg-master является репликой

11.Создание тестовой таблицы
<img width="933" height="372" alt="image" src="https://github.com/user-attachments/assets/4ddd54fe-d01b-4262-a8fb-ecec70ad2d98" />

12. Проверка репликации
После создания таблицы на лидере было выполнено подключение к pg-master и выполнена команда: SELECT * FROM test_table;
<img width="918" height="237" alt="image" src="https://github.com/user-attachments/assets/a4d85b50-0303-4875-a9e3-8e23a5e22c4a" />

репликация работает

13. Проверка что нельзя записать в реплику
Для проверки режима только для чтения на pg-master была выполнена попытка добавить новую запись: INSERT INTO test_table (name) VALUES ('Replica test');
<img width="963" height="129" alt="image" src="https://github.com/user-attachments/assets/a2806c51-be59-48fe-b2be-f62795a86fe0" />

В результате PostgreSQL вернул ошибку ERROR: cannot execute INSERT in a read-only transaction. Это подтверждает, что pg-master работает в режиме только для чтения и не принимает операции записи.
