# 05 – The Role of Automation: Handling the Predictable 80%

While human developers would focus on the creative and strategic aspects of software development, automation would handle the **predictable majority** of development tasks with speed, consistency, and reliability. The Pareto Factory's automation engine would transform metadata definitions into production-quality code across multiple application layers, eliminating the repetitive work that traditionally consumes the bulk of development time.

This would not be simple code generation or templating. It would be a **comprehensive development automation system** that understands the relationships between different parts of an application and generates cohesive, integrated code that follows enterprise-grade patterns and practices.

## What Automation Would Handle

**The Build/Generation Process**

In its envisioned initial state, a Java backend process would use the Pareto Factory's API to request "Project Components" and download them into specified subdirectories as instructed by the build and configuration in the Project Component. This would be the **Direct generated code within an application** methodology. For developers and/or pipelines working on Angular applications, only the Angular components would need to be requested. Alternatively, all components could be requested for full-stack developers. The build process would be designed to be flexible so all needs could be met. 

In its future state, Pareto Factory would work in conjunction with GitHub's build/pipeline process to create artifacts in various formats like Maven JARs or npm packages so developers could import libraries directly into the application build without directly importing code. This methodology would be much simpler from a developer's perspective and is referred to as the **Libraries that developers use within an application** methodology (for lack of a better term).

The build process would be flexible enough to support the needs of the project. Setting up build processes for development teams would be a hybrid of both automation (where it makes sense) and human activities (where required).

### **Database Layer Generation**
The framework would automatically create complete database schemas from metadata definitions:

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

This is where Pareto would really shine. Notice that the following would do much more than simply persist a record. It would perform comprehensive validation checks and log updates. Manually handling these tasks would be daunting when considering that each table could have create, update, delete, and soft-delete stored procedures. 

*Note: Handling persistence using stored procedures would NOT BE A REQUIREMENT; it would simply be a design decision based on the **Data Locality Principle**, which establishes that it's a best practice to process data close to its source. Using stored procedures would ensure validations are checked and reported back to the caller.*

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

### **Envisioned Generation**

There would be additional layers to modern applications including the user interface and middle-tier API. Providing all the code for all layers would simply be too overwhelming for this whitepaper. In summary, the following application components would be generated **in the envisioned implementation**:

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

**What Would Be Missing?** 

Angular forms and table search pages would not be generated in the initial implementation. This would be mainly due to the fact that GitHub Copilot is capable of looking at the model and service classes and performing the generation using that medium. In the future, the patterns for tables and forms would be further reviewed to determine if developing a plugin for generation is feasible.

Security would be partially managed in the Pareto Factory. Generated code would ensure that items like JWTs (JSON Web Tokens) are propagated throughout the API, for example. Items like Route Guards and User Roles in the Angular applications would be managed by humans. As in Compiere, however, the Pareto Factory could extend the metadata to include functions/APIs and user roles. Other functions/capabilities could also be included.  

**Cross-Platform Development**

The Pareto Factory would maintain a plugin for MS DOS scripts that generate the database. The scripts would be designed to stop at the point of any error so the error can be identified and corrected accordingly. Many developers (including myself) also use Linux-based systems. In order to support this, the only task needed would be to create a set of generated bash (or other) shell scripts to perform this task elsewhere.

### **Version Control Integration**
Generated code would be designed to work seamlessly with modern development workflows:

- **Merge-friendly generation** that minimizes conflicts
- **Incremental updates** that preserve custom modifications
- **Clear separation** between generated and custom code
- **Automated testing** of generated components

The file strategy would be simple replacement. When generation is performed, ALL files would be overwritten with either their previous version or new version. Files would be removed when no longer generated. There would be no updates in the sense that files are changed/versioned. That task would be relegated to the git repository. If any merges need to be evaluated manually, GitHub (or your repository of choice) would present the manual merge in Pull Requests, and a human would be required to make that change. 

## Quality Assurance Through Automation

### **Consistency Enforcement**
Every generated component would follow identical patterns:

- **Naming conventions** would be applied uniformly across all layers
- **Error handling** would follow the same patterns everywhere
- **Security measures** would be built into every endpoint
- **Logging and monitoring** would be consistently implemented

### **Built-in Best Practices**
Generated code would incorporate enterprise-grade practices:

- **Input validation** at multiple layers
- **SQL injection prevention** through parameterized queries
- **Audit trails** for all data modifications
- **Performance optimization** through proper indexing and caching

### **Comprehensive Testing**
A future enhancement would be to generate test suites alongside functional code. Theoretically, the metadata should be capable of evaluating itself and generating useful integration and end-to-end tests that ensure the system is working as intended. The intent would be to ensure that every stored procedure and API call is performed as intended with all validations in place. Looking further, the User Interface might also be included in tests. For now, however, this would be a human-performed activity.

## Integration with Development Workflow

### **CI/CD Pipeline Integration**
Generated code would integrate smoothly with modern deployment practices:

- **Build scripts** that compile and package generated components
- **Database migration scripts** for schema changes

The framework would perform database script generation. It's anticipated that the pipelines would be human-developed for all CI/CD activities. 

### **Summary**

By handling the predictable 80% of development tasks, automation would free human developers to focus on the creative and strategic work that truly adds value. The result would be software that is both more innovative and more reliable—combining the best of human creativity with the consistency and speed of automation.

*The next section examines a real-world case study demonstrating how this human-automation partnership would work in practice.*
