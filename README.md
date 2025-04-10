| **<br/>Лабораторная работа №11 по курсу "PostgreSQL для администраторов баз данных и разработчиков"<br/>"Триггеры, поддержка заполнения витрин"<br/>**|
|---|

<br/>

## Задание:
### * Создать триггер для поддержки витрины в актуальном состоянии;


<br/>

## Решение:

* Создал виртуальную машину, установил Ubuntu 22.04, PostgreSQL v.16
* Создал новый кластер PostgreSQL main2 (sudo pg_createcluster 16 main2)
* Создал таблицы из приложенного файла
* Создал триггер и функцию для таблицы sales
* Операция INSERT работает по следующему принципу: т.к. у нас в отчете накопительные данные, то при добавлении данных сначала необходимо удалить существующие данные для вставляемой позиции из таблицы good_sum_mart, а затем уже вставить новые
* Операция UPDATE работает по следующему принципу: если в таблице sales изменяется код товара (good_id), то происходит удаление данных о продажах из good_sum_mart для старого и нового товара, затем происходит вставка данных для старого и нового товара. Если меняется только количество, то удаляются данные о продажах товара и запорлняются заново
* Операция DELETE удаляет старые данные по товару из таблицы good_sum_mart, затем вставляет новые
* Триггер
```
CREATE OR REPLACE TRIGGER sales_after_trg AFTER INSERT OR UPDATE OR DELETE
ON pract_functions.sales
FOR EACH ROW EXECUTE PROCEDURE my_function();
```
* Триггерная функция
```
CREATE OR REPLACE FUNCTION my_function()
RETURNS trigger
AS
$$
BEGIN

	MERGE INTO good_sum_mart gsm USING (
		SELECT G.good_name, sum(G.good_price * COALESCE(S.sales_qty,0)) sales
			FROM goods G LEFT JOIN sales S ON S.good_id = G.goods_id
		GROUP BY G.good_name
	) g
		ON gsm.good_name = g.good_name
	
    WHEN NOT MATCHED THEN 
		INSERT VALUES (good_name, sales)
		
    WHEN MATCHED AND sum_sale != g.sales AND g.sales > 0 THEN
	    UPDATE SET sum_sale = g.sales
		
	WHEN MATCHED AND sum_sale != g.sales AND g.sales = 0 THEN
		DELETE; 
	
    
    
	RETURN NEW;
END;
$$ LANGUAGE plpgsql;```
* Дописал триггер на случай TRUNCATE sales
```
CREATE TRIGGER sales_after_truncate_trg
AFTER TRUNCATE ON pract_functions.sales
FOR EACH STATEMENT
EXECUTE FUNCTION my_function_for_truncate();


CREATE OR REPLACE FUNCTION my_function_for_truncate()
RETURNS TRIGGER AS
$$
BEGIN
    TRUNCATE good_sum_mart;
    RETURN NULL;
END;
$$
LANGUAGE plpgsql;

```

* Схема создания отчета витрина + триггер предпочтительнее прямого выполнения запроса тем, что при увеличении таблицы запрос будет выполняться дольше, т.е. запрос откроет транзакцию, установит горизонт событий и будет препятствовать удалению мертвых строк. Что приведет к раздуванию таблиц и, как следствие, будет сильнее влиять на снижение производительности