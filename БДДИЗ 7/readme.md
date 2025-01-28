-- 1. Соединение с базой данных под пользователем waldexa.

 ![image](https://github.com/user-attachments/assets/537c8b8c-6497-4536-91e8-ed2dd89824ac)

-- 2. Создание новой роли r1_waldexa с паролем
CREATE ROLE r1_waldexa WITH LOGIN PASSWORD '12345'(password1);

 ![image](https://github.com/user-attachments/assets/6d113e48-70ba-4d11-bf59-a863ed7022f5)

-- 3. Установка срока действия роли
ALTER ROLE r1_waldexa VALID UNTIL '2025-04-27';

 ![image](https://github.com/user-attachments/assets/012fd5bd-5e45-46a0-a4bf-c0cd56cc555c)

-- 4. Передача права на выборку из таблицы real_estate_objects
GRANT SELECT ON TABLE real_estate_objects TO r1_waldexa;
-- 5. Зайти под новой ролью и выполнить запросы
SET ROLE r1_waldexa;
 ![image](https://github.com/user-attachments/assets/958f5793-3bed-4de2-901c-a6e0d83347b3)

-- Проверка текущего пользователя
SELECT current_user;

 ![image](https://github.com/user-attachments/assets/49fcccc6-1f43-48d1-a605-8da55facd980)

-- Попытка выборки и добавления записи
SELECT * FROM real_estate_objects;
INSERT INTO real_estate_objects (district, address, floor, number_of_rooms, type, status, price, building_material, area, listing_date, class) 
VALUES (1, '8514 Mandrake Drive, New Entry', 10, 3, 1, 1, 123000, 2, 750, NOW(), 'economy');

 ![image](https://github.com/user-attachments/assets/088cd438-6d3b-4b03-afd0-c9a9fc662f15)

-- ошибка на добавление записи
-- 6. Попытка создать новую роль под r1_waldexa
CREATE ROLE r3_waldexa WITH LOGIN PASSWORD 'password3';
 ![image](https://github.com/user-attachments/assets/df1f904f-3081-4046-8b3b-e1841b88d52f)

-- Назначить возможность создания ролей роли r1_waldexa
SET ROLE postgres;
ALTER ROLE r1_waldexa CREATEROLE;

 ![image](https://github.com/user-attachments/assets/495c9b8e-e0d6-411e-8512-6ffa256bb20d)

-- 7. Разрешение на добавление и обновление записей
GRANT INSERT, UPDATE ON real_estate_objects TO r1_waldexa;
GRANT USAGE, SELECT, UPDATE ON SEQUENCE real_estate_objects_object_id_seq TO r1_waldexa;

 ![image](https://github.com/user-attachments/assets/d61e527a-d07a-451c-a2b8-289906f3dccf)

-- Попробовать добавить запись под ролью r1_waldexa
SET ROLE r1_waldexa;
INSERT INTO real_estate_objects (district, address, floor, number_of_rooms, type, status, price, building_material, area, listing_date, class) 
VALUES (1, '8514 Mandrake Drive, New Entry', 13, 3, 1, 0, 21000000, 1, 80, NOW(), 'economy');

 ![image](https://github.com/user-attachments/assets/4742a7c4-ce53-4973-baa1-6bfb61581049)

-- 8. Передать роли r1_waldexa все привилегии на таблицы real_estate_objects и districts
GRANT ALL PRIVILEGES ON TABLE real_estate_objects TO r1_waldexa;
GRANT ALL PRIVILEGES ON TABLE districts TO r1_waldexa;

 ![image](https://github.com/user-attachments/assets/457336bb-2c83-4dd4-a53e-604a991e03bd)

-- 9. Выполнение трех запросов под r1_waldexa
SELECT * FROM districts;
UPDATE real_estate_objects SET area = 70 WHERE address = '8514 Mandrake Drive, New Entry';
DELETE FROM real_estate_objects WHERE object_id = 42;

 ![image](https://github.com/user-attachments/assets/73c0608b-d784-4bce-ada9-34c5b2f972b9)

-- 10. Зайти под r3_waldexa и выполнить запросы
SET ROLE r3_waldexa;
SELECT * FROM real_estate_objects;

 ![image](https://github.com/user-attachments/assets/32b4cca2-cf5a-4e6d-b0f3-8f6d09aa3291)

-- 11. Включение роли r3_waldexa в групповую роль r1_waldexa
SET ROLE postgres;
GRANT r1_waldexa TO r3_waldexa;
SET ROLE r3_waldexa;
SELECT * FROM real_estate_objects;

 ![image](https://github.com/user-attachments/assets/3e35930d-bf1f-4ef2-b5a7-6f5235136fce)

-- 12. Отнять у роли r1_waldexa возможность обновления таблицы real_estate_objects
SET ROLE postgres;
REVOKE UPDATE ON real_estate_objects FROM r1_waldexa;

 ![image](https://github.com/user-attachments/assets/97b1dd0e-ead1-40f9-bba7-744b4f1e6b8e)

SET ROLE r1_waldexa;
UPDATE real_estate_objects SET area = 70 WHERE address = '8514 Mandrake Drive, New Entry';

 ![image](https://github.com/user-attachments/assets/dfcd90b8-2bea-4113-8002-bc716c472efe)
 
-- ошибка
-- 13. Создание роли r2_waldexa и назначение её членом r1_waldexa
SET ROLE postgres;
CREATE ROLE r2_waldexa WITH LOGIN PASSWORD 'password2';
GRANT r1_waldexa TO r2_waldexa;
 ![image](https://github.com/user-attachments/assets/70b8efd9-3673-41c5-9c8d-fcacad49d3f3)

-- 14. Зайти под ролью r2_waldexa и добавить запись
SET ROLE r2_waldexa;
INSERT INTO real_estate_objects (district, address, floor, number_of_rooms, type, status, price, building_material, area, listing_date, class) 
VALUES (1, '8514 Mandrake Drive, Entry 2', 13, 3, 1, 0, 21000000, 1, 80, NOW(), 'economy');

 ![image](https://github.com/user-attachments/assets/613659f5-fb9a-42b3-88cf-0a31bfa40207)

-- 15. Сделать r2_waldexa владельцем таблицы
SET ROLE postgres;
ALTER TABLE real_estate_objects OWNER TO r2_waldexa;

 ![image](https://github.com/user-attachments/assets/a6ce05f7-c806-46d4-953a-102471db40e2)

-- 16. Попробовать добавить колонку под r1_waldexa и r2_waldexa
SET ROLE r1_waldexa;
ALTER TABLE real_estate_objects ADD COLUMN rooms INT;

 ![image](https://github.com/user-attachments/assets/8fb1703d-1261-46b5-90a4-dfac05150910)

SET ROLE r2_waldexa;
ALTER TABLE real_estate_objects ADD COLUMN rooms INT;

 ![image](https://github.com/user-attachments/assets/091436e5-07f9-4ba5-b830-31103bcd37fd)

![image](https://github.com/user-attachments/assets/d40bcf16-32f1-4f94-ab76-f3c4ec4e0525)

 
