# Zoo_app
―――
ESTRUCTURA GENERAL
El proyecto consta de varios componentes:
• zoo_etl.py
– Define la configuración de la BD, las funciones ETL (create_db, create_tables, extract, transform, load)
– Incluye pruebas de calidad básicas (run_quality_tests) y avanzadas (run_additional_checks)
– Orquesta todo en run_etl().
• test_zoo_etl.py
– Pruebas unitarias con pytest para las transformaciones.
―――
2) CONFIGURACIÓN Y HELPERS

• DBConfig (dataclass)
– Almacena host, port, user, password, database.
– Genera sqlalchemy_url para crear el engine de SQLAlchemy.

• get_mysql_conn(init=False)
– Si init=True devuelve conexión sin database (para crearla).
– Si init=False se conecta a la base indicada.

• engine = create_engine(dbcfg.sqlalchemy_url)
– Motor de SQLAlchemy para lecturas y escrituras en bloque (pandas.to_sql o ejecuciones SQL).

―――
3) INFRAESTRUCTURA

• create_database(dbcfg)
– Usando get_mysql_conn(init=True), ejecuta CREATE DATABASE IF NOT EXISTS ….

• create_tables(dbcfg)
– Con SQLAlchemy ejecuta dos DDL:
– Tabla zoologicos con UNIQUE(nombre,ciudad,pais).
– Tabla animales con UNIQUE(numero_identificacion) y FK→zoologicos(id) ON DELETE CASCADE.

―――
4) EXTRACCIÓN (EXTRACT)

• extract_zoos(path)
– Lee zoos.csv con pandas y devuelve un DataFrame.

• extract_animals(path)
– Lee animales.csv con pandas y devuelve un DataFrame.

―――
5) TRANSFORMACIÓN (TRANSFORM)

• transform_zoos(df)
– Valida y normaliza:
– tamanio_m2, especies_total, presupuesto_anual ≥ 0
– Asegura tipos Python nativos (float, int).

• fetch_zoo_map(dbcfg)
– Consulta SELECT id, nombre FROM zoologicos y construye dict {nombre: id}
– Necesario para asignar claves foráneas a los animales.

• transform_animals(df, zoo_map)
– Elimina filas con datos nulos en campos obligatorios.
– Crea la columna zoologico_id mapeando df["zoologico"] al dict.
– Lanza error si algún animal queda sin correspondiente zoo_map.
– Convierte anio_nacimiento a int puro.
– Elimina la columna textual zoologico.

―――
6) CARGA (LOAD)

• load_zoos(df, dbcfg) – filtrado de duplicados
1. Consulta los zoológicos ya insertados (SELECT nombre,ciudad,pais).
2. Anti-join en pandas para quedarse sólo con los nuevos.
3. df_nuevos.to_sql(... if_exists="append").

• load_animals(df, dbcfg) – filtrado de duplicados
1. Consulta SELECT numero_identificacion FROM animales.
2. Selecciona en pandas sólo los numero_identificacion que no estén ya en BD.
3. df_nuevos_anim.to_sql(... if_exists="append").

Este patrón hace que el ETL sea idempotente: puedes ejecutarlo varias veces sin reprocesar ni chocar contra constraints.
