## Создание и вызов функций PL/SQL Oracle

### Задача №1. 
--------------
~~~~
Написать функцию, которая возвращает количество абонентов активных на 01.11.2021 
(Дата 01.11.2021 находится между SUBSCRIBERS.START_DATE и SUBSCRIBERS.END_DATE).

   	На вход функции подаем дату 01.11.2021. 
	
Функция должна возвращать числовой результат.
~~~~
### Решение
~~~~
CREATE OR REPLACE FUNCTION am_activ_abon (p_date date)
  RETURN NUMBER
AS
  v_activ_abon NUMBER;
BEGIN
  SELECT COUNT(*)
  INTO v_activ_abon
  FROM
    (SELECT *
    FROM subscribers
    WHERE END_DATE > p_date
    OR END_DATE   IS NULL
    ) subscribers
  WHERE START_DATE < p_date;
  RETURN v_activ_abon;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  v_activ_abon := 0;
WHEN OTHERS THEN
  v_activ_abon := -1;
END;
~~~~
### Вызов функции
~~~~
SELECT am_activ_abon (to_date('01.11.2021','dd.mm.yyyy'))"Активные абоненты"
FROM dual;
~~~~

### Задача №2
------------
~~~~
Написать функцию, которая возвращает количество неактивных(см. глоссарий) участников 
бонусной программы по МРФ Центр (SUBSCRIBERS.MRF = 'Центр'). 
   
   Обратите внимание, что есть участники, которые выходили из бонусной программы и потом зарегистрировались повторно 
   (В таблице MEMBERS есть больше двух записей по MEMBERS.SUBS_ID). 

На вход функции подаем строку ‘Центр’. 

Функция должна возвращать числовой результат.
~~~~
### Решение
~~~~
CREATE OR REPLACE FUNCTION am_noactiv_memb (p_MRF SUBSCRIBERS.MRF%type)
  RETURN NUMBER
AS
  v_noactiv_memb NUMBER;
BEGIN
  SELECT COUNT (*)
  INTO v_noactiv_memb
  FROM
    (SELECT MEMBERS.SUBS_ID,
      MEMBERS.START_DATE,
      MEMBERS.END_DATE,
      row_number() OVER (PARTITION BY MEMBERS.SUBS_id ORDER by MEMBERS.START_DATE DESC) rn
    FROM MEMBERS
    ) MEMBERS
  INNER JOIN SUBSCRIBERS
  ON MEMBERS.SUBS_id     = SUBSCRIBERS.subs_id
  WHERE MEMBERS.end_date < sysdate
  AND SUBSCRIBERS.MRF    = p_MRF
  AND RN                 = 1 ;
  RETURN v_noactiv_memb;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  v_noactiv_memb := 0;
WHEN OTHERS THEN
  v_noactiv_memb := -1;
END;
~~~~
### Вызов функции
~~~~
SELECT am_noactiv_memb ('Центр')"Неактивные участники" FROM dual;
~~~~
### Задача №3
-------------
~~~~
Написать функцию, которая возвращает итоговую сумму (BILLS.VALUE) 
по счетам абонентов за январь 2022г. 
   	
	На вход функции подаем дату 01.01.2022. 

Функция должна возвращать числовой результат.
~~~~

### Решение
~~~~
CREATE OR REPLACE FUNCTION am_value_total(
    p_date DATE)
  RETURN NUMBER
AS
  v_selection BILLS%ROWTYPE;
  v_value_total NUMBER:=0;
  CURSOR cur_bills
  IS
    SELECT * FROM bills WHERE start_date = p_date;
BEGIN
  OPEN cur_bills;
  LOOP
    FETCH cur_bills INTO v_selection;
    EXIT
  WHEN cur_bills %NOTFOUND;
    v_value_total:= v_selection.value + v_value_total;
  END LOOP;
CLOSE cur_bills;
RETURN v_value_total;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  v_value_total := 0;
WHEN OTHERS THEN
  v_value_total := -1;
END;
~~~~
### Вызов функции
~~~~
SELECT am_value_total (to_date('01.01.2022','dd.mm.yyyy'))"Итоговая сумма по счетам"
FROM dual;
~~~~

### Задача №4
-------------
~~~~ 
 Написать функцию, которая возвращает количество бонусных баллов (BILLS.VALUE/10) 
 по абоненту 79150933324 за январь 2022г. 
   
   При этом необходимо учесть, что
   «10руб. расходов по счету (Сумма в рублях без НДС) = 1 бонусному баллу»

На вход подаем дату 01.01.2022 и строку 79150933324.
	
Функция должна возвращать целочисленный результат, округленный по математическим правилам.  
~~~~

### Решение
~~~~
CREATE OR REPLACE FUNCTION am_bonus(
    p_date DATE,
    p_abon VARCHAR2)
  RETURN NUMBER
AS
  v_selection_VALUE NUMBER(6,2);
  V_BONUS           NUMBER(6,2);
  CURSOR CUR_BILLS
  IS
    SELECT BILLS.VALUE
    FROM BILLS
    INNER JOIN SUBSCRIBERS
    ON BILLS.SUBS_ID       = SUBSCRIBERS.SUBS_ID
    WHERE BILLS.start_date = p_date
    AND SUBSCRIBERS.MSISDN = p_abon;
BEGIN
  OPEN CUR_BILLS;
  FETCH cur_bills INTO v_selection_VALUE;
  CLOSE CUR_BILLS;
  V_BONUS := v_selection_VALUE/10;
  V_BONUS := ROUND (V_BONUS,0);
  RETURN V_BONUS;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  V_BONUS := 0;
WHEN OTHERS THEN
  V_BONUS := -1;
END;
~~~~
### Вызов функции
~~~~
SELECT am_bonus (to_date('01.01.2022','dd.mm.yyyy'), '79150933324')"Бонус"
FROM dual;
~~~~
### Задача №5
-------------
~~~~ 
 Написать функцию без параметров, возвращающую МРФ (SUBSCRIBERS.MRF), 
 в котором было больше всего участников бонусной программы за все время. 
   	
Функция должна возвращать строковый результат.
~~~~
### Решение
~~~~
CREATE OR REPLACE FUNCTION am_bonus_max
  RETURN VARCHAR2
AS
  v_max_abon     VARCHAR2(100);
  v_count_colums NUMBER(8);
BEGIN
  SELECT *
  INTO v_max_abon,
    v_count_colums
  FROM
    (SELECT DISTINCT SUBSCRIBERS.MRF AS v_mrf,
      COUNT(*)                       AS kolvo_count
    FROM members
    INNER JOIN SUBSCRIBERS
    ON SUBSCRIBERS.subs_id = members.SUBS_ID
    GROUP BY SUBSCRIBERS.MRF
    )
  WHERE kolvo_count =
    (SELECT MAX (kolvo_count)
    FROM
      (SELECT DISTINCT SUBSCRIBERS.MRF AS v_mrf,
        COUNT(*)                       AS kolvo_count
      FROM members
      INNER JOIN SUBSCRIBERS
      ON SUBSCRIBERS.subs_id = members.SUBS_ID
      GROUP BY SUBSCRIBERS.MRF
      )
    );
  RETURN v_max_abon;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  v_max_abon := 'Нет данных';
WHEN OTHERS THEN
  v_max_abon := '-1';
END;
~~~~
### Вызов функции
~~~~
SELECT am_bonus_max FROM dual;
~~~~
### Задача №6
~~~
Определить регионы с количество участников менее 4. 
Для этих регионов, с помощью курсора и цикла, 
начислить каждому из участников по 50 поощрительных баллов (то есть получить и увеличить значение столбца BILLS.VALUE).
Сохранять результат в БД не требуется. 
    Функция должна вернуть числовое значение - общую сумму начисленных баллов.    
~~~~

### Решение
~~~~
CREATE OR REPLACE FUNCTION am_nachislenie_bonus
  RETURN NUMBER
AS
  v_total_abon NUMBER := 0;
  CURSOR cur_mrf
  IS
    SELECT BILLS.SUBS_ID,
      SUM(BILLS.value) value
    FROM SUBSCRIBERS
    INNER JOIN BILLS
    ON SUBSCRIBERS.SUBS_ID = BILLS.SUBS_ID
    INNER JOIN MEMBERS
    ON SUBSCRIBERS.SUBS_ID = MEMBERS.SUBS_ID
    WHERE MRF             IN
      (SELECT MRF
      FROM SUBSCRIBERS
      WHERE END_DATE IS NULL
      GROUP BY MRF
      HAVING COUNT(MRF) < 4
      )
  AND SUBSCRIBERS.END_DATE IS NULL
  AND MEMBERS.END_DATE     IS NULL
  GROUP BY BILLS.SUBS_ID;
BEGIN
  FOR v_select IN cur_mrf
  LOOP
    v_select.value := v_select.value + 50;
    v_total_abon   := v_total_abon   + 50;
  END LOOP;
RETURN v_total_abon;
EXCEPTION
WHEN NO_DATA_FOUND THEN
  v_total_abon := 0;
WHEN OTHERS THEN
  v_total_abon := -1;
END;
~~~~
### Вызов функции
~~~~
SELECT am_nachislenie_bonus FROM dual;
~~~~
