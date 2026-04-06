# Relational-to-Graph Schema Mapper

This repository provides a framework for utilizing LLMs (Gemini) within BigQuery to analyze relational database schemas and automatically generate a **Knowledge Graph (KG) mapping specification**.

Instead of extracting raw strings from text, this approach performs **Semantic Schema Modeling**—interpreting DDLs and table metadata to determine which components of your database represent "Entities" (Nodes) and which represent "Relationships" (Edges).

## 💡 The Logic Workflow

Transitioning from structured tables to a graph requires a shift in how data is perceived:
1.  **Input:** Table definitions (DDL) or `INFORMATION_SCHEMA` metadata.
2.  **Inference:** Gemini identifies the semantic role of each table and column.
3.  **Output:** A structured JSON/Table spec that dictates how to transform relational rows into Graph nodes and edges.



## 🛠️ Implementation

### 1. The Mapping Query
Use the following pattern to generate your graph specification. This query feeds your table schema into the model and receives a structured mapping array.

```sql
CREATE OR REPLACE TABLE `${PROJECT_ID}.${DATASET_ID}.GRAPH_MAPPING_SPEC` AS
SELECT 
  table_name,
  AI.GENERATE(
    -- The prompt instructs the AI to act as a Graph Architect
    """
    You are a technical Graph Architect. 
    Analyze the following SQL Table Schema and determine:
    1. Is this table a 'NODE' (an entity), an 'EDGE' (a relationship), or 'BOTH'?
    2. Which columns should be the 'Source' and 'Target' for edges?
    3. What are the Entity Types (e.g., Product, User, Location)?

    Schema to analyze:
    """ || ddl_text,
    
    -- ARGUMENT 2: The Output Schema (Structured for downstream automation)
    output_schema => """
      mappings ARRAY<STRUCT<
        source_table STRING,
        mapping_type STRING, -- 'NODE' or 'EDGE'
        node_label STRING,   -- e.g., 'Part'
        edge_label STRING,   -- e.g., 'CONSISTS_OF'
        source_id_col STRING,
        target_id_col STRING,
        properties ARRAY<STRING>
      >>
    """,
    endpoint => 'gemini-1.5-pro'
  ) AS mapping_logic
FROM `${PROJECT_ID}.${DATASET_ID}.SCHEMA_METADATA_TABLE`;
```

### 2. Identifying Nodes vs. Edges
The model is instructed to follow these core heuristics:

| Target | Relational Logic | Graph Representation |
| :--- | :--- | :--- |
| **Nodes** | Tables with a unique PK (e.g., `Users`, `Inventory`) | Unique vertices with property keys. |
| **Edges (Direct)** | Foreign Keys (e.g., `category_id` inside a `Product` table) | A directed link: `(Product)-[:HAS_CATEGORY]->(Category)`. |
| **Edges (Junction)** | Many-to-Many tables (e.g., `Part_Assembly`) | A relationship connecting two distinct nodes. |



## ⚠️ Best Practices
* **Domain Isolation:** For large enterprise schemas, do not pass every table at once. Group DDLs by "Subject Area" (e.g., Supply Chain vs. Sales) to ensure the LLM maintains focus and accuracy.
* **Property Selection:** Encourage the model to only map high-value attributes as node properties to keep the graph performant.
* **Recursive Links:** Pay special attention to tables that reference themselves (e.g., an `Employees` table with a `manager_id`), as these create critical hierarchical edges.

