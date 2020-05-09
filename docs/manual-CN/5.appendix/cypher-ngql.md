# Cypher 和 nGQL

## 基本概念对比

|概念名称               | Cypher | nGQL          |
| --- | --- | --- |
| vertex, node       | node  | vertex        |
| edge, relationship | relationship    | edge          |
| vertex type        | label   | tag           |
| edge type          | relationship type   | edge type     |
| vertex identifier          | node id generated by default | vid           |
| edge identifier        | edge id generated by default   |<src, dst, rank>  |

## 图基本操作

|操作                   | Cypher         | nGQL          |
| --- | ------------ | ------------ |
| 列出所有 labels/tags   | * MATCH (n) RETURN distinct labels(n);  <br/> * call db.labels(); | SHOW TAGS |
| 插入指定类型的点 | CREATE (:Person {age: 16}) | INSERT VERTEX <tag_name> (prop_name_list) VALUES \<vid>:(prop_value_list) |
| 插入指定类型的边| CREATE (src)-[:LIKES]->(dst) <br/> SET rel.prop = V | INSERT EDGE <edge_type> ( <prop_name_list> ) VALUES <src_vid> -> <dst_vid>[@<ranking>]: ( <prop_value_list> ) |
| 删除点 | MATCH (n) WHERE ID(n) = vid <br/> DETACH DELETE n | DELETE VERTEX \<vid> |
| 删除边  | MATCH ()-[r]->() WHERE ID(r)=edgeID <br/> DELETE r | DELETE EDGE <edge_type> \<src_vid> -> \<dst_vid>[@<ranking>] |
| 更新点属性 |SET n.name = V | UPDATE VERTEX \<vid> SET <update_columns> |
| 查询指定点的属性| MATCH (n) <br/> WHERE ID(n) = vid  <br/> RETURN properties(n) | FETCH PROP ON <tag_name> \<vid>|
| 查询指定边的属性  | MATCH (n)-[r]->() <br/> WHERE ID(r)=edgeID <br/> return properties(r)| FETCH PROP ON <edge_type> <src_vid> -> <dst_vid>[@<ranking>] |
| 查询指定点的某一类关系 |MATCH (n)-[r:edge_type]->() WHERE ID(n) = vid| GO FROM \<vid> OVER  \<edge_type> |
| 指定点的某一类反向关系 | MATCH (n)<-[r:edge_type]-() WHERE ID(n) = vid| GO FROM \<vid>  OVER \<edge_type> REVERSELY |
| 指定点某一类关系第 N-Hop 查询  |MATCH (n)-[r:edge_type*N]->() <br/> WHERE ID(n) = vid <br/> return r | GO N STEPS FROM \<vid> OVER \<edge_type> |
| 两点路径 | MATCH p =(a)-[]->(b) <br/> WHERE ID(a) = a_vid AND ID(b) = b_vid <br/> RETURN p | FIND ALL PATH FROM \<a_vid> TO \<b_vid> OVER * |

## 示例查询

示例使用以下数据:

![image](https://user-images.githubusercontent.com/42762957/71503167-0e264b80-28af-11ea-87c5-76f4fd1275cd.png)

- 插入数据

```ngql
# 插入点
nebula> INSERT VERTEX character(name, age, type) VALUES hash("saturn"):("saturn", 10000, "titan"), hash("jupiter"):("jupiter", 5000, "god");

# 插入边
nebula> INSERT EDGE father() VALUES hash("jupiter")->hash("saturn"):();

// cypher
cypher> CREATE (src:character {name:"saturn", age: 10000, type:"titan"})
      > CREATE (dst:character {name:"jupiter", age: 5000, type:"god"})
      > CREATE (src)-[rel:father]->(dst)
 ```

- 删除点

```ngql
nebula> DELETE VERTEX hash("prometheus");

cypher> MATCH (n:character {name:"prometheus"})
        > DETACH DELETE n
```

- 更新点的属性

```ngql
nebula> UPDATE VERTEX hash("jesus") SET character.type = 'titan';

cypher> MATCH (n:character {name:"jesus"})
      > SET n.type = 'titan'
```

- 查看点的属性

```ngql
nebula> FETCH PROP ON character hash("saturn");
  ===================================================
  | character.name | character.age | character.type |
  ===================================================
  | saturn         | 10000         | titan          |
  ---------------------------------------------------

cypher> MATCH (n:character {name:"saturn"})
      > RETURN properties(n)
  ╒════════════════════════════════════════════╕
  │"properties(n)"                             │
  ╞════════════════════════════════════════════╡
  │{"name":"saturn","type":"titan","age":10000}│
  └────────────────────────────────────────────┘
```

- 查询 hercules 祖父的姓名

```ngql
nebula> GO 2 STEPS FROM hash("hercules") OVER father YIELD  $$.character.name;
    =====================
    | $$.character.name |
    =====================
    | saturn            |
    ---------------------

cypher> MATCH (src:character{name:"hercules"})-[r:father*2]->(dst:character)
      > RETURN dst.name;
      ╒══════════╕
      │"dst.name"│
      ╞══════════╡
      │"satun"   │
      └──────────┘
```

- 查询 hercules 父亲的姓名

```ngql
nebula> GO FROM hash("hercules") OVER father YIELD $$.character.name;
    =====================
    | $$.character.name |
    =====================
    | jupiter           |
    ---------------------

cypher> MATCH (src:character{name:"hercules"})-[r:father]->(dst:character)
      > RETURN dst.name
      ╒══════════╕
      │"dst.name"│
      ╞══════════╡
      │"jupiter" │
      └──────────┘
```

- 查询百岁老人的姓名

```ngql
nebula> # coming soon

cypher> MATCH (src:character)
      > WHERE src.age > 100
      > RETURN src.name
      ╒═══════════╕
      │"src.name" │
      ╞═══════════╡
      │  "saturn" │
      ├───────────┤
      │ "jupiter" │
      ├───────────┤
      │ "neptune" │
      │───────────│
      │  "pluto"  │
      └───────────┘
```

- 找出 pluto 和谁住

```ngql
nebula> GO FROM hash("pluto") OVER lives YIELD lives._dst AS place | GO FROM $-.place OVER lives REVERSELY WHERE \
      > $$.character.name != "pluto" YIELD $$.character.name AS cohabitants;
    ===============
    | cohabitants |
    ===============
    | cerberus    |
    ---------------

cypher> MATCH (src:character{name:"pluto"})-[r1:lives]->()<-[r2:lives]-(dst:character)
      > RETURN dst.name
      ╒══════════╕
      │"dst.name"│
      ╞══════════╡
      │"cerberus"│
      └──────────┘
```

- 查询 Pluto 的兄弟们以及他们的居住地

```ngql
nebula> GO FROM hash("pluto") OVER brother YIELD brother._dst AS god | \
      > GO FROM $-.god OVER lives YIELD $^.character.name AS Brother, $$.location.name AS Habitations;
    =========================
    | Brother | Habitations |
    =========================
    | jupiter | sky         |
    -------------------------
    | neptune | sea         |
    -------------------------

cypher> MATCH (src:Character{name:"pluto"})-[r1:brother]->(bro:Character)-[r2:lives]->(dst)
      > RETURN bro.name, dst.name
      ╒═════════════════════════╕
      │"bro.name"    │"dst.name"│
      ╞═════════════════════════╡
      │ "jupiter"    │  "sky"   │
      ├─────────────────────────┤
      │ "neptune"    │ "sea"    │
      └─────────────────────────┘
```