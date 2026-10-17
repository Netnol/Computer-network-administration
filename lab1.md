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

## Часть 3

1. Для балансировки трафика добавили новый сервис haproxy в docker-compose.yml
3. Добавляем файл haproxy.cfg
<img width="204" height="132" alt="Screenshot 2026-10-16 at 18 12 21" src="https://github.com/user-attachments/assets/7dc8a3ad-87a1-4c02-92a8-ae8d425313cd" />

4. Перезапускаем проект, проверяем что:
-Обе ноды корректно поднялись и распределили между собой роли мастера и слейва (аналогично Пункту 4 Части 1)
<img width="1291" height="273" alt="Screenshot 2026-10-16 at 18 17 14" src="https://github.com/user-attachments/assets/a0d573cc-baa0-4d1e-8f84-f9002ebe88b1" />

Зукипер подцепился к кластеру без ошибок (аналогично Пункту 4 Части 1)
<img width="1298" height="427" alt="Screenshot 2026-10-16 at 18 20 41" src="https://github.com/user-attachments/assets/4cca3e16-cc55-40a2-987b-eb00ad78d67d" />

Хапрокси поднялась без ошибок
<img width="1297" height="257" alt="Screenshot 2026-10-16 at 18 20 57" src="https://github.com/user-attachments/assets/9cdef2d7-c43d-4ac1-a886-5fdbe113e097" />

<img width="810" height="48" alt="Screenshot 2026-10-16 at 18 21 10" src="https://github.com/user-attachments/assets/85f24dfb-9d58-4af0-9b85-9debbb59720c" />

4. Проверяем базу данных, подключаемся через адрес имя контейнера HAProxy
<img width="842" height="421" alt="Screenshot 2026-10-16 at 18 26 27" src="https://github.com/user-attachments/assets/bec083f4-7dd3-4fef-986d-381b9f5cd2b1" />

<img width="1508" height="430" alt="Screenshot 2026-10-16 at 18 24 58" src="https://github.com/user-attachments/assets/ddf2f02c-a86e-42f3-b3d8-7280118196f7" />

Любым способом выключаем доступ до ноды, которая сейчас является мастером (например, через docker stop ). Некоторое время ждем, после этого анализируем логи и так же пытаемся считать/записать что-то в БД через entrypoint подключение. Затем необходимо расписать, получилось или нет, а так же объяснить, что в итоге произошло после принудительного выключения мастера (со скриншотами):
Остановили главное бд:
<img width="795" height="210" alt="Screenshot 2026-10-16 at 18 27 05" src="https://github.com/user-attachments/assets/549d4870-22a0-42f2-a736-fd5e7f0f7b07" />

Заходим в slave бд а адресом HAProxy и проверяем что оно главное и можно редактировать бд:
<img width="861" height="479" alt="Screenshot 2026-10-16 at 18 27 53" src="https://github.com/user-attachments/assets/388c579f-c851-4c81-bdb7-9c22eeba6f82" />

После принудительного выключения мастера страница статистики поменялась, также была разорвана связь с БД, которую выключили в терминале:
<img width="1512" height="431" alt="Screenshot 2026-10-16 at 18 31 12" src="https://github.com/user-attachments/assets/87b38d55-248b-4634-84b3-e788ebfa89c3" />

<img width="862" height="472" alt="Screenshot 2026-10-16 at 18 31 30" src="https://github.com/user-attachments/assets/ed1ac73d-db67-493c-9f56-d2e3e057530c" />


# Дополнительное задание: 

Улучшить лабораторную, настроив кластер так, чтобы после “возвращенияˮ второй ноды в кластер она автоматически получала данные, которые
были записаны в ее отсутствие

1. Запускаем master:
<img width="563" height="32" alt="Screenshot 2026-10-16 at 18 35 17" src="https://github.com/user-attachments/assets/648e888d-86a3-4f90-9f98-41f45e9335f6" />

2. Проверяем состояние кластера:
<img width="802" height="100" alt="Screenshot 2026-10-16 at 18 36 19" src="https://github.com/user-attachments/assets/d9fb7a0e-7b49-406d-9d4b-adb735c8c365" />

3. Подключаемся к восстановленной ноде и проверяем данные:
<img width="734" height="300" alt="Screenshot 2026-10-16 at 18 39 44" src="https://github.com/user-attachments/assets/e9a351ff-345c-4b8e-987e-de2815ffcbde" />

Ключевая запись AfterFailover. Она была добавлена, когда pg-master был выключен. Если сейчас она присутствует, это подтверждает, что сервер автоматически получил изменения, пропущенные за время отсутствия.

5. 




