# DZ_07

-- Описание/Пошаговая инструкция выполнения домашнего задания:
-- Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

postgres@postgres25032022:/home/anadyrov$ psql
psql (14.2 (Ubuntu 14.2-1.pgdg18.04+1))
Type "help" for help.

postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout='200';
ALTER SYSTEM
postgres=# SHOW deadlock_timeout;
 deadlock_timeout 
------------------
 200ms
(1 row)


postgres=# create database lab7;
CREATE DATABASE
postgres=# \c lab7
You are now connected to database "lab7" as user "postgres".
lab7=# 
lab7=# CREATE TABLE accounts(
lab7(#   acc_no integer PRIMARY KEY,
lab7(#   amount numeric
lab7(# );
CREATE TABLE
lab7=# INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
INSERT 0 3
lab7=# 


postgres@postgres25032022:/home/anadyrov$ tail -n 10 /var/log/postgresql/postgresql-14-main.log .
==> /var/log/postgresql/postgresql-14-main.log <==
2022-04-06 04:03:20.587 UTC [17011] postgres@postgres ERROR:  relation "accounts" does not exist at character 15
2022-04-06 04:03:20.587 UTC [17011] postgres@postgres STATEMENT:  SELECT * FROM accounts;
2022-04-06 04:04:28.680 UTC [17065] postgres@locks FATAL:  database "locks" does not exist
2022-04-06 04:15:16.669 UTC [17450] postgres@lab7 LOG:  process 17450 still waiting for ShareLock on transaction 737 after 200.137 ms
2022-04-06 04:15:16.669 UTC [17450] postgres@lab7 DETAIL:  Process holding the lock: 17453. Wait queue: 17450.
2022-04-06 04:15:16.669 UTC [17450] postgres@lab7 CONTEXT:  while updating tuple (0,4) in relation "accounts"
2022-04-06 04:15:16.669 UTC [17450] postgres@lab7 STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2022-04-06 04:16:09.072 UTC [17450] postgres@lab7 LOG:  process 17450 acquired ShareLock on transaction 737 after 52602.332 ms
2022-04-06 04:16:09.072 UTC [17450] postgres@lab7 CONTEXT:  while updating tuple (0,4) in relation "accounts"
2022-04-06 04:16:09.072 UTC [17450] postgres@lab7 STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

-- Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.


-- Первая
lab7=*# SELECT * FROM block WHERE pid = 4874;
 pid  |   locktype    |  lockid  |       mode       | granted 
------+---------------+----------+------------------+---------
 4874 | relation      | pg_locks | AccessShareLock  | t
 4874 | relation      | locks    | AccessShareLock  | t
 4874 | relation      | test7    | RowExclusiveLock | t
 4874 | virtualxid    | 4/24     | ExclusiveLock    | t
 4874 | transactionid | 747      | ExclusiveLock    | t
(5 rows)

-- Вторая
lab7=# SELECT * FROM block WHERE pid = 5336;
 pid  |   locktype    | lockid  |       mode       | granted 
------+---------------+---------+------------------+---------
 5336 | relation      | test7   | RowExclusiveLock | t
 5336 | virtualxid    | 6/5     | ExclusiveLock    | t
 5336 | transactionid | 748     | ShareLock        | f
 5336 | transactionid | 749     | ExclusiveLock    | t
 5336 | tuple         | test7:5 | ExclusiveLock    | t
(5 rows)

-- Третья
lab7=# SELECT * FROM block WHERE pid = 4886;
 pid  |   locktype    | lockid |       mode       | granted 
------+---------------+--------+------------------+---------
 4886 | relation      | test7  | RowExclusiveLock | t
 4886 | virtualxid    | 5/4    | ExclusiveLock    | t
 4886 | transactionid | 748    | ExclusiveLock    | t
(3 rows)

-- AccessShareLock устанавливаются на читаемые отношения
-- RowExclusiveLock устанавливается на изменяемое отношение
-- ExclusiveLock удерживаются каждой транзакцией


-- Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
-- Думаю да, подробную информацию можно получить из логов

lab7=*# UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 3;
UPDATE 1
lab7=*# UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 1;
ERROR:  deadlock detected
DETAIL:  Process 4577 waits for ShareLock on transaction 743; blocked by process 4349.
Process 4349 waits for ShareLock on transaction 744; blocked by process 4463.
Process 4463 waits for ShareLock on transaction 745; blocked by process 4577.
HINT:  See server log for query details.

tail -20f /var/log/postgresql/postgresql-14-main.log

2022-04-07 05:47:14.528 UTC [4577] postgres@lab7 ERROR:  deadlock detected
2022-04-07 05:47:14.528 UTC [4577] postgres@lab7 DETAIL:  Process 4577 waits for ShareLock on transaction 743; blocked by process 4349.
        Process 4349 waits for ShareLock on transaction 744; blocked by process 4463.
        Process 4463 waits for ShareLock on transaction 745; blocked by process 4577.
        Process 4577: UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 1;
        Process 4349: UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 2;
        Process 4463: UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 3;
2022-04-07 05:47:14.528 UTC [4577] postgres@lab7 HINT:  See server log for query details.
2022-04-07 05:47:14.528 UTC [4577] postgres@lab7 CONTEXT:  while updating tuple (0,1) in relation "test7"
2022-04-07 05:47:14.528 UTC [4577] postgres@lab7 STATEMENT:  UPDATE test7 SET name2 = name2 - 2 WHERE name1 = 1;
2022-04-07 05:47:14.529 UTC [4463] postgres@lab7 LOG:  process 4463 acquired ShareLock on transaction 745 after 12416.682 ms

-- Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
-- Возможно но в жизни маловероятно :))

-- Попробуйте воспроизвести такую ситуацию.

lab7=*# FETCH laba1;
ERROR:  deadlock detected
DETAIL:  Process 6019 waits for ShareLock on transaction 753; blocked by process 6075.
Process 6075 waits for ShareLock on transaction 752; blocked by process 6019.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,3) in relation "test7"

tail -30f /var/log/postgresql/postgresql-14-main.log

2022-04-07 06:44:14.600 UTC [6019] postgres@lab7 DETAIL:  Process 6019 waits for ShareLock on transaction 753; blocked by process 6075.
        Process 6075 waits for ShareLock on transaction 752; blocked by process 6019.
        Process 6019: FETCH laba1;
        Process 6075: FETCH laba2;
2022-04-07 06:44:14.600 UTC [6019] postgres@lab7 HINT:  See server log for query details.
2022-04-07 06:44:14.600 UTC [6019] postgres@lab7 CONTEXT:  while locking tuple (0,3) in relation "test7"
2022-04-07 06:44:14.600 UTC [6019] postgres@lab7 STATEMENT:  FETCH laba1;
2022-04-07 06:44:14.601 UTC [6075] postgres@lab7 LOG:  process 6075 acquired ShareLock on transaction 752 after 197732.571 ms
2022-04-07 06:44:14.601 UTC [6075] postgres@lab7 CONTEXT:  while locking tuple (0,2) in relation "test7"
2022-04-07 06:44:14.601 UTC [6075] postgres@lab7 STATEMENT:  FETCH laba2;



