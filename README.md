# super_query_mod
modificación de super query3#
------------------------------------------------------------------------------------------------
-------------------------TABLAS TEMPORALES CREADAS----------------------------------------------
 
 ---#leads_lead
 SELECT id,
		created_on
INTO #leads_lead	
FROM leads_lead
---#tracking_session
SELECT id,
        visitor_id,
        url
INTO #tracking_session		
FROM tracking_session
---#tracking_event
SELECT id,
       session_id,
       data
INTO #tracking_event
FROM tracking_event

--- #tracking_visitor
SELECT id,
       lead_id
INTO #tracking_visitor
FROM tracking_visitor

----------------------------------------------------------------------------------------------------------------------


SELECT CASE WHEN leads_lead.id IS NULL 
                THEN 100000000 
                  ELSE leads_lead.id 
		        END      AS lead_id,
       CASE WHEN current_email IS NULL 
	              THEN 'venta@sin_lead' 
				          ELSE current_email 
	          END AS current_email,
       CASE WHEN leads_lead.created_on IS NULL 
	              THEN '1990-01-01' 
			            ELSE leads_lead.created_on 
	          END AS created_on_lead,
      To_char (leads_lead.created_on, 'YYYYMM') AS vintage,
       max_bcs,
       max_num_tradelines,
       most_recent_created_on_respuesta,
       recommendation_1_id,
       recommendation_2_id,
       recommendation_3_id,
       created_on_recommendation,
       ventas_offline.created_on_sale           AS created_on_sale,
       ventas_offline.product_id                AS product_id_sale,
       ventas_offline.sale_type                 AS sale_type,
       hsbc.status_received                     AS status_received_hsbc,
       hsbc.created_on                          AS created_on_pixel_hsbc,
       hsbc.source                              AS source_hsbc,
       bbva.status_received                     AS status_received_bbva,
       bbva.created_on                          AS created_on_pixel_bbva,
       bbva.source                              AS source_bbva,
       bbva.source_bbva                         AS source_bbva_url,
       coppel.status_received                   AS status_received_coppel,
       coppel.created_on                        AS created_on_pixel_coppel,
       coppel_form.status                       AS success_coppel_form,
       coppel_form.created_on                   AS created_on_coppel_form,
       coppel_form.source                       AS source_coppel,
       coppel_form.campaign                     AS coppel_capana,
       creditea.status_received                 AS status_received_creditea,
       creditea.created_on                      AS created_on_creditea,
       creditea.source                          AS source_creditea,
       kueski.status_received                   AS status_received_kueski,
       kueski.created_on                        AS created_on_kueski,
       kueski.source                            AS source_kueski,
       amex.status_received                     AS status_received_amex,
       amex.created_on                          AS created_on_amex,
       amex.source                              AS amex_source,
----------------------------------------------------------------------------------------------------------------       
	     yotepresto_credito.status_received       AS status_received_yotepresto_credito,
       yotepresto_credito.created_on            AS created_on_yotepresto_credito,
       yotepresto_credito.source                AS source_yotepresto_credito,
----------------------------------------------------------------------------------------------------------------       
	     CASE WHEN ventas_offline.created_on_sale >= '2018-10-01'AND ventas_offline.product_id = 205 
               THEN 1700 ELSE 0 
            END AS amount_bbva_approved,
       CASE WHEN ventas_offline.created_on_sale < '2018-10-01' AND ventas_offline.product_id = 205 
               THEN 200 ELSE 0 
            END AS amount_bbva_completed,
       CASE WHEN ventas_offline.created_on_sale IS NOT NULL AND ventas_offline.product_id = 181 
               THEN 85 
            END AS amount_kueski,
       CASE WHEN ventas_offline.created_on_sale IS NOT NULL AND ventas_offline.product_id = 216 
               THEN 1800 ELSE 0 
            END AS amount_hsbc,
       CASE WHEN ventas_offline.created_on_sale IS NOT NULL AND ventas_offline.product_id = 206 
               THEN 670 ELSE 0 
            END AS amount_coppel,
       CASE WHEN ventas_offline.created_on_sale IS NOT NULL AND ventas_offline.product_id = 62 
               THEN 4000 ELSE 0 
            END AS amount_amex,
  ---------------------------------------------------------------------------------------------------          
	     CASE WHEN ventas_offline.created_on_sale IS NOT NULL AND ventas_offline.product_id = 203 
               THEN 1500 ELSE 0
            END AS amount_yotepresto_credito,
 -----------------------------------------------------------------------------------------------------           
        call_center_contact.total_calls,
        call_center_contact.status,
        call_center_contact.subcategory,
        call_center_contact.status_last_call,
        call_center_contact.created_on_contact,
        call_center_contact.created_on_call,
        is_black_listed
FROM   leads_lead
--------------------------------------------- Criterios de seleccion de buro--------------------------------------------------
--- Maximo score que puede tener un usuario, debido a diferentes RFC o prospectors
--- Maximo numero de tradelines, para ver si alguno de sus RFCs tiene cuentas

LEFT JOIN ( SELECT   lead_id,
                  max(valor_score)                AS max_bcs,
                  max(num_tradelines)             AS max_num_tradelines,
                  max(buros.created_on_respuesta)    most_recent_created_on_respuesta
            FROM     (
                            SELECT    lead_id,
                                      buro_respuesta.id AS respuesta_id,
                                      valor_score,
                                      num_tradelines,
                                      buro_respuesta.created_on AS created_on_respuesta
                            FROM      buro_consulta
                            LEFT JOIN (SELECT id,
                                              created_on,
                                              consulta_id
                                       FROM buro_respuesta) AS buro_respuesta
			                           ON buro_respuesta.consulta_id = buro_consulta.id
                            LEFT JOIN (SELECT respuesta_id,
                                              valor_score
                                        FROM buro_bcscorerespuesta) AS  buro_bcscorerespuesta 
			                           ON        buro_bcscorerespuesta.respuesta_id = buro_respuesta.id
                            LEFT JOIN (SELECT   respuesta_id,
                                                count(*) AS num_tradelines
                                        FROM     buro_cuentarespuesta
                                          GROUP BY respuesta_id) AS cuentas 
			                           ON        cuentas.respuesta_id = buro_respuesta.id
                            WHERE     lead_id IS NOT NULL
                            AND       buro_respuesta.id IS NOT NULL
                            ORDER BY  buro_respuesta.created_on DESC) AS buros
         LEFT JOIN #leads_lead
		          ON #leads_lead.id = buros.lead_id
         GROUP BY lead_id
         ) AS conteos_buro ON conteos_buro.lead_id = leads_lead.id
--------------------------------------------- recomendaciones de producto--------------------------------------------------
LEFT JOIN ( SELECT DISTINCT ON ( lead_id) lead_id,
                   buro_recommendation.created_on AS created_on_recommendation,
                   recommendation_1_id,
                   recommendation_2_id,
                   recommendation_3_id
            FROM   buro_recommendation
              LEFT JOIN #leads_lead
		              ON #leads_lead.id = buro_recommendation.lead_id
           ORDER BY   lead_id DESC, buro_recommendation.created_on DESC
           ) AS recomendacion ON recomendacion.lead_id = leads_lead.id
--------------------------------------------- Completudes HSBC ------------------------------------------------------------
LEFT JOIN (SELECT DISTINCT ON (tracking_visitor.lead_id)
                  CASE WHEN tracking_saleevent.lead_id IS NULL 
		                     THEN tracking_visitor.lead_id
                           ELSE tracking_saleevent.lead_id 
                             END AS lead_id,
                  tracking_saleevent.created_on,
                  tracking_saleevent.status_received,
                  CASE WHEN status_received = 'approved' THEN 1
                       WHEN status_received = 'completed' THEN 2
                       WHEN status_received = 'declined-1' THEN 3
                       WHEN status_received = 'declined' THEN 4
                       WHEN status_received = 'domicilio' THEN 5
                       WHEN status_received = 'personal' THEN 6
                         ELSE 7 END AS status_received_valor,
                  product_id,
                  tracking_saleevent.credit_entity_id,
                  tracking_saleevent.source,
                  folio
           FROM tracking_saleevent
             LEFT JOIN #tracking_session 
		              ON #tracking_session.id = tracking_saleevent.conversion_session_id
             LEFT JOIN #tracking_event 
		              ON #tracking_event.id = tracking_saleevent.event_id
             LEFT JOIN leads_creditentity  
		              ON tracking_saleevent.credit_entity_id = leads_creditentity.id
             LEFT JOIN #tracking_visitor 
		              ON #tracking_visitor.id = tracking_session.visitor_id
           WHERE tracking_saleevent.credit_entity_id = 16
             AND tracking_visitor.lead_id IS NOT NULL
		       ORDER BY tracking_visitor.lead_id, 
		            status_received_valor, 
		            cast(tracking_saleevent.created_on as date)
		    ) AS hsbc on hsbc.lead_id = leads_lead.id
--------------------------------------------- Completudes BBVA ------------------------------------------------------------
LEFT JOIN (SELECT DISTINCT ON (lead_id) *
           FROM ( SELECT  CASE WHEN tracking_saleevent.lead_id IS NULL THEN tracking_visitor.lead_id
                                 ELSE tracking_saleevent.lead_id 
				                            END AS lead_id,
                          cast(tracking_saleevent.created_on as TIMESTAMP) as created_on,
                          tracking_saleevent.status_received,
                          product_id,
                          tracking_saleevent.credit_entity_id,
                          bbva_source.source AS source_bbva,
                          folio,
                          tracking_saleevent.id_received,
                          "RFC",
                          tracking_saleevent.id as pixel_id,
                          CASE WHEN status_received = 'approved' THEN 1
                               WHEN status_received = 'declined' THEN 2
                               WHEN status_received = 'noauth' THEN 3
                                 ELSE 4 END AS status_received_bbva,
                          tracking_saleevent.source
                  FROM tracking_saleevent
                    LEFT JOIN #tracking_session 
	                       ON #tracking_session.id = tracking_saleevent.conversion_session_id
                    LEFT JOIN #tracking_event
	                       ON #tracking_event.id = tracking_saleevent.event_id
                    LEFT JOIN leads_creditentity  
	                       ON tracking_saleevent.credit_entity_id = leads_creditentity.id
                    LEFT JOIN #tracking_visitor 
	                       ON #tracking_visitor.id = tracking_session.visitor_id
                    LEFT JOIN (SELECT DISTINCT ON (lead_id)
                                      lead_id,
                                     "RFC"
                               FROM buro_consulta) AS buro_consulta 
	                       ON buro_consulta.lead_id = tracking_saleevent.lead_id
                    LEFT JOIN (SELECT DISTINCT ON (aux.pixel) *
                               FROM  (SELECT SALE.id    AS Pixel,
                                             SALE.created_on      AS created_on_sale,
                                             leads_redirect.created_on    AS created_on_redirect,
                                            ( SALE.created_on - leads_redirect.created_on ) AS diff_date,
                                             CASE WHEN SALE.source IS NULL
                                                THEN leads_redirect.source
                                                   ELSE SALE.source 
                                                      END AS source
                                      FROM tracking_saleevent) AS SALE  
                                        LEFT JOIN leads_redirect 
				                                     ON SALE.lead_id = leads_redirect.lead_id
                                      WHERE credit_entity_id = 10
                                      ORDER  BY diff_date) AS AUX
                               WHERE  created_on_sale >= created_on_redirect ) AS bbva_source 
	   ON tracking_saleevent.id = bbva_source.Pixel
  WHERE credit_entity_id = 10
    AND tracking_saleevent.lead_id IS NOT NULL
  ORDER BY  CASE WHEN status_received = 'approved' THEN 1
  WHEN status_received = 'declined' THEN 2
  WHEN status_received = 'noauth' THEN 3 END
) AS todo_pixel_bbva
ORDER BY lead_id, status_received_bbva) AS bbva ON bbva.lead_id =  leads_lead.id
--------------------------------------------- Completudes creditea ------------------------------------------------------------
LEFT JOIN (SELECT DISTINCT ON (tracking_visitor.lead_id)
                  CASE WHEN tracking_saleevent.lead_id IS NULL THEN tracking_visitor.lead_id
                         ELSE tracking_saleevent.lead_id 
                           END AS lead_id,
                  tracking_saleevent.created_on,
                  tracking_saleevent.status_received,
                  CASE WHEN status_received = 'completed' THEN 1
                       WHEN status_received = 'creditea-aprobado' THEN 2
                       WHEN status_received = 'creditea-declinado' THEN 3
                       WHEN status_received = 'creditea-buro' THEN 4
                       WHEN status_received = 'completed' THEN 5
                       WHEN status_received = 'creditea-completado' THEN 6
                       WHEN status_received = 'creditea-iniciar' THEN 7
                         ELSE 8 END AS status_received_valor,
                  product_id,
                  tracking_saleevent.credit_entity_id,
                  tracking_saleevent.source,
                  folio
           FROM tracking_saleevent
             LEFT JOIN #tracking_session 
              	ON #tracking_session.id = tracking_saleevent.conversion_session_id
             LEFT JOIN #tracking_event 
	             ON #tracking_event.id = tracking_saleevent.event_id
             LEFT JOIN leads_creditentity  
               ON tracking_saleevent.credit_entity_id = leads_creditentity.id
             LEFT JOIN #tracking_visitor 
               ON tracking_visitor.id = tracking_session.visitor_id
           WHERE tracking_saleevent.credit_entity_id = 47
             AND tracking_visitor.lead_id IS NOT NULL
           ORDER BY tracking_visitor.lead_id, status_received_valor, cast(tracking_saleevent.created_on as date)
) AS creditea on creditea.lead_id = leads_lead.id
--------------------------------------------- Completudes kueski ------------------------------------------------------------
LEFT JOIN (SELECT distinct on (tracking_saleevent.lead_id) *
           FROM (SELECT  CASE WHEN tracking_saleevent.lead_id IS NULL THEN tracking_visitor.lead_id
                            ELSE tracking_saleevent.lead_id 
                              END AS lead_id,
                         tracking_saleevent.created_on,
                         tracking_saleevent.status_received,
                         CASE WHEN status_received = 'aprobado' THEN 1
                              WHEN status_received = 'scoring' THEN 2
                              WHEN status_received = 'rejected' THEN 3
                                 ELSE 4 END AS status_received_valor,
                         product_id,
                         tracking_saleevent.credit_entity_id,
                         tracking_saleevent.source,
                         folio
                 FROM tracking_saleevent
                   LEFT JOIN #tracking_session
				             ON #tracking_session.id = tracking_saleevent.conversion_session_id
                   LEFT JOIN #tracking_event 
				             ON #tracking_event.id = tracking_saleevent.event_id
                   LEFT JOIN leads_creditentity  
				             ON tracking_saleevent.credit_entity_id = leads_creditentity.id
                   LEFT JOIN #tracking_visitor
				             ON #tracking_visitor.id = tracking_session.visitor_id
                 WHERE tracking_saleevent.credit_entity_id =  29
                    AND tracking_visitor.lead_id IS NOT NULL
                 ORDER BY  cast(tracking_saleevent.created_on as date) desc, 
				                   tracking_saleevent.status_received 
				        ) AS tracking_saleevent
         ) AS kueski on kueski.lead_id = leads_lead.id

--------------------------------------------- Completudes amex ------------------------------------------------------------
LEFT JOIN (SELECT DISTINCT ON (tracking_saleevent.lead_id)
                     CASE WHEN tracking_saleevent.lead_id IS NULL 
		                        THEN tracking_visitor.lead_id
                               ELSE tracking_saleevent.lead_id 
                                 END AS lead_id,
                  tracking_saleevent.created_on,
                  tracking_saleevent.status_received,
                  CASE WHEN status_received = 'amex-01' THEN 1
                       WHEN status_received = 'amex-02' THEN 2
                       WHEN status_received = 'amex-03' THEN 3
                       WHEN status_received = '004' THEN 4
                       WHEN status_received = 'Invalid' THEN 5
                          ELSE 6 
                            END AS status_received_valor,
                  product_id,
                  tracking_saleevent.credit_entity_id,
                  tracking_saleevent.source,
                  folio
           FROM tracking_saleevent
             LEFT JOIN #tracking_session 
		           ON #tracking_session.id = tracking_saleevent.conversion_session_id
             LEFT JOIN #tracking_event
		           ON #tracking_event.id = tracking_saleevent.event_id
             LEFT JOIN leads_creditentity 
		           ON tracking_saleevent.credit_entity_id = leads_creditentity.id
             LEFT JOIN #tracking_visitor 
		           ON #tracking_visitor.id = tracking_session.visitor_id
           WHERE tracking_saleevent.credit_entity_id =  3
              AND tracking_visitor.lead_id IS NOT NULL
           ORDER BY tracking_saleevent.lead_id, status_received_valor, cast(tracking_saleevent.created_on as date)
          ) AS amex on amex.lead_id = leads_lead.id
----------------------------------------------------------------------------------------------------------		  
---------------------------------------------ESTATUS PRUEBA YOTEPRESTO_CREDITO----------------------------	  
 
LEFT JOIN (SELECT DISTINCT ON (tracking_visitor.lead_id)
                  CASE WHEN tracking_saleevent.lead_id IS NULL THEN tracking_visitor.lead_id
                       ELSE tracking_saleevent.lead_id END AS lead_id,
                  tracking_saleevent.created_on,
                  tracking_saleevent.status_received,
                  CASE WHEN status_received = 'APROBADA' THEN 1
		                WHEN status_received = 'PRE-APROBADA' THEN 2
                       WHEN status_received = 'DECLINADA' THEN 3
                       ELSE 4 END AS status_received_valor,
                  product_id,
                  tracking_saleevent.credit_entity_id,
                  tracking_saleevent.source,
                  folio
           FROM tracking_saleevent
             LEFT JOIN #tracking_session 
	             ON #tracking_session.id = tracking_saleevent.conversion_session_id
             LEFT JOIN #tracking_event 
	             ON #tracking_event.id = tracking_saleevent.event_id
             LEFT JOIN leads_creditentity  
               ON tracking_saleevent.credit_entity_id = leads_creditentity.id
             LEFT JOIN #tracking_visitor 
	             ON #tracking_visitor.id = tracking_session.visitor_id
           WHERE tracking_saleevent.credit_entity_id =41 OR  tracking_saleevent.credit_entity_id =51
             AND tracking_visitor.lead_id IS NOT NULL
           ORDER BY tracking_visitor.lead_id, 
                    status_received_valor, 
                    cast(tracking_saleevent.created_on as date)
         )AS yotepresto_credito on yotepresto_credito.lead_id = leads_lead.id		  
		  
		  
----------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------- estatus coppel en pixel ------------------------------------------------------------
LEFT JOIN (SELECT distinct on (lead_id)
                   lead_id,
                   status_received,
                   created_on
           FROM tracking_saleevent
           WHERE credit_entity_id= 43
             AND lead_id IS NOT NULL
           ORDER BY lead_id DESC, created_on DESC
           ) AS coppel 
  ON coppel.lead_id = leads_lead.id
----------------------------------------------Formalizadas en tabla de ventas offline---------------------------------------------
RIGHT JOIN (SELECT lead_id,
                    created_on_sale::timestamp::date AS created_on_sale,
                    sale_type,
                    product_id
            FROM tracking_saleoffline 
            where credit_entity_id = 10
               AND lead_id > 0
        UNION ALL 
            SELECT DISTINCT ON (lead_id)
                   lead_id,
                   created_on_sale::timestamp::date AS created_on_sale,
                   sale_type,
                   product_id
            FROM tracking_saleoffline
            WHERE credit_entity_id = 3
              AND lead_id > 0
        UNION ALL
            SELECT DISTINCT ON (lead_id)
                   lead_id,
                   created_on_sale::timestamp::date AS created_on_sale,
                   sale_type,
                   product_id
            FROM tracking_saleoffline
            WHERE credit_entity_id = 43
              AND lead_id > 0
        UNION ALL
            SELECT DISTINCT ON (lead_id)
                   lead_id,
                   created_on_sale::timestamp::date AS created_on_sale,
                   sale_type,
                   product_id
            FROM tracking_saleoffline
            WHERE credit_entity_id = 16
              AND lead_id > 0
-------------------------------------------------------------------------------------------			
		    UNION ALL 		
            SELECT DISTINCT ON (lead_id)
                   lead_id,
                   created_on_sale::timestamp::date AS created_on_sale,
                   sale_type,
                   product_id
            FROM tracking_saleoffline
            WHERE credit_entity_id = 41 or credit_entity_id = 51
              AND lead_id > 0
---------------------------------------------------------------------------------------------			
		UNION ALL
            SELECT distinct on (tracking_saleevent.folio)
                      CASE WHEN lead_id IS NULL THEN 100000000 ELSE lead_id END AS lead_id, 
                   CAST (created_on AT TIME ZONE 'CDT' AS DATE) AS created_on_sale,
                  'completud' AS sale_type,
                   181 AS product_id
            FROM ( SELECT distinct on (folio)
                         CASE WHEN tracking_saleevent.lead_id IS NULL THEN tracking_visitor.lead_id
                                ELSE tracking_saleevent.lead_id 
                                  END AS lead_id,
                         tracking_saleevent.created_on,
                         tracking_saleevent.status_received,
                         CASE WHEN status_received = 'approved' THEN 1
                              WHEN status_received = 'scoring' THEN 2
                              WHEN status_received = 'rejected' THEN 3
                                ELSE 4 
                                  END AS status_received_valor,
                         product_id,
                         tracking_saleevent.credit_entity_id,
                         tracking_saleevent.source,
                         folio
                  FROM tracking_saleevent
                    LEFT JOIN #tracking_session 
				              ON #tracking_session.id = tracking_saleevent.conversion_session_id
                    LEFT JOIN #tracking_event 
				              ON #tracking_event.id = tracking_saleevent.event_id
                    LEFT JOIN leads_creditentity  
				              ON tracking_saleevent.credit_entity_id = leads_creditentity.id
                    RIGHT JOIN #tracking_visitor
				              ON #tracking_visitor.id = tracking_session.visitor_id
                  WHERE tracking_saleevent.credit_entity_id =  29
                    AND status_received IN ('approved')
                    AND folio != 'undefined') AS tracking_saleevent
       UNION ALL
            SELECT DISTINCT ON (lead_id) lead_id, 
                   CAST (created_on AT TIME ZONE 'CDT' AS DATE) AS created_on_sale,
                  'completud' AS sale_type,
                   205 AS product_id
            FROM (SELECT  CASE WHEN tracking_saleevent.lead_id IS NULL 
				               THEN tracking_visitor.lead_id
                               ELSE tracking_saleevent.lead_id END AS lead_id,
                          cast(tracking_saleevent.created_on as TIMESTAMP) as created_on,
                          tracking_saleevent.status_received,
                          product_id,
                          tracking_saleevent.credit_entity_id,
                          bbva_source.source AS source_bbva,
                          folio,
                          tracking_saleevent.id_received,
                          "RFC",
                          tracking_saleevent.id as pixel_id,
                          CASE WHEN status_received = 'approved' THEN 1
                               WHEN status_received = 'declined' THEN 2
                               WHEN status_received = 'noauth' THEN 3
                               ELSE 4 END AS status_received_bbva,
                          tracking_saleevent.source
                  FROM tracking_saleevent
                    LEFT JOIN #tracking_session 
				              ON #tracking_session.id = tracking_saleevent.conversion_session_id
                    LEFT JOIN #tracking_event 
				              ON #tracking_event.id = tracking_saleevent.event_id
                    LEFT JOIN leads_creditentity  
				              ON tracking_saleevent.credit_entity_id = leads_creditentity.id
                    LEFT JOIN #tracking_visitor 
				              ON #tracking_visitor.id = tracking_session.visitor_id
                    LEFT JOIN (SELECT DISTINCT ON (lead_id) lead_id,
                                      "RFC"
                                FROM buro_consulta ) AS buro_consulta 
				              ON buro_consulta.lead_id = tracking_saleevent.lead_id
                    LEFT JOIN (SELECT DISTINCT ON (aux.pixel) *
                               FROM  (SELECT SALE.id   AS Pixel,
                                               SALE.created_on  AS created_on_sale,
                                               leads_redirect.created_on   AS created_on_redirect,
                                               ( SALE.created_on - leads_redirect.created_on ) AS diff_date,
                                               CASE WHEN SALE.source IS NULL 
										           THEN leads_redirect.source
                                                   ELSE SALE.source END AS source
                                        FROM tracking_saleevent AS SALE
                                          LEFT JOIN leads_redirect ON SALE.lead_id = leads_redirect.lead_id
                                        WHERE credit_entity_id = 10
                                        ORDER  BY diff_date) AS AUX
                               WHERE  created_on_sale >= created_on_redirect ) AS bbva_source
				       ON tracking_saleevent.id = bbva_source.Pixel
                  WHERE credit_entity_id = 10
                    AND tracking_saleevent.lead_id IS NOT NULL
                    AND CAST (tracking_saleevent.created_on AT TIME ZONE 'CDT' AS DATE) < '2018-10-01'
                    AND tracking_saleevent.status_received IN ('approved', 'declined')
                  ORDER BY CASE WHEN status_received = 'approved' THEN 1
                                WHEN status_received = 'declined' THEN 2
                                WHEN status_received = 'noauth' THEN 3 END
                  ) AS completud_bbva
           ) AS ventas_offline on ventas_offline.lead_id = leads_lead.id

--------------------------------------------- crm ------------------------------------------------------------
LEFT JOIN (SELECT DISTINCT ON (lead_id)
                  lead_id,
                  call_center_contact.created_on AS created_on_contact,
                  call_center_call.created_on AS created_on_call,
                  total_calls,
                  status,
                  subcategory,
                  status_last_call
           FROM call_center_contact
             LEFT JOIN  #leads_lead AS leads_lead 
		           ON leads_lead.id = call_center_contact.lead_id
             LEFT JOIN (SELECT DISTINCT ON (contact_id)
                                contact_id,
                                status AS status_last_call,
                                created_on
                        FROM call_center_call
                        ORDER BY contact_id, created_on DESC ) AS call_center_call 
		           ON call_center_call.contact_id = call_center_contact.id
          ) AS call_center_contact 
   ON call_center_contact.lead_id = leads_lead.id
---------------------------------------------prueba coppel---------------------------------------
LEFT JOIN (SELECT lower("email") as email,
                   status,
                   CAST (created_on AT TIME ZONE 'CDT' AS TIMESTAMP) AS created_on,
                   source,
                   campaign
           FROM (SELECT  distinct on (email) "created_on",
				         "status", 
				         "email", 
				         source, 
				         campaign
                 FROM(SELECT response,
                             "email",
                             status,
                             "created_on",
                             case when (status LIKE '%Exito_en_la_inyeccion%') then 1 end as "nivel",
                             CAST (created_on AT TIME ZONE 'CDT' AS TIMESTAMP) AS "Fecha",
                             source,
                             campaign
                      FROM (SELECT response, 
							                     created_on,
							                    "email",
							                     status, 
							                     source,
                                   campaign
                            FROM(SELECT response, 
								                        created_on,
								                        "email",
                                        CASE WHEN (response LIKE '%ERROR_WSVALIDACION%') 
                                                THEN 'Exito_en_la_inyeccion'
                                             WHEN (response LIKE '%Se registro la solicitud correctamente.%') 
                                                THEN 'Exito_en_la_inyeccion'
                                             WHEN (response LIKE '%Se registro la solicitud correctamente%')
                                                THEN 'Exito_en_la_inyeccion'
                                             ELSE 'otro' end as "status" ,
                                        CONCAT(name ,' ' ,last_name ,' ' ,second_last_name) as "nombre_completo",
                                        source,
                                        campaign
                                 FROM custom_forms_coppel) AS "UNICOS" 
						   ) As "totales"
                     ) as A1
                 ORDER BY email, nivel asc
              ) AS resultado
           WHERE status = 'Exito_en_la_inyeccion'
             AND source = 'rocket'
             AND campaign = 'recommendation'
          ) AS coppel_form 
  on coppel_form.email = lower(leads_lead.current_email)
---------------------------------------------Filtros globales del query---------------------------------------
---AND (ventas_bbva.created_on_sale <> '' OR ventas_hsbc.created_on_sale <> '' OR ventas_coppel.created_on_sale <> '')  
		  

     
       
