1
CREATE OR REPLACE FUNCTION reduce_realtor_bonus() RETURNS 
TRIGGER AS $$  
BEGIN 
UPDATE bonuses 
SET bonus = bonus - OLD.cost * 0.05 
WHERE realtor_id = OLD.realtor_id; 
RETURN OLD; 
END 
$$ LANGUAGE plpgsql;

![image](https://github.com/user-attachments/assets/3c99625d-7a5f-4e2f-995f-c6f6ebe645f4)

CREATE TRIGGER reduce_realtor_bonus 
AFTER DELETE ON sales 
FOR EACH ROW 
EXECUTE FUNCTION reduce_realtor_bonus(); 
select * from bonuses;

![image](https://github.com/user-attachments/assets/d33fbfbe-8d34-428e-af52-58cd16d10161)

delete from sales where object_id = 1;
select * from bonuses;

2
CREATE OR REPLACE FUNCTION change_status_on_delete() RETURNS 
TRIGGER AS $$  
BEGIN 
UPDATE realty_object SET status = 1 WHERE object_id = OLD.object_id;  
RETURN OLD; 
END 
$$ LANGUAGE plpgsql; 
CREATE TRIGGER change_status_on_delete 
BEFORE DELETE ON sales 
FOR EACH ROW 
EXECUTE FUNCTION change_status_on_delete(); 
delete from sales where object_id = 1; 
select status from realty_object where object_id = 1;
![image](https://github.com/user-attachments/assets/318c1fa7-4558-4b40-a6ba-9eded8a2d682)

3
CREATE OR REPLACE FUNCTION uglify_phone_number(phone_number 
TEXT) RETURNS TEXT AS $$  
SELECT 
'+' || substr(phone_number, 1, 1) || 
' (' || substr(phone_number, 2, 3) || ') ' || 
substr(phone_number, 5, 3) || ' ' || 
substr(phone_number, 8, 2) || ' ' || 
substr(phone_number, 10, 2) 
$$ LANGUAGE sql IMMUTABLE; 

CREATE FUNCTION uglify_realtor_number() RETURNS TRIGGER AS $$  
BEGIN 
NEW.phone_number = uglify_phone_number(NEW.phone_number);  
RETURN NEW; 
END 
CREATE TRIGGER uglify_realtor_number 
BEFORE INSERT ON realtors 
FOR EACH ROW 
EXECUTE FUNCTION uglify_realtor_number(); 
$$ LANGUAGE plpgsql;
insert into realtors(lastname, firstname, phone_number) values('foo', 'bar', 
'79996667788'); 

![image](https://github.com/user-attachments/assets/6ed064c2-f287-4453-ab3a-959d774a1cae)

select * from realtors where lastname = 'foo';

4
CREATE TABLE Journal( 
timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),  
action VARCHAR(28) NOT NULL, 
actor TEXT NOT NULL 
);
CREATE OR REPLACE FUNCTION add_journal_entry() RETURNS 
TRIGGER SECURITY INVOKER AS $$  
BEGIN 
INSERT INTO Journal(action, actor) VALUES(TG_OP, current_role); 
RETURN NULL; 
END 
$$ LANGUAGE plpgsql; 
CREATE TRIGGER add_journal_entry 
AFTER INSERT OR UPDATE OR DELETE ON sales FOR EACH 
STATEMENT 
EXECUTE FUNCTION add_journal_entry(); 
delete from sales where sale_id = 2;
select * from journal ;

![image](https://github.com/user-attachments/assets/a82e465d-3ef0-428b-ba70-720ae6a930e5)

5

CREATE TABLE realtor_schedule(  
date DATE NOT NULL, 
start TIME NOT NULL, 
"end" TIME NOT NULL,  
realtor_id INTEGER NOT NULL);  

CREATE OR REPLACE FUNCTION maintain_correct_schedule() RETURNS 
TRIGGER AS $$  
BEGIN 
IF ( 
SELECT count(1) FROM realtor_schedule
WHERE realtor_id = NEW.realtor_id AND date = NEW.date 
) >= 3 THEN 
RAISE NOTICE 'Realtor % has more than 3', NEW.realtor_id; 
END IF; 

IF ( 
SELECT bool_or( 
  NEW."start" < "end" + interval '1h' AND NEW."end" > "start" - interval '1h' 
) 
FROM realtor_schedule 
WHERE realtor_id = NEW.realtor_id AND date = NEW.date 
) THEN 
RAISE EXCEPTION 'Deal window must be at least 1 hour between appointments'; 
END IF; 

IF ( 
SELECT bool_or((start, "end") OVERLAPS (NEW.start, NEW."end"))  
FROM realtor_schedule 
WHERE realtor_id = NEW.realtor_id AND date = NEW.date 
) THEN 
RAISE EXCEPTION 'Realtor % has overlapping schedules', NEW.realtor_id;  
END IF; 
 
IF extract(dow FROM NEW.date) = 0 THEN 
RAISE EXCEPTION 'Cannot create a deal on sunday';  
END IF; 
RETURN NEW;
END 
$$ LANGUAGE plpgsql; 
CREATE TRIGGER maintain_correct_schedule 
BEFORE INSERT ON realtor_schedule 
FOR EACH ROW 
EXECUTE FUNCTION maintain_correct_schedule(); 
insert into realtor_schedule(date, start, "end", realtor_id) values('2023-12-30', 
'16:00', '17:00', 1);
insert into realtor_schedule(date, start, "end", realtor_id) values('2023-12-30', 
'22:00', '23:00', 1); 

![image](https://github.com/user-attachments/assets/54c9f224-d0e9-4ee8-a1b1-1380d397a258)

insert into realtor_schedule(date, start, "end", realtor_id) values('2023-12-30', 
'21:00', '22:00', 1); 

![image](https://github.com/user-attachments/assets/828828cf-c4ad-4807-aaa6-7c867aef2812)

6
CREATE TABLE prices( 
timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),  
object_id INTEGER NOT NULL, 
price REAL NOT NULL );
CREATE OR REPLACE FUNCTION add_price() RETURNS TRIGGER AS $$ 
BEGIN 
INSERT INTO prices(object_id, price) VALUES (NEW.object_id, NEW.cost);  
RETURN NEW; 
END 
$$ LANGUAGE plpgsql;
CREATE TRIGGER add_price 
AFTER INSERT OR UPDATE ON realty_object 
FOR EACH ROW 
EXECUTE FUNCTION add_price();
insert into realty_object(cost) values(5000) returning object_id;

![image](https://github.com/user-attachments/assets/ee0e3f8b-0484-42bb-b312-c4002160864d)

update realty_object set cost = 6000 where object_id = 39;
![image](https://github.com/user-attachments/assets/a7e26ba0-d3d4-4c32-a8d4-68fc537af99e)

