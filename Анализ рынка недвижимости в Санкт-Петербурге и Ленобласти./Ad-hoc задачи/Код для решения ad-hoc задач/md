-- Время активности объявлений
-- Определение аномальных значений (выбросов) по значению перцентилей:
WITH limits AS (
    SELECT
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY total_area) AS total_area_limit,
        PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY rooms) AS rooms_limit,
        PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY balcony) AS balcony_limit,
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY ceiling_height) AS ceiling_height_limit_h,
        PERCENTILE_CONT(0.01) WITHIN GROUP (ORDER BY ceiling_height) AS ceiling_height_limit_l
    FROM real_estate.flats
),
-- id объявлений, которые не содержат выбросы, также оставим пропущенные данные:
filtered_id AS(
    SELECT id
    FROM real_estate.flats
    WHERE
        total_area < (SELECT total_area_limit FROM limits)
        AND (rooms < (SELECT rooms_limit FROM limits) OR rooms IS NULL)
        AND (balcony < (SELECT balcony_limit FROM limits) OR balcony IS NULL)
        AND ((ceiling_height < (SELECT ceiling_height_limit_h FROM limits)
            AND ceiling_height > (SELECT ceiling_height_limit_l FROM limits)) OR ceiling_height IS NULL)
    )

SELECT 
COUNT (f.id) AS count_of_advertisements,
CASE
	WHEN city = 'Санкт-Петербург' THEN 'Санкт-Петербург'
	ELSE 'Лен.область'
	END AS city_category,
CASE
	WHEN days_exposition between 1 AND 30 THEN 'до 1 месяца'
	WHEN days_exposition between 31 AND 90 THEN 'от 1 месяца до 3'
	WHEN days_exposition between 91 AND 180 THEN 'от 3 месяцев до 6'
	WHEN days_exposition >= 181 THEN 'от 6 месяцев'
	WHEN days_exposition IS null THEN 'non_category'
	END AS days_exposition_category,
ROUND(AVG (a.last_price::numeric / f.total_area::numeric),2) AS avg_price_m2,
ROUND(AVG (a.last_price::numeric),2) AS avg_price,
ROUND(AVG (f.total_area::numeric),2) AS avg_total_area,
ROUND(AVG (f.living_area::numeric),2) AS avg_living_area,
ROUND(AVG (f.ceiling_height::numeric),2) AS avg_ceiling_height,
PERCENTILE_DISC(0.50) WITHIN GROUP (ORDER BY f.rooms) AS avg_rooms,
PERCENTILE_DISC(0.50) WITHIN GROUP (ORDER BY f.balcony) AS avg_balcony,
PERCENTILE_DISC(0.50) WITHIN GROUP (ORDER BY f.floor) AS avg_floor,
ROUND(AVG (f.airports_nearest::numeric),0) AS avg_airports_nearest,
ROUND(AVG (f.parks_around3000::numeric),1) AS avg_parks_around3000,
ROUND(AVG (f.ponds_around3000::numeric),1) AS avg_ponds_around3000
FROM real_estate.flats AS f
JOIN filtered_id AS re USING (id)--добавляем отфильтрованные данные
LEFT JOIN real_estate.advertisement AS a USING (id)
LEFT JOIN real_estate.city AS c USING (city_id)
LEFT JOIN real_estate.type AS t USING (type_id)
WHERE TYPE = 'город' AND EXTRACT(YEAR FROM a.first_day_exposition) BETWEEN 2015 AND 2018
GROUP BY city_category,days_exposition_category
ORDER BY city_category DESC;



-- Сезонность объявлений
-- Определение аномальных значений (выбросов) по значению перцентилей:
WITH limits AS (
    SELECT
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY total_area) AS total_area_limit,
        PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY rooms) AS rooms_limit,
        PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY balcony) AS balcony_limit,
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY ceiling_height) AS ceiling_height_limit_h,
        PERCENTILE_CONT(0.01) WITHIN GROUP (ORDER BY ceiling_height) AS ceiling_height_limit_l
    FROM real_estate.flats
),
--id объявлений, которые не содержат выбросы, также оставим пропущенные данные:
filtered_id AS(
    SELECT id
    FROM real_estate.flats
    WHERE
        total_area < (SELECT total_area_limit FROM limits)
        AND (rooms < (SELECT rooms_limit FROM limits) OR rooms IS NULL)
        AND (balcony < (SELECT balcony_limit FROM limits) OR balcony IS NULL)
        AND ((ceiling_height < (SELECT ceiling_height_limit_h FROM limits)
            AND ceiling_height > (SELECT ceiling_height_limit_l FROM limits)) OR ceiling_height IS NULL)
    ),

-- id объявлений (СТЕ filtered_id), которые не содержат выбросы при анализе данных
months_of_advertisements AS (
	SELECT 
		f.id,
		a.last_price,
		f.total_area,
	EXTRACT(MONTH FROM a.first_day_exposition) AS post_month, -- месяц публикации объявления
	EXTRACT(MONTH FROM a.first_day_exposition + (interval '1' DAY * a.days_exposition)) AS sell_month -- месяц продажи
	FROM real_estate.flats AS f 
	JOIN filtered_id AS fi USING (id) --добавляем отфильтрованные данные
	LEFT JOIN real_estate.advertisement AS a USING (id)
	LEFT JOIN real_estate.type AS t USING (type_id)
	WHERE TYPE = 'город' AND EXTRACT(YEAR FROM a.first_day_exposition) BETWEEN 2015 AND 2018
	),
avg_stats_post AS ( --статистика по месяцу публикации объявления
	SELECT
		post_month,
	COUNT(*) AS count_advertisements,
	ROUND(AVG(last_price::numeric / total_area::numeric),2) AS avg_price_m2, --средняя стоимость квадратного метра
	ROUND(AVG(total_area::numeric),2) AS avg_total_area --средняя площадь недвижимости
	FROM months_of_advertisements
	GROUP BY post_month
	),
avg_stats_sell AS ( --статистика по месяцу продажи
	SELECT
		sell_month,
	COUNT(*) AS count_sold_advertisements,
	ROUND(AVG(last_price::numeric / total_area::numeric),2) AS avg_price_m2, --средняя стоимость квадратного метра
	ROUND(AVG(total_area::numeric),2) AS avg_total_area --средняя площадь недвижимости
	FROM months_of_advertisements
	GROUP BY sell_month
)
SELECT --объединяем таблицы
	COALESCE(a_s_p.post_month, a_s_s.sell_month) AS №_month, 
    COALESCE(a_s_p.count_advertisements, 0) AS count_post, 
    RANK () OVER (ORDER BY a_s_p.count_advertisements DESC) AS post_rank,
    COALESCE(a_s_p.avg_price_m2, 0) AS avg_price_m2_post,
    COALESCE(a_s_p.avg_total_area, 0) AS avg_total_area_post,
    COALESCE(a_s_s.count_sold_advertisements, 0) AS count_sell,
    RANK () OVER (ORDER BY a_s_s.count_sold_advertisements DESC) AS sell_rank,
    COALESCE(a_s_s.avg_price_m2, 0) AS avg_price_m2_sell,
    COALESCE(a_s_s.avg_total_area, 0) AS avg_total_area_sell
FROM avg_stats_sell AS a_s_s
FULL JOIN avg_stats_post AS a_s_p ON a_s_s.sell_month = a_s_p.post_month
WHERE COALESCE(a_s_p.post_month, a_s_s.sell_month) IS NOT NULL
ORDER BY №_month;
