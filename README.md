# i2i-Academy-RDBMSwithOracle-1

This project is a Spring Boot REST API connected to an Oracle database running on Docker. I used Flyway for automatic table creations and PL/SQL packages to handle XML and JSON data parsing inside the database. I built this project to practice relational database management and backend development.

## Local Environment Setup & Deployment:
  In this part I configured Spring Boot and Oracle to connect them to the same network (i2i-network). 
This way they can work and communicate with each other. I created a Dbeaver file and connected with mt Java project. 

    version: '3.8'
    
    services:
      oracle-db:
        image: gvenzl/oracle-xe:latest
        container_name: oracle-xe-container
        environment:
          - ORACLE_PASSWORD=12345
        ports:
          - "1521:1521"
        networks:
          - i2i-network
    
      spring-app:
        build: .
        container_name: spring-boot-app
        ports:
          - "8080:8080"
        depends_on:
          - oracle-db
        networks:
          - i2i-network 
          networks:
        i2i-network:
          driver: bridge



Then I built FlyWay structure. Firstly I downloaded libraries from my .xml file. 

         <dependency>
              <groupId>org.flywaydb</groupId>
              <artifactId>flyway-core</artifactId>
          </dependency>
          <dependency>
              <groupId>org.flywaydb</groupId>
              <artifactId>flyway-database-oracle</artifactId>
          </dependency>



After that I wrote the FlyWay codes. When I run the code FlyWay will come this file and create tables 
and because of we connected Dbeaver we are going to see these tables on the Dbeaver if our code 
run correctly. Also at the end of the code there is  Audit_Log structure. When we add a book FlyWay can add the 
information into the tables with this.

    -- We created this table to store author datas
    CREATE TABLE AUTHORS (
                             id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                             name VARCHAR2(100) NOT NULL
    );
    
    -- This table is for publishers
    CREATE TABLE PUBLISHERS (
                                id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                                name VARCHAR2(100) NOT NULL
    );
    
    -- Main books table. It has foreign keys to connect authors and publishers
    CREATE TABLE BOOKS (
                           id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                           title VARCHAR2(200) NOT NULL,
                           author_id NUMBER,
                           publisher_id NUMBER,
                           CONSTRAINT fk_author FOREIGN KEY (author_id) REFERENCES AUTHORS(id),
                           CONSTRAINT fk_publisher FOREIGN KEY (publisher_id) REFERENCES PUBLISHERS(id)
    );
    
    -- We built this table to log insert operations
    CREATE TABLE AUDIT_LOGS (
                                id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                                log_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                                db_user VARCHAR2(50),
                                action_desc VARCHAR2(200)
    );
    
    -- This trigger automatically works when a new book is added
    CREATE OR REPLACE TRIGGER trg_books_audit
    AFTER INSERT ON BOOKS
    FOR EACH ROW
    BEGIN
    INSERT INTO AUDIT_LOGS (db_user, action_desc)
    VALUES (USER, 'New book added: ' || :NEW.title);
    END;
    /



## PL/SQL Implementation: 
In this section, I created a single PL/SQL package named BOOK_OPERATIONS via Flyway. This package 
handles all the database logic. It includes functions to parse raw text strings into XML and JSON 
formats. It also contains an insert_books procedure to safely save this data into the related tables 
and a get_all_books procedure with a cursor to fetch all book records. 

    -- We create package header first. This is like an interface.
    CREATE OR REPLACE PACKAGE BOOK_OPERATIONS AS
        FUNCTION parse_to_xml(p_raw_data IN VARCHAR2) RETURN CLOB;
        FUNCTION parse_to_json(p_raw_data IN VARCHAR2) RETURN CLOB;
        PROCEDURE insert_books(p_xml_data IN CLOB, p_json_data IN CLOB);
        PROCEDURE get_all_books(p_recordset OUT SYS_REFCURSOR);
    END BOOK_OPERATIONS;
    /
    
    -- This is the package body. Real operations are here.
    CREATE OR REPLACE PACKAGE BODY BOOK_OPERATIONS AS
    
        -- 1. Function to make XML from raw string
        FUNCTION parse_to_xml(p_raw_data IN VARCHAR2) RETURN CLOB IS
            v_xml CLOB;
    BEGIN
            -- We get data like "AuthorName|PublisherName|BookTitle"
            -- We split it and build XML string
            v_xml := '<books><book><author>' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 1) || '</author>' ||
                     '<publisher>' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 2) || '</publisher>' ||
                     '<title>' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 3) || '</title></book></books>';
    RETURN v_xml;
    END parse_to_xml;
    
        -- 2. Function to make JSON from raw string
        FUNCTION parse_to_json(p_raw_data IN VARCHAR2) RETURN CLOB IS
            v_json CLOB;
    BEGIN
            -- We build JSON format from the same string
            v_json := '{"books": [{"author": "' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 1) || '", ' ||
                      '"publisher": "' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 2) || '", ' ||
                      '"title": "' || REGEXP_SUBSTR(p_raw_data, '[^|]+', 1, 3) || '"}]}';
    RETURN v_json;
    END parse_to_json;
    
        -- 3. Procedure to insert parsed data into tables
        PROCEDURE insert_books(p_xml_data IN CLOB, p_json_data IN CLOB) IS
            v_author_id NUMBER;
            v_publisher_id NUMBER;
            v_title VARCHAR2(200);
            v_author_name VARCHAR2(100);
            v_pub_name VARCHAR2(100);
    BEGIN
            -- Read values from XML data easily
    SELECT author, publisher, title
    INTO v_author_name, v_pub_name, v_title
    FROM XMLTABLE('/books/book'
                      PASSING XMLTYPE(p_xml_data)
                 COLUMNS author VARCHAR2(100) PATH 'author',
                  publisher VARCHAR2(100) PATH 'publisher',
                  title VARCHAR2(200) PATH 'title');
    
    -- Insert author if not exists
    BEGIN
    SELECT id INTO v_author_id FROM AUTHORS WHERE name = v_author_name;
    EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    INSERT INTO AUTHORS (name) VALUES (v_author_name) RETURNING id INTO v_author_id;
    END;
    
            -- Insert publisher if not exists
    BEGIN
    SELECT id INTO v_publisher_id FROM PUBLISHERS WHERE name = v_pub_name;
    EXCEPTION
                WHEN NO_DATA_FOUND THEN
                    INSERT INTO PUBLISHERS (name) VALUES (v_pub_name) RETURNING id INTO v_publisher_id;
    END;
    
            -- Finally insert the book
    INSERT INTO BOOKS (title, author_id, publisher_id)
    VALUES (v_title, v_author_id, v_publisher_id);
    
    COMMIT;
    EXCEPTION
            WHEN OTHERS THEN
                -- If something goes wrong, rollback everything
                ROLLBACK;
                RAISE_APPLICATION_ERROR(-20001, 'Error while inserting book data: ' || SQLERRM);
    END insert_books;
    
        -- 4. Procedure to send all books to Java
        PROCEDURE get_all_books(p_recordset OUT SYS_REFCURSOR) IS
    BEGIN
            -- Open a cursor to read all joined tables
    OPEN p_recordset FOR
    SELECT b.id, b.title, a.name as author_name, p.name as publisher_name
    FROM BOOKS b
             LEFT JOIN AUTHORS a ON b.author_id = a.id
             LEFT JOIN PUBLISHERS p ON b.publisher_id = p.id;
    END get_all_books;
    
    END BOOK_OPERATIONS;
    /



## Java Implementation 
For the Java implementation, I built a Spring Boot REST API. I organized my code into three main 
layers: 

1- I created BookDto.java to hold and transfer book data easily using Lombok.

    package org.example.dto;
    
    import lombok.Data;
    
    @Data
    public class BookDto {
        private Long id;
        private String title;
        private String authorName;
        private String publisherName;
    }



2- I created BookRepository.java to connect with the database.  

    package org.example.repository;

    import org.example.dto.BookDto;
    import org.springframework.jdbc.core.JdbcTemplate;
    import org.springframework.jdbc.core.simple.SimpleJdbcCall;
    import org.springframework.stereotype.Repository;
    
    import java.util.List;
    import java.util.Map;
    
    @Repository
    public class BookRepository {
    
        private final JdbcTemplate jdbcTemplate;
    
        // Constructor injection for database connection tool
        public BookRepository(JdbcTemplate jdbcTemplate) {
            this.jdbcTemplate = jdbcTemplate;
        }
    
        // Method to get XML format from DB function
        public String generateXml(String rawData) {
            String sql = "SELECT BOOK_OPERATIONS.parse_to_xml(?) FROM DUAL";
            return jdbcTemplate.queryForObject(sql, String.class, rawData);
        }
    
        // Method to get JSON format from DB function
        public String generateJson(String rawData) {
            String sql = "SELECT BOOK_OPERATIONS.parse_to_json(?) FROM DUAL";
            return jdbcTemplate.queryForObject(sql, String.class, rawData);
        }
    
        // Method to call our PL/SQL insert procedure
        public void callInsertBooksProcedure(String xmlData, String jsonData) {
            // We directly call the procedure inside our package
            String sql = "CALL BOOK_OPERATIONS.insert_books(?, ?)";
            jdbcTemplate.update(sql, xmlData, jsonData);
        }
    
        // Method to fetch all books using PL/SQL cursor
        @SuppressWarnings("unchecked")
        public List<BookDto> callGetAllBooksProcedure() {
            // SimpleJdbcCall helps us to read OUT parameters (like cursors) easily
            SimpleJdbcCall jdbcCall = new SimpleJdbcCall(jdbcTemplate)
                    .withCatalogName("BOOK_OPERATIONS") // Name of our package
                    .withProcedureName("get_all_books") // Name of our procedure
                    .returningResultSet("p_recordset", (rs, rowNum) -> {
                        // We map database columns to our Java DTO object
                        BookDto dto = new BookDto();
                        dto.setId(rs.getLong("id"));
                        dto.setTitle(rs.getString("title"));
                        dto.setAuthorName(rs.getString("author_name"));
                        dto.setPublisherName(rs.getString("publisher_name"));
                        return dto;
                    });
    
            // Execute the call and get the results
            Map<String, Object> result = jdbcCall.execute();
    
            // Return the mapped list
            return (List<BookDto>) result.get("p_recordset");
        }
    }



3- I created BookController.java to open REST endpoints.

    package org.example.controller;
    
    import org.example.dto.BookDto;
    import org.example.repository.BookRepository;
    import org.springframework.http.HttpStatus;
    import org.springframework.http.ResponseEntity;
    import org.springframework.web.bind.annotation.*;
    
    import java.util.List;
    
    @RestController
    @RequestMapping("/api/books")
    public class BookController {
    
        private final BookRepository bookRepository;
    
        public BookController(BookRepository bookRepository) {
            this.bookRepository = bookRepository;
        }
    
        // 1. POST Endpoint to import raw data
        @PostMapping("/import")
        public ResponseEntity<?> importBooks(@RequestBody String rawData) {
            try {
                // Generate XML and JSON using Oracle functions
                String xmlData = bookRepository.generateXml(rawData);
                String jsonData = bookRepository.generateJson(rawData);
    
                // Insert generated data into tables
                bookRepository.callInsertBooksProcedure(xmlData, jsonData);
    
                return ResponseEntity.ok("Book imported successfully!");
            } catch (Exception e) {
                // If anything goes wrong, return 400 Bad Request
                return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                        .body("Error while importing book: " + e.getMessage());
            }
        }
    
        // 2. GET Endpoint to fetch all books
        @GetMapping
        public ResponseEntity<List<BookDto>> getAllBooks() {
            try {
                List<BookDto> books = bookRepository.callGetAllBooksProcedure();
                return ResponseEntity.ok(books);
            } catch (Exception e) {
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(null);
            }
        }
    }


## Test Section:
Finally I successfully tested all these endpoints using the api-test.http file. 

    ### 1. Post the books to database
    POST http://localhost:8080/api/books/import
    Content-Type: text/plain
    
    Author1|Book_Name1
    Author2|Book_Name2
    Author3|Book_Name3
    
    
    ### 2. Get every books
    GET http://localhost:8080/api/books
