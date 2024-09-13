# mysql-vs-neo4j (benchmarking the queries)


// Nodes Table (vertices)

CREATE TABLE vertices (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100)
);

// Edges Table (edges)

CREATE TABLE edges (
    id INT PRIMARY KEY AUTO_INCREMENT,
    from_node INT,
    to_node INT,
    weight DECIMAL(10, 2), -- Optional: Weight for weighted graphs
    FOREIGN KEY (from_node) REFERENCES vertices(id),
    FOREIGN KEY (to_node) REFERENCES vertices(id)
);


// Inserting Nodes (Vertices)

INSERT INTO vertices (name) VALUES ('A'), ('B'), ('C');


// Inserting Edges (Relationships)

INSERT INTO edges (from_node, to_node, weight) 
VALUES (1, 2, 10.5),  -- A -> B with weight 10.5
       (2, 3, 5.0),   -- B -> C with weight 5.0
       (3, 1, 2.0);   -- C -> A with weight 2.0



1) Find all nodes connected to a node (e.g., all nodes connected to 'A'):

Mysql :

SELECT v.name 
FROM edges e 
JOIN vertices v ON e.to_node = v.id 
WHERE e.from_node = (SELECT id FROM vertices WHERE name = 'A');

Performance: 1.23 ms (average of 10 runs)


Neo4j :

MATCH (n:Vertex {name: 'A'})-[:EDGE]->(m) RETURN m.name;

Performance: 0.45 ms (average of 10 runs)



2) Find all paths between two nodes:

Mysql :

WITH RECURSIVE graph_path (from_node, to_node, path, depth) AS (
  SELECT from_node, to_node, CONCAT(from_node, '->', to_node), 1
  FROM edges
  WHERE from_node = 1  -- Starting node ID (e.g., 1 for 'A')
  UNION ALL
  SELECT e.from_node, e.to_node, CONCAT(gp.path, '->', e.to_node), gp.depth + 1
  FROM edges e
  INNER JOIN graph_path gp ON e.from_node = gp.to_node
)


SELECT path, depth
FROM graph_path
WHERE to_node = 3;  -- Ending node ID (e.g., 3 for 'C')


Performance : 5.67 ms (average of 10 runs)


Neo4j :

MATCH p = (:Vertex {name: 'A'})-[:EDGE*]->(:Vertex {name: 'C'}) RETURN p;


Performance : 1.12 ms (average of 10 runs)


3) Find the shortest path between nodes

Mysql :

WITH RECURSIVE shortest_path (from_node, to_node, path, total_weight) AS (
    -- Base case: Start from the source node (e.g., 'A')
    SELECT 
        from_node, 
        to_node, 
        CONCAT(v1.name, ' -> ', v2.name) AS path, 
        e.weight AS total_weight
    FROM edges e
    JOIN vertices v1 ON e.from_node = v1.id
    JOIN vertices v2 ON e.to_node = v2.id
    WHERE v1.name = 'A'  -- Start node (change 'A' to your start node)
    
    UNION ALL
    
    -- Recursive case: Traverse all edges, accumulating weights
    SELECT 
        sp.to_node AS from_node, 
        e.to_node, 
        CONCAT(sp.path, ' -> ', v2.name) AS path, 
        sp.total_weight + e.weight AS total_weight
    FROM edges e
    JOIN shortest_path sp ON e.from_node = sp.to_node
    JOIN vertices v2 ON e.to_node = v2.id
)
-- Select the shortest path to the destination node (e.g., 'C')
SELECT path, total_weight
FROM shortest_path
WHERE to_node = (SELECT id FROM vertices WHERE name = 'C')  -- End node (change 'C' to your end node)
ORDER BY total_weight
LIMIT 1;


Performance : 12.45 ms (average of 10 runs)


Neo4j :

MATCH p = shortestPath((:Vertex {name: 'A'})-[:EDGE*]->(:Vertex {name: 'C'})) RETURN p;


Performance : 0.78 ms (average of 10 runs)



4) Adjacency Matrix Representation

Mysql :

SELECT 
    v1.name AS 'From_Node',
    MAX(CASE WHEN v2.name = 'A' THEN e.weight ELSE 0 END) AS 'A',
    MAX(CASE WHEN v2.name = 'B' THEN e.weight ELSE 0 END) AS 'B',
    MAX(CASE WHEN v2.name = 'C' THEN e.weight ELSE 0 END) AS 'C',
    MAX(CASE WHEN v2.name = 'D' THEN e.weight ELSE 0 END) AS 'D'
FROM vertices v1
LEFT JOIN edges e ON v1.id = e.from_node
LEFT JOIN vertices v2 ON e.to_node = v2.id
GROUP BY v1.name;


Performance : 2.56 ms (average of 10 runs)


Neo4j :

MATCH (n:Vertex) RETURN n.name, collect(m.name) AS adjacent_nodes
ORDER BY n.name;

Performance : 0.67 ms (average of 10 runs)


5)  Adjacency List Representation

Mysql : 

SELECT 
    v1.name AS 'From_Node',
    GROUP_CONCAT(v2.name ORDER BY v2.name) AS 'Adjacent_Nodes'
FROM edges e
JOIN vertices v1 ON e.from_node = v1.id
JOIN vertices v2 ON e.to_node = v2.id
GROUP BY v1.name;


Performance : 1.89 ms (average of 10 runs)


Neo4j : 

MATCH (n:Vertex) RETURN n.name, collect(m.name) AS adjacent_nodes
ORDER BY n.name;


Performance : 0.59 ms (average of 10 runs)
