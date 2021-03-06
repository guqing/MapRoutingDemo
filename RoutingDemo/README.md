﻿运行环境geoserver、postgressql 、postgis等

基于pGrouting最短路径导航实例

使用PostGis进行多边形坐标转换实例



--带方向的路网搜索，下面为造示例数据sql语句

CREATE TABLE edge_table (
id BIGSERIAL,
dir character varying,
source BIGINT,
target BIGINT,
cost FLOAT,
reverse_cost FLOAT,
capacity BIGINT,
reverse_capacity BIGINT,
category_id INTEGER,
reverse_category_id INTEGER,
x1 FLOAT,
y1 FLOAT,
x2 FLOAT,
y2 FLOAT,
the_geom geometry
);

INSERT INTO edge_table (
category_id, reverse_category_id,
cost, reverse_cost,
capacity, reverse_capacity,
x1, y1,
x2, y2) VALUES
(3, 1, 1, 1, 80, 130, 2, 0, 2, 1),
(3, 2, -1, 1, -1, 100, 2, 1, 3, 1),
(2, 1, -1, 1, -1, 130, 3, 1, 4, 1),
(2, 4, 1, 1, 100, 50, 2, 1, 2, 2),
(1, 4, 1, -1, 130, -1, 3, 1, 3, 2),
(4, 2, 1, 1, 50, 100, 0, 2, 1, 2),
(4, 1, 1, 1, 50, 130, 1, 2, 2, 2),
(2, 1, 1, 1, 100, 130, 2, 2, 3, 2),
(1, 3, 1, 1, 130, 80, 3, 2, 4, 2),
(1, 4, 1, 1, 130, 50, 2, 2, 2, 3),
(1, 2, 1, -1, 130, -1, 3, 2, 3, 3),
(2, 3, 1, -1, 100, -1, 2, 3, 3, 3),
(2, 4, 1, -1, 100, -1, 3, 3, 4, 3),
(3, 1, 1, 1, 80, 130, 2, 3, 2, 4),
(3, 4, 1, 1, 80, 50, 4, 2, 4, 3),
(3, 3, 1, 1, 80, 80, 4, 1, 4, 2),
(1, 2, 1, 1, 130, 100, 0.5, 3.5, 1.999999999999,3.5),
(4, 1, 1, 1, 50, 130, 3.5, 2.3, 3.5,4);
UPDATE edge_table SET the_geom = st_makeline(st_point(x1,y1),st_point(x2,y2)),
dir = CASE WHEN (cost>0 AND reverse_cost>0) THEN 'B' -- both ways 双向可通行 
WHEN (cost>0 AND reverse_cost<0) THEN 'FT' -- direction of the LINESSTRING 正向通行
WHEN (cost<0 AND reverse_cost>0) THEN 'TF' -- reverse direction of the LINESTRING 反向同行
ELSE '' END; -- unknown

--创建路网拓扑表
SELECT pgr_createTopology('edge_table',0.001,'the_geom','id','source','target',clean:=true);

--分析边缘表的边缘和顶点
SELECT pgr_analyzeGraph('edge_table',0.001,'the_geom','id','source','target');





select a.* 
from edge_table a
inner join   pgr_dijkstra('  
SELECT id,                      
source,                         
target,                        
cost,reverse_cost
FROM edge_table',%a%, %b%,true) b
on a.id = b.edge



用pgrouting对openstreetmap数据进行路径分析

1，将道路osm数据增加列字段

ALTER TABLE changsharoad_connected ADD COLUMN source integer;

ALTER TABLE changsharoad_connected ADD COLUMN target integer;
ALTER TABLE changsharoad_connected ADD COLUMN length double precision;
ALTER TABLE changsharoad_connected ADD COLUMN cost double precision;
ALTER TABLE changsharoad_connected ADD COLUMN reverse_cost double precision;

2，创建拓扑及索引
SELECT pgr_createTopology('changsharoad_connected',0.000001, 'geom', 'gid');
create index road_source_idx on changsharoad_connected("source");
create index road_target_idx on changsharoad_connected("target");

3，分析单双向
SELECT pgr_analyzeOneway('changsharoad_connected', ARRAY['', 'B', 'T'], ARRAY['', 'B', 'F'],ARRAY['', 'B', 'F'], ARRAY['', 'B', 'T'])
update changsharoad_connected set length = ST_LengthSpheroid(geom, 'SPHEROID["WGS 84",6378137,298.257223563]');
update changsharoad_connected set cost=length where oneway='B' or oneway='F'
update changsharoad_connected set cost=-length where oneway='T'
update changsharoad_connected set reverse_cost=-length where oneway='F'
update changsharoad_connected set reverse_cost=length where oneway='B' or oneway='T'

4，最短路径分析
drop table if exists dijkstra_res;
SELECT seq, id1 AS node, id2 AS edge, a.cost as cost, geom into dijkstra_res FROM pgr_dijkstra('     
    SELECT gid AS id,
        source::integer,
        target::integer,
        cost::double precision AS cost,
        reverse_cost::double precision AS reverse_cost
    FROM changsharoad_connected', 20946, 19758, true, true)a 
LEFT JOIN changsharoad_connected b 
ON (a.id2 = b.gid) order by seq;