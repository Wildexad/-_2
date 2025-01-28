1. Создать триггер, который меняет значение поля «Статус» в таблице 
«Объекты недвижимости» на 0 – при добавлении новой продажи. -- Создание функции триггера 
CREATE OR REPLACE FUNCTION update_realty_object_status() 
RETURNS TRIGGER AS $$ 
BEGIN 
  UPDATE realty_object 
  SET status = 0 
  WHERE object_id = NEW.object_id; 
  RETURN NEW; 
END; 
$$ LANGUAGE plpgsql; 
 -- Создание триггера 
CREATE TRIGGER update_status_after_insert 
AFTER INSERT ON sales 
FOR EACH ROW 
EXECUTE FUNCTION update_realty_object_status(); 
 
Запрос для проверки: 
INSERT INTO realty_object ( 
    object_id, 
    type_id, 
    material_id, 
    district_id, 
    area, 
    house_class, 
    rooms_count, 
    floor, 
    status, 
address, 
cost, 
listing_date 
) VALUES ( 
1001,        
1,           
1,           
1,           
75.0,        -- уникальный идентификатор объекта -- существующий type_id -- существующий material_id -- существующий district_id -- площадь в квадратных метрах 
'эконом',    -- класс дома 
3,           
5,           
1,           -- количество комнат -- этаж -- статус (1 - доступен) 
'ул. Ленина, д. 10', -- адрес 
8500000.00,  -- стоимость 
NOW()        -- дата размещения 
); 
Добавляем новую продажу 
INSERT INTO sales ( 
    sale_id, 
    object_id, 
    realtor_id, 
    sale_date, 
    cost, 
    realtor_commision 
) VALUES ( 
    5001, 
    1001, 
    1, 
    NOW(), 
    8300000.00, 
    200000.00 
); 
 
Проверка статуса продажи 
SELECT object_id, status 
FROM realty_object 
WHERE object_id = 1001; 
2. 
Создать триггер, который при продаже будет выводить 
предупреждающее сообщение при разнице заявленной и продажной 
стоимости объекта недвижимости более чем на 20%. 
СОЗДАЁМ ТРИГГЕР -- Создание функции триггера 
CREATE OR REPLACE FUNCTION check_price_difference() 
RETURNS TRIGGER AS $$ 
DECLARE 
listing_cost numeric; 
percentage_difference numeric; 
BEGIN -- Получаем заявленную стоимость из таблицы realty_object 
SELECT cost INTO listing_cost 
    FROM realty_object 
    WHERE object_id = NEW.object_id; 
     
    -- Проверяем, найдена ли заявленная стоимость 
    IF listing_cost IS NULL THEN 
        RAISE EXCEPTION 'Заявленная стоимость не найдена для object_id %', 
NEW.object_id; 
    END IF; 
     
    -- Вычисляем процентную разницу между продажной и заявленной 
стоимостью 
    percentage_difference := abs((NEW.cost - listing_cost) / listing_cost); 
     
    -- Если разница превышает 20%, выводим предупреждение 
    IF percentage_difference > 0.2 THEN 
        RAISE WARNING 'Разница между заявленной и продажной 
стоимостью для object_id % превышает 20%%', NEW.object_id; 
    END IF; 
     
    RETURN NEW; 
END; 
$$ LANGUAGE plpgsql; 
 -- Создание триггера 
CREATE TRIGGER price_difference_warning 
AFTER INSERT ON sales 
FOR EACH ROW 
EXECUTE FUNCTION check_price_difference(); 
 
 
ДОБАВЛЯЕМ НОВЫЙ ОБЪЕКТ 
INSERT INTO realty_object ( 
    object_id, type_id, material_id, district_id, area, house_class, 
    rooms_count, floor, status, address, cost, listing_date 
) VALUES ( 
    1005, 1, 1, 1, 100.0, 'эконом', 4, 10, 1, 
    'ул. Пушкина, д. 1', 1000000.00, NOW() 
); 
 
INSERT INTO realty_object ( 
    object_id, type_id, material_id, district_id, area, house_class, 
    rooms_count, floor, status, address, cost, listing_date 
) VALUES ( 
    1006, 1, 1, 1, 100.0, 'эконом', 4, 10, 1, 
    'ул. Пушкина, д. 1', 1000000.00, NOW() 
); 
 
ПРОДАЁМ БОЛЕЕ 20% РАЗНИЦА 
INSERT INTO sales ( 
    sale_id, object_id, realtor_id, sale_date, cost, realtor_commision 
) VALUES ( 
    6001, 1005, 1, NOW(), 750000.00, 50000.00 
); 
ПРОДАЁМ МЕНЕЕ 20% РАЗНИЦА 
INSERT INTO sales ( 
sale_id, object_id, realtor_id, sale_date, cost, realtor_commision 
) VALUES ( 
6002, 1006, 1, NOW(), 850000.00, 50000.00 
); 
3. Создать триггер, который при добавлении новой продажи будет 
осуществлять проверку статуса объекта недвижимости. Если в таблице 
«Продажи» уже имеется запись о продаже данного объекта, выводить 
соответствующее сообщение и запрещать новую продажу. 
СОЗДАЕМ ФУНКЦИЮ 
CREATE FUNCTION do_not_sell_twice() RETURNS TRIGGER AS $$  
BEGIN 
IF EXISTS (SELECT 1 FROM sales WHERE object_id = NEW.object_id) 
THEN  
RAISE EXCEPTION 'Cannot sell the building twice'; 
END IF; 
RETURN NEW;  
END 
$$ LANGUAGE plpgsql; 
СОЗДАЕМ ТРИГГЕР 
CREATE TRIGGER do_not_sell_twice 
BEFORE INSERT ON sales 
FOR EACH ROW 
EXECUTE FUNCTION do_not_sell_twice(); 
Проверяем insert into sales(object_id, cost) values(1, 1); 
 
4. Создать триггер, который при добавлении новой записи в таблицу 
«Структура объекта недвижимости» проверяет несоответствие суммы 
площадей всех заявленных комнат (в большую сторону) с общей площадью 
объекта недвижимости. Выводить сообщение на сколько превышена площадь. 
СОЗДАЁМ ФУНКЦИЮ 
CREATE OR REPLACE FUNCTION area_mismatch() RETURNS TRIGGER 
AS $$  
    DECLARE 
        diff NUMERIC := NULL; 
    BEGIN 
        diff := ( 
            SELECT sum(area)::numeric 
            FROM realty_structure 
            WHERE realty_object_id = NEW.realty_object_id 
        ) - ( 
            SELECT area FROM realty_object 
            WHERE object_id = NEW.realty_object_id 
        ); 
        IF diff != 0 THEN 
            RAISE EXCEPTION 'Area sum of roms != building area (%)', diff;  
        END IF; 
        RETURN NEW;  
    END 
$$ LANGUAGE plpgsql; 
СОЗДАЁМ ТРИГГЕР 
CREATE TRIGGER area_mismatch 
BEFORE INSERT ON realty_structure 
FOR EACH ROW 
EXECUTE FUNCTION area_mismatch(); 
Проверяем работу insert into realty_structure(structure_id, realty_object_id, 
area) values(213321, 1, 9999999); 
5. Добавить таблицу «Бонусы», в которой будет две колонки: код 
риэлтора и размер накопленных бонусов. Размер бонуса рассчитывается по 
формуле – стоимость продажи*5%. Создать триггер, который будет 
автоматически увеличивать размер накопленного бонуса при добавлении 
новой продажи. Необходимо учесть, что продажа риэлтора может быть 
впервые, следовательно необходимо добавить новую запись в таблицу. 
СОЗДАЮ ТАБЛИЦУ БОНУСЫ 
CREATE TABLE IF NOT EXISTS bonuses( 
realtor_id INTEGER REFERENCES realtors PRIMARY KEY,  
bonus DOUBLE PRECISION DEFAULT 0 
); 
 
 
СОЗДАМ ФУНКЦИЮ 
CREATE OR REPLACE FUNCTION accumulate_bonus() RETURNS 
TRIGGER AS $$  
    BEGIN 
        INSERT INTO bonuses(realtor_id, bonus) 
        VALUES (NEW.realtor_id, NEW.cost * 1.05) 
        ON CONFLICT (realtor_id) DO UPDATE SET 
            bonus = bonuses.bonus + NEW.cost * 1.05; 
        RETURN NEW;  
    END 
$$ LANGUAGE plpgsql; 
 
СОЗДАЁМ ТРИГГЕР 
CREATE TRIGGER accumulate_bonus 
BEFORE INSERT ON sales 
FOR EACH ROW 
EXECUTE FUNCTION accumulate_bonus(); 
ПРОВЕРЯЕМ insert into sales(object_id, realtor_id, cost) values(6, 1, 100);

6. Создать триггер, который будет информировать о превышении 
размера накопленного бонуса риэлтором. Максимальный размер бонуса 
установить самостоятельно. 

CREATE OR REPLACE FUNCTION check_if_bonus_exceeds() RETURNS 
TRIGGER AS $$  
BEGIN 
IF NEW.bonus > 1 THEN 
RAISE NOTICE 'Realtor exceeds bonus limit'; 
END IF; 
RETURN NEW;  
END 
$$ LANGUAGE plpgsql; 
СОЗДАЁМ ТРИГГЕР 
CREATE TRIGGER check_if_bonus_exceeds BEFORE UPDATE ON bonuses 
FOR EACH ROW 
EXECUTE FUNCTION check_if_bonus_exceeds(); 
Проверяем insert into sales(object_id, realtor_id, cost) values(6, 1, 100); 
7. В таблицу «Риэлторы» добавить колонку «Паспортные данные». 
Создать триггер, который будет проверять корректность паспортных данных 
по маске: ХХХХ УУУУУУ, где Х – серия паспорта, У – номер паспорта. 
Между серией и номером паспорта пробел. При несоответствии вводимой 
информации маске, выводить сообщение. 
ДОПОЛНЯЕМ ТАБЛИЦУ НОВЫМ СТОЛБЦОМ 
ALTER TABLE realtors ADD COLUMN passport TEXT; 
СОЗДАЁМ ФУНКЦИЮ 
CREATE OR REPLACE FUNCTION assert_valid_passport() RETURNS 
TRIGGER AS $$ 
BEGIN 
IF NEW.passport IS NOT NULL AND NEW.passport NOT SIMILAR TO 
'[1-9]{4} [1-9] {6}' 
THEN 
RAISE EXCEPTION '% is not a valid passport', NEW.passport; 
END IF; 
RETURN NEW;  
END 
$$ LANGUAGE plpgsql; 
СОЗДАЁМ ТРИГГЕР 
CREATE TRIGGER assert_valid_passport 
BEFORE INSERT OR UPDATE ON realtors 
FOR EACH ROW 
EXECUTE FUNCTION assert_valid_passport(); 
ПРОВЕРЯЕМ update realtors set passport = '1234 123456' where realtor_id 
= 1; 
update realtors set passport = '1234 А123456' where realtor_id = 1; 
