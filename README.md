--creates rental_detailed table-- 
CREATE TABLE IF NOT EXISTS rental_detailed( 
 rental_id INTEGER, 
 rental_date DATE, 
 customer_id INTEGER, 
 first_name VARCHAR(50), 
 last_name VARCHAR(50), 
 email VARCHAR(75),
 customer_active VARCHAR(3),
 staff_id INTEGER,
 store_id INTEGER
);

--creates rental_summary table-- 
CREATE TABLE IF NOT EXISTS rental_summary( 
 rental_id INTEGER,
 rental_date DATE, 
 customer_id INTEGER, 
 first_name VARCHAR(20), 
 last_name VARCHAR(50), 
 email VARCHAR(50),
 customer_active VARCHAR(3)
); 


--extracts raw data for detailed report-- 
INSERT INTO rental_detailed( 
 rental_id, 
 rental_date, 
 customer_id, 
 first_name, 
 last_name, 
 email,
 customer_active,
 staff_id,
 store_id
)

SELECT r.rental_id, r.rental_date, r.customer_id, cust.first_name, cust.last_name, cust.email, cust.active, r.staff_id, cust.store_id
FROM rental AS r 
INNER JOIN customer AS cust ON cust.customer_id = r.customer_id;


view the now filled in detailed report-- 
SELECT * FROM rental_detailed; 


--creates function to update the rental_summary table-- 
--updates the rental_summary table when new data is added to rental_detailed--
--tranforms the customer_active field from a 1 or 0 into a Yes or No statement-- 
CREATE FUNCTION rental_summary_transformation() 
	RETURNS TRIGGER
	LANGUAGE plpgsql
	AS
$$ 
BEGIN 
	DELETE FROM rental_summary;
	
 	INSERT INTO rental_summary(
		SELECT 
			COUNT(*) AS rental_count,
			rental_id,
			rental_date, 
			customer_id, 
			CONCAT (first_name, ' ', last_name) AS customer_name,
			email,
			CASE customer_active
				WHEN '1' THEN 'Yes'
				WHEN '0' THEN 'No'
			END active
			FROM rental_detailed
			GROUP BY customer_id
);
RETURN NEW;
END;
$$


--creates the trigger to run the rental_summary_transformation function-- 
CREATE TRIGGER rental_summary_trigger
AFTER INSERT ON rental_detailed 
FOR EACH STATEMENT 
EXECUTE PROCEDURE rental_summary_transformation();


--creates a stored procedure to refresh the summary and detailed reports-- 
CREATE PROCEDURE refresh_tables_procedure()
LANGUAGE plpgsql 
AS 
$BODY$ 
BEGIN 
DELETE FROM rental_detailed; 
INSERT INTO rental_detailed( 
 rental_id, 
 rental_date, 
 staff_id,
 customer_id, 
 first_name , 
 last_name, 
 email,
 store_id
) 

SELECT
 r.rental_id, r.rental_date, r.customer_id, cust.first_name, cust.last_name, cust.email, cust.customer_active, r.staff_id, cust.store_id
FROM rental AS r 
INNER JOIN customer AS cust ON cust.customer_id = r.customer_id;
END; 
$BODY$ 

--Run daily to ensure that the customer rewards are always accurate and up to date.--
