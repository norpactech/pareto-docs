# 05 – The Role of Automation: Handling the Predictable 80%

While human developers focus on the creative and strategic aspects of software development, automation handles the **predictable majority** of development tasks with speed, consistency, and reliability. The Pareto Factory's automation engine transforms metadata definitions into production-quality code across multiple application layers, eliminating the repetitive work that traditionally consumes the bulk of development time.

This is not simple code generation or templating. It's a **comprehensive development automation system** that understands the relationships between different parts of an application and generates cohesive, integrated code that follows enterprise-grade patterns and practices.

## What Automation Handles

**The Build/Generation Process**

In its current state, a Java backend process uses the Pareto Factory's API to request "Project Components" and download them into specified subdirectories as instructed by the build and configuration in the Project Component. This is the **Direct generated code within an application** methodology. For developers and/or pipelines working on Angular applications, only the Angular components need to be requested. Alternatively, all components can be requested for full-stack developers. The build process is designed to be flexible so all needs can be met. 

In its future state, Pareto Factory will work in conjunction with GitHub's build/pipeline process to create artifacts in various formats like Maven JARs or npm packages so developers can import libraries directly into the application build without directly importing code. This methodology is much simpler from a developer's perspective and is referred to as the **Libraries that developers use within an application** methodology (for lack of a better term).

The build process is flexible enough to support the needs of the project. Setting up build processes for development teams is a hybrid of both automation (where it makes sense) and human activities (where required).

### **Database Layer Generation**
The framework automatically creates complete database schemas from metadata definitions:

**Table Creation**
```sql
CREATE TABLE pareto.plugin (
  id                               UUID             NOT NULL    DEFAULT GEN_RANDOM_UUID(), 
  id_context                       UUID             NOT NULL, 
  name                             VARCHAR(32)      NOT NULL    CHECK (name ~ '^[A-Za-z0-9_][A-Za-z0-9\s\-,\.&''()*_:]{0,30}[A-Za-z0-9_]$'), 
  description                      TEXT             NULL, 
  plugin_service                   TEXT             NOT NULL    CHECK (plugin_service ~ '^[a-z][a-zA-Z0-9_]*$'), 
  created_at                       TIMESTAMP        NOT NULL    DEFAULT CURRENT_TIMESTAMP, 
  created_by                       VARCHAR(32)      NOT NULL, 
  updated_at                       TIMESTAMP        NOT NULL    DEFAULT CURRENT_TIMESTAMP, 
  updated_by                       VARCHAR(32)      NOT NULL, 
  is_active                        BOOLEAN          NOT NULL    DEFAULT TRUE
);

ALTER TABLE pareto.plugin ADD PRIMARY KEY (id);

CREATE UNIQUE INDEX plugin_alt_key
    ON pareto.plugin(id_context, LOWER(name));

ALTER TABLE pareto.plugin
  ADD CONSTRAINT plugin_id_context
  FOREIGN KEY (id_context)
  REFERENCES pareto.context(id)
  ON DELETE CASCADE;

CREATE TRIGGER update_at
  BEFORE UPDATE ON pareto.plugin 
    FOR EACH ROW
      EXECUTE FUNCTION update_at();
```
**Stored Procedures**

This is where Pareto really shines. Notice that the following does much more than simply persist a record. It performs comprehensive validation checks and logs updates. Manually handling these tasks would be daunting when considering that each table can have create, update, delete, and soft-delete stored procedures. 

*Note: Handling persistence using stored procedures is NOT A REQUIREMENT; it was simply a design decision based on the **Data Locality Principle**, which establishes that it's a best practice to process data close to its source. Using stored procedures ensures validations are checked and reported back to the caller.*

```sql
CREATE FUNCTION pareto.i_plugin(
  IN p_id_context UUID, 
  IN p_name VARCHAR, 
  IN p_description TEXT, 
  IN p_plugin_service TEXT, 
  IN p_created_by VARCHAR
)
RETURNS pg_resp
AS $$
DECLARE

  c_service_name TEXT := 'i_plugin';

  v_metadata     JSONB := '{}'::JSONB;
  v_errors       JSONB := '[]'::JSONB;
  v_val_resp     pareto.pg_val;  
  v_response     pareto.pg_resp;
  v_updated_at   TIMESTAMP;
  
  -- Primary Key Field(s)
  v_id uuid := NULL;

BEGIN

  -- ------------------------------------------------------
  -- Metadata
  -- ------------------------------------------------------

  v_metadata := jsonb_build_object(
    'id_context', p_id_context, 
    'name', p_name, 
    'description', p_description, 
    'plugin_service', p_plugin_service, 
    'created_by', p_created_by
  );
  
  -- ------------------------------------------------------
  -- Validations
  -- ------------------------------------------------------
  
  v_val_resp := is_name('name', p_name);
  IF NOT v_val_resp.passed THEN
    v_errors := v_errors || jsonb_build_object('type', 'validation', 'field', v_val_resp.field, 'message', v_val_resp.message);
  END IF;

  v_val_resp := is_service('plugin_service', p_plugin_service);
  IF NOT v_val_resp.passed THEN
    v_errors := v_errors || jsonb_build_object('type', 'validation', 'field', v_val_resp.field, 'message', v_val_resp.message);
  END IF;

  IF jsonb_array_length(v_errors) > 0 THEN
    v_response := (
      'ERROR', 
      NULL, 
      v_errors, 
      '23514', 
      'A CHECK constraint was violated due to incorrect input', 
      'Ensure all fields in the ''errors'' array are correctly formatted', 
      'The provided data did not pass validation checks'
    );
    CALL pareto.i_logs(v_response.status, v_response.message, c_service_name, p_created_by, v_metadata);
    RETURN v_response;
  END IF;
  
  -- ------------------------------------------------------
  -- Persist
  -- ------------------------------------------------------
 
  INSERT INTO pareto.plugin (
    id_context, 
    name, 
    description, 
    plugin_service, 
    created_by,
    updated_by
  )
  VALUES (
    p_id_context, 
    p_name, 
    p_description, 
    p_plugin_service, 
    p_created_by,
    p_created_by
  )
  RETURNING id, updated_at INTO v_id, v_updated_at;

  v_response := (
    'OK',
    jsonb_build_object('id', v_id, 'updated_at', v_updated_at), 
    NULL, 
    '00000',
    'Insert was successful', 
    NULL, 
    NULL
  );
  RETURN v_response;

  -- ------------------------------------------------------
  -- Exceptions
  -- ------------------------------------------------------
  
  EXCEPTION
    WHEN UNIQUE_VIOLATION THEN
      v_response := (
        'ERROR', 
        NULL, 
        jsonb_build_object('type', 'database', 'message', 'A UNIQUE constraint was violated due to duplicate data'), 
        '23514', 
        'A UNIQUE constraint was violated due to duplicate data', 
        'A record already exists in the plugin table', 
        'Check the provided data and try again'
      );
      CALL pareto.i_logs(v_response.status, v_response.message, c_service_name, p_created_by, v_metadata);
      RETURN v_response;
  
    WHEN OTHERS THEN
      v_response := (
        'ERROR', 
        NULL, 
        jsonb_build_object('type', 'database', 'message', SQLERRM), 
        SQLSTATE, 
        'An unexpected error occurred', 
        'Check database logs for more details', 
        SQLERRM
      );
      CALL pareto.i_logs(v_response.status, v_response.message, c_service_name, p_created_by, v_metadata);
      RETURN v_response;
  
END;
$$ LANGUAGE plpgsql;
```

### **Current Generation**

There are additional layers to modern applications including the user interface and middle-tier API. Providing all the code for all layers would simply be too overwhelming for this whitepaper. In summary, the following application components are generated **as of today**:

**PostgreSQL**
- Create Tables DDL
- Create Validation Stored Procedures
- Insert Functions
- Update Functions
- Delete Functions
- Active/Inactive Functions

**MS SQL Server Scripting**
- Create Tables
- Create Validations
- Create Insert Functions
- Create Update Functions
- Create Delete Functions
- Create Active/Inactive (soft-delete) Functions

**Java/Spring Boot**
- Java Model Classes
- Controller Classes
- DTO (Data Transfer Objects) Classes
- Post API Requests
- Put API Requests
- Delete API
- Active/Inactive (soft-delete) API

**TypeScript**
- Model Classes
- Service Classes
- DTO Classes

**What's missing?** 

Angular forms and table search pages are not being generated as of this writing. This is mainly due to the fact that GitHub Copilot is capable of looking at the model and service classes and performing the generation using that medium. In the future, the patterns for tables and forms will be further reviewed to determine if developing a plugin for generation is feasible.

Security is partially managed in the Pareto Factory. Generated code ensures that items like JWTs (JSON Web Tokens) are propagated throughout the API, for example. Items like Route Guards and User Roles in the Angular applications are managed by humans. As in Compiere, however, the Pareto Factory can extend the metadata to include functions/APIs and user roles. Other functions/capabilities can also be included.  

**Cross-Platform Development**

The Pareto Factory maintains a plugin for MS DOS scripts that generate the database. The scripts are designed to stop at the point of any error so the error can be identified and corrected accordingly. Many developers (including myself) also use Linux-based systems. In order to support this, the only task needed is to create a set of generated bash (or other) shell scripts to perform this task elsewhere.

### **Version Control Integration**
Generated code is designed to work seamlessly with modern development workflows:

- **Merge-friendly generation** that minimizes conflicts
- **Incremental updates** that preserve custom modifications
- **Clear separation** between generated and custom code
- **Automated testing** of generated components

The file strategy is simple replacement. When generation is performed, ALL files are overwritten with either their previous version or new version. Files are removed when no longer generated. There are no updates in the sense that files are changed/versioned. That task is relegated to the git repository. If any merges need to be evaluated manually, GitHub (or your repository of choice) will present the manual merge in Pull Requests, and a human is required to make that change. 

## Quality Assurance Through Automation

### **Consistency Enforcement**
Every generated component follows identical patterns:

- **Naming conventions** are applied uniformly across all layers
- **Error handling** follows the same patterns everywhere
- **Security measures** are built into every endpoint
- **Logging and monitoring** are consistently implemented

### **Built-in Best Practices**
Generated code incorporates enterprise-grade practices:

- **Input validation** at multiple layers
- **SQL injection prevention** through parameterized queries
- **Audit trails** for all data modifications
- **Performance optimization** through proper indexing and caching

### **Comprehensive Testing**
A future enhancement is to generate test suites alongside functional code. Theoretically, the metadata should be capable of evaluating itself and generating useful integration and end-to-end tests that ensure the system is working as intended. The intent is to ensure that every stored procedure and API call is performed as intended with all validations in place. Looking further, the User Interface may also be included in tests. For now, however, this is a human-performed activity.

## Integration with Development Workflow

### **CI/CD Pipeline Integration**
Generated code integrates smoothly with modern deployment practices:

- **Build scripts** that compile and package generated components
- **Database migration scripts** for schema changes

The framework performs database script generation. It's anticipated that the pipelines will be human-developed for all CI/CD activities. 

### **Summary**

By handling the predictable 80% of development tasks, automation frees human developers to focus on the creative and strategic work that truly adds value. The result is software that is both more innovative and more reliable—combining the best of human creativity with the consistency and speed of automation.

*The next section examines a real-world case study demonstrating how this human-automation partnership works in practice.*
