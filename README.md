INSTRUCCIONES DE EJECUCIÓN
1. Levantar el entorno
bash
docker-compose up --build

2. CRÍTICO: Generar datos para consultas OLAP
Las tablas DW estarán VACÍAS hasta ejecutar estos INSERTs:

-- CONECTAR A MySQL
docker exec -it red_social_db mysql -uroot -prootpassword red_social_db

-- GENERAR REACCIONES (50K)
INSERT INTO reacciones (id_publicacion, id_usuario, tipo_reaccion)
SELECT p.id_publicacion, (SELECT id_usuario FROM usuarios ORDER BY RAND() LIMIT 1),
       ELT(1 + FLOOR(RAND()*5), 'like','love','wow','triste','enojo')
FROM publicaciones p WHERE p.visibilidad = 'publico' LIMIT 50000;

-- GENERAR COMENTARIOS (10K)  
INSERT INTO comentarios (id_publicacion, id_usuario, contenido)
SELECT p.id_publicacion, (SELECT id_usuario FROM usuarios ORDER BY RAND() LIMIT 1),
       CONCAT('Comentario ', FLOOR(RAND()*1000))
FROM publicaciones p WHERE p.visibilidad = 'publico' LIMIT 10000;


3. Ejecutar ETL

docker-compose exec app python etl_warehouse.py

4. Verificar datos DW

SELECT COUNT(*) FROM hechos_actividad_social;  -- Debe ser >20,000
SELECT COUNT(*) FROM dim_tiempo;               -- ~366 días

5. Ejecutar consultas OLAP

-- Copiar consultas_olap.sql
docker exec -it red_social_db mysql -uroot -prootpassword red_social_db < consultas_olap.sql

ESTRUCTURA DEL PROYECTO

practica5/
├── schema.sql                 # OLTP (6 tablas normalizadas)
├── dw_schema.sql              # Data Warehouse (estrella)
├── etl_warehouse.py           # ETL completo (Extract→Transform→Load)
├── consultas_olap.sql         # 15+ consultas OLAP
├── poblar_leve.py             # 5K registros
├── poblar_moderado.py         # 120K registros  
├── poblar_masivo.py           # 2.5M registros
├── docker-compose.yml         # MySQL + App Python
└── entrypoint.sh              # Inicialización automática

ADVERTENCIAS IMPORTANTES
SIN INSERTS de reacciones/comentarios:

hechos_actividad_social = 0 filas
consultas_olap.sql = VACÍAS

hechos_actividad_social = 20K+ filas
consultas_olap.sql = RESULTADOS reales

FLUJO COMPLETO (5 minutos)

1. docker-compose up --build
2. INSERT reacciones/comentarios 
3. docker-compose exec app python etl_warehouse.py
4. Verificar COUNT(*) > 20K
5. Ejecutar consultas_olap.sql 
