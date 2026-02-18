# Factory Method Pattern - Complete Guide

## Introduction

The **Factory Method Pattern** is a creational design pattern that provides an interface for creating objects, but allows subclasses to alter the type of objects that will be created. It's also known as the "Virtual Constructor" pattern.

## Definition

> **Factory Method Pattern**: Defines an interface for creating an object, but lets subclasses decide which class to instantiate.

## Problem Solved

- **Object Creation Complexity**: Hide complex object creation logic from clients
- **Dependency Management**: Reduce coupling between client code and concrete classes
- **Extensibility**: Easy to add new product types without modifying existing code
- **Code Reusability**: Centralize object creation logic
- **Runtime Decision**: Choose object type at runtime based on conditions

## Structure

```
┌─────────────────┐    ┌─────────────────┐
│    Creator      │───▶│    Product      │
├─────────────────┤    ├─────────────────┤
│ +factoryMethod()│    │ +operation()    │
│ +someOperation()│    └─────────────────┘
└─────────────────┘              △
          △                      │
          │                      │
┌─────────────────┐    ┌─────────────────┐
│ ConcreteCreator │    │ ConcreteProduct │
├─────────────────┤    ├─────────────────┤
│ +factoryMethod()│───▶│ +operation()    │
└─────────────────┘    └─────────────────┘
```

## Basic Implementation

### Product Interface and Concrete Products

```java
/**
 * Abstract Product - Document interface
 */
public interface Document {
    void open();
    void save();
    void close();
    void print();
    String getType();
}

/**
 * Concrete Product - Word Document
 */
public class WordDocument implements Document {
    
    private String content = "";
    private boolean isOpen = false;
    
    @Override
    public void open() {
        if (!isOpen) {
            System.out.println("Opening Word document...");
            isOpen = true;
        } else {
            System.out.println("Word document is already open");
        }
    }
    
    @Override
    public void save() {
        if (isOpen) {
            System.out.println("Saving Word document with rich formatting...");
        } else {
            throw new RuntimeException("Cannot save closed document");
        }
    }
    
    @Override
    public void close() {
        if (isOpen) {
            System.out.println("Closing Word document");
            isOpen = false;
        }
    }
    
    @Override
    public void print() {
        if (isOpen) {
            System.out.println("Printing Word document with page layout...");
        } else {
            throw new RuntimeException("Cannot print closed document");
        }
    }
    
    @Override
    public String getType() {
        return "Microsoft Word Document";
    }
    
    public void addFormatting(String formatting) {
        if (isOpen) {
            System.out.println("Adding formatting to Word document: " + formatting);
        }
    }
}

/**
 * Concrete Product - PDF Document
 */
public class PDFDocument implements Document {
    
    private boolean isOpen = false;
    private boolean isReadOnly = true;
    
    @Override
    public void open() {
        if (!isOpen) {
            System.out.println("Opening PDF document...");
            isOpen = true;
        } else {
            System.out.println("PDF document is already open");
        }
    }
    
    @Override
    public void save() {
        if (isOpen) {
            if (isReadOnly) {
                throw new RuntimeException("Cannot save read-only PDF document");
            }
            System.out.println("Saving PDF document...");
        } else {
            throw new RuntimeException("Cannot save closed document");
        }
    }
    
    @Override
    public void close() {
        if (isOpen) {
            System.out.println("Closing PDF document");
            isOpen = false;
        }
    }
    
    @Override
    public void print() {
        if (isOpen) {
            System.out.println("Printing PDF document with high fidelity...");
        } else {
            throw new RuntimeException("Cannot print closed document");
        }
    }
    
    @Override
    public String getType() {
        return "Portable Document Format";
    }
    
    public void setReadOnly(boolean readOnly) {
        this.isReadOnly = readOnly;
        System.out.println("PDF document read-only status: " + readOnly);
    }
}

/**
 * Concrete Product - Text Document
 */
public class TextDocument implements Document {
    
    private StringBuilder content = new StringBuilder();
    private boolean isOpen = false;
    
    @Override
    public void open() {
        if (!isOpen) {
            System.out.println("Opening text document...");
            isOpen = true;
        } else {
            System.out.println("Text document is already open");
        }
    }
    
    @Override
    public void save() {
        if (isOpen) {
            System.out.println("Saving text document as plain text...");
        } else {
            throw new RuntimeException("Cannot save closed document");
        }
    }
    
    @Override
    public void close() {
        if (isOpen) {
            System.out.println("Closing text document");
            isOpen = false;
        }
    }
    
    @Override
    public void print() {
        if (isOpen) {
            System.out.println("Printing text document (plain text)...");
        } else {
            throw new RuntimeException("Cannot print closed document");
        }
    }
    
    @Override
    public String getType() {
        return "Plain Text Document";
    }
    
    public void appendText(String text) {
        if (isOpen) {
            content.append(text);
            System.out.println("Appended text: " + text);
        }
    }
    
    public String getContent() {
        return content.toString();
    }
}
```

### Creator and Concrete Creators

```java
/**
 * Abstract Creator - Document Application
 */
public abstract class DocumentApplication {
    
    // Factory method - to be implemented by subclasses
    public abstract Document createDocument();
    
    // Template method that uses the factory method
    public void newDocument() {
        Document doc = createDocument();
        doc.open();
        System.out.println("Created new document: " + doc.getType());
    }
    
    public void openAndProcessDocument() {
        Document doc = createDocument();
        
        try {
            doc.open();
            
            // Common processing logic
            processDocument(doc);
            
            doc.save();
            doc.close();
            
        } catch (Exception e) {
            System.err.println("Error processing document: " + e.getMessage());
        }
    }
    
    protected void processDocument(Document doc) {
        System.out.println("Processing document of type: " + doc.getType());
        // Common processing logic can go here
    }
}

/**
 * Concrete Creator - Word Application
 */
public class WordApplication extends DocumentApplication {
    
    @Override
    public Document createDocument() {
        System.out.println("Creating Word document using Word factory method");
        return new WordDocument();
    }
    
    @Override
    protected void processDocument(Document doc) {
        super.processDocument(doc);
        
        if (doc instanceof WordDocument) {
            WordDocument wordDoc = (WordDocument) doc;
            wordDoc.addFormatting("Bold headers and bullet points");
        }
    }
}

/**
 * Concrete Creator - PDF Application
 */
public class PDFApplication extends DocumentApplication {
    
    @Override
    public Document createDocument() {
        System.out.println("Creating PDF document using PDF factory method");
        return new PDFDocument();
    }
    
    @Override
    protected void processDocument(Document doc) {
        super.processDocument(doc);
        
        if (doc instanceof PDFDocument) {
            PDFDocument pdfDoc = (PDFDocument) doc;
            pdfDoc.setReadOnly(false); // Allow editing for processing
        }
    }
}

/**
 * Concrete Creator - Text Application
 */
public class TextApplication extends DocumentApplication {
    
    @Override
    public Document createDocument() {
        System.out.println("Creating text document using Text factory method");
        return new TextDocument();
    }
    
    @Override
    protected void processDocument(Document doc) {
        super.processDocument(doc);
        
        if (doc instanceof TextDocument) {
            TextDocument textDoc = (TextDocument) doc;
            textDoc.appendText("Sample content for text document");
        }
    }
}
```

## Parameterized Factory Method

### Enhanced Factory with Parameters

```java
/**
 * Enhanced creator with parameterized factory method
 */
public class EnhancedDocumentApplication extends DocumentApplication {
    
    public enum DocumentType {
        WORD, PDF, TEXT, POWERPOINT, EXCEL
    }
    
    // Parameterized factory method
    public Document createDocument(DocumentType type) {
        switch (type) {
            case WORD:
                return new WordDocument();
            case PDF:
                return new PDFDocument();
            case TEXT:
                return new TextDocument();
            case POWERPOINT:
                return new PowerPointDocument();
            case EXCEL:
                return new ExcelDocument();
            default:
                throw new IllegalArgumentException("Unsupported document type: " + type);
        }
    }
    
    // Default implementation for abstract method
    @Override
    public Document createDocument() {
        return createDocument(DocumentType.TEXT); // Default to text
    }
    
    // Enhanced method with type-specific processing
    public void processDocumentByType(DocumentType type) {
        Document doc = createDocument(type);
        
        try {
            doc.open();
            
            // Type-specific processing
            switch (type) {
                case WORD:
                    processWordDocument((WordDocument) doc);
                    break;
                case PDF:
                    processPDFDocument((PDFDocument) doc);
                    break;
                case TEXT:
                    processTextDocument((TextDocument) doc);
                    break;
                case POWERPOINT:
                    processPowerPointDocument((PowerPointDocument) doc);
                    break;
                case EXCEL:
                    processExcelDocument((ExcelDocument) doc);
                    break;
            }
            
            doc.save();
            doc.close();
            
        } catch (Exception e) {
            System.err.println("Error processing " + type + " document: " + e.getMessage());
        }
    }
    
    private void processWordDocument(WordDocument doc) {
        doc.addFormatting("Professional formatting applied");
    }
    
    private void processPDFDocument(PDFDocument doc) {
        doc.setReadOnly(false);
        System.out.println("PDF optimized for printing");
    }
    
    private void processTextDocument(TextDocument doc) {
        doc.appendText("Standard text content added");
    }
    
    private void processPowerPointDocument(PowerPointDocument doc) {
        doc.addSlide("Title Slide");
        doc.addSlide("Content Slide");
    }
    
    private void processExcelDocument(ExcelDocument doc) {
        doc.addWorksheet("Data");
        doc.addFormula("A1", "=SUM(B1:B10)");
    }
}

// Additional document types
class PowerPointDocument implements Document {
    private List<String> slides = new ArrayList<>();
    private boolean isOpen = false;
    
    @Override
    public void open() {
        isOpen = true;
        System.out.println("Opening PowerPoint presentation...");
    }
    
    @Override
    public void save() {
        if (isOpen) {
            System.out.println("Saving PowerPoint presentation...");
        }
    }
    
    @Override
    public void close() {
        isOpen = false;
        System.out.println("Closing PowerPoint presentation");
    }
    
    @Override
    public void print() {
        if (isOpen) {
            System.out.println("Printing PowerPoint slides...");
        }
    }
    
    @Override
    public String getType() {
        return "Microsoft PowerPoint Presentation";
    }
    
    public void addSlide(String title) {
        if (isOpen) {
            slides.add(title);
            System.out.println("Added slide: " + title);
        }
    }
}

class ExcelDocument implements Document {
    private Map<String, String> worksheets = new HashMap<>();
    private Map<String, String> formulas = new HashMap<>();
    private boolean isOpen = false;
    
    @Override
    public void open() {
        isOpen = true;
        System.out.println("Opening Excel spreadsheet...");
    }
    
    @Override
    public void save() {
        if (isOpen) {
            System.out.println("Saving Excel spreadsheet...");
        }
    }
    
    @Override
    public void close() {
        isOpen = false;
        System.out.println("Closing Excel spreadsheet");
    }
    
    @Override
    public void print() {
        if (isOpen) {
            System.out.println("Printing Excel spreadsheet...");
        }
    }
    
    @Override
    public String getType() {
        return "Microsoft Excel Spreadsheet";
    }
    
    public void addWorksheet(String name) {
        if (isOpen) {
            worksheets.put(name, "");
            System.out.println("Added worksheet: " + name);
        }
    }
    
    public void addFormula(String cell, String formula) {
        if (isOpen) {
            formulas.put(cell, formula);
            System.out.println("Added formula to " + cell + ": " + formula);
        }
    }
}
```

## Real-World Example: Database Connection Factory

### Database Connection Products

```java
/**
 * Database Connection interface
 */
public interface DatabaseConnection {
    void connect();
    void disconnect();
    ResultSet executeQuery(String sql);
    int executeUpdate(String sql);
    void beginTransaction();
    void commit();
    void rollback();
    String getConnectionType();
    String getConnectionString();
}

/**
 * MySQL Connection Implementation
 */
public class MySQLConnection implements DatabaseConnection {
    
    private final String host;
    private final String database;
    private final String username;
    private final String password;
    private boolean connected = false;
    private boolean inTransaction = false;
    
    public MySQLConnection(String host, String database, String username, String password) {
        this.host = host;
        this.database = database;
        this.username = username;
        this.password = password;
    }
    
    @Override
    public void connect() {
        if (!connected) {
            System.out.println("Connecting to MySQL database: " + host + "/" + database);
            // Simulate connection logic
            connected = true;
            System.out.println("MySQL connection established");
        }
    }
    
    @Override
    public void disconnect() {
        if (connected) {
            if (inTransaction) {
                rollback();
            }
            connected = false;
            System.out.println("MySQL connection closed");
        }
    }
    
    @Override
    public ResultSet executeQuery(String sql) {
        checkConnection();
        System.out.println("Executing MySQL query: " + sql);
        // Return mock result set
        return new MockResultSet(sql, "MySQL");
    }
    
    @Override
    public int executeUpdate(String sql) {
        checkConnection();
        System.out.println("Executing MySQL update: " + sql);
        return 1; // Mock affected rows
    }
    
    @Override
    public void beginTransaction() {
        checkConnection();
        inTransaction = true;
        System.out.println("MySQL transaction started");
    }
    
    @Override
    public void commit() {
        checkConnection();
        if (inTransaction) {
            inTransaction = false;
            System.out.println("MySQL transaction committed");
        }
    }
    
    @Override
    public void rollback() {
        checkConnection();
        if (inTransaction) {
            inTransaction = false;
            System.out.println("MySQL transaction rolled back");
        }
    }
    
    @Override
    public String getConnectionType() {
        return "MySQL";
    }
    
    @Override
    public String getConnectionString() {
        return String.format("mysql://%s/%s", host, database);
    }
    
    private void checkConnection() {
        if (!connected) {
            throw new RuntimeException("Not connected to MySQL database");
        }
    }
}

/**
 * PostgreSQL Connection Implementation
 */
public class PostgreSQLConnection implements DatabaseConnection {
    
    private final String host;
    private final String database;
    private final String username;
    private final String password;
    private boolean connected = false;
    private boolean inTransaction = false;
    
    public PostgreSQLConnection(String host, String database, String username, String password) {
        this.host = host;
        this.database = database;
        this.username = username;
        this.password = password;
    }
    
    @Override
    public void connect() {
        if (!connected) {
            System.out.println("Connecting to PostgreSQL database: " + host + "/" + database);
            connected = true;
            System.out.println("PostgreSQL connection established");
        }
    }
    
    @Override
    public void disconnect() {
        if (connected) {
            if (inTransaction) {
                rollback();
            }
            connected = false;
            System.out.println("PostgreSQL connection closed");
        }
    }
    
    @Override
    public ResultSet executeQuery(String sql) {
        checkConnection();
        System.out.println("Executing PostgreSQL query: " + sql);
        return new MockResultSet(sql, "PostgreSQL");
    }
    
    @Override
    public int executeUpdate(String sql) {
        checkConnection();
        System.out.println("Executing PostgreSQL update: " + sql);
        return 1;
    }
    
    @Override
    public void beginTransaction() {
        checkConnection();
        inTransaction = true;
        System.out.println("PostgreSQL transaction started");
    }
    
    @Override
    public void commit() {
        checkConnection();
        if (inTransaction) {
            inTransaction = false;
            System.out.println("PostgreSQL transaction committed");
        }
    }
    
    @Override
    public void rollback() {
        checkConnection();
        if (inTransaction) {
            inTransaction = false;
            System.out.println("PostgreSQL transaction rolled back");
        }
    }
    
    @Override
    public String getConnectionType() {
        return "PostgreSQL";
    }
    
    @Override
    public String getConnectionString() {
        return String.format("postgresql://%s/%s", host, database);
    }
    
    private void checkConnection() {
        if (!connected) {
            throw new RuntimeException("Not connected to PostgreSQL database");
        }
    }
}

/**
 * Mock ResultSet for demonstration
 */
class MockResultSet implements ResultSet {
    private final String query;
    private final String dbType;
    
    public MockResultSet(String query, String dbType) {
        this.query = query;
        this.dbType = dbType;
    }
    
    @Override
    public String toString() {
        return String.format("MockResultSet{query='%s', dbType='%s'}", query, dbType);
    }
    
    // Implement minimal ResultSet methods for demo
    @Override public boolean next() { return false; }
    @Override public void close() {}
    @Override public boolean wasNull() { return false; }
    // ... other methods omitted for brevity
}
```

### Database Factory Implementation

```java
/**
 * Abstract Database Factory
 */
public abstract class DatabaseFactory {
    
    protected String host;
    protected String database;
    protected String username;
    protected String password;
    
    public DatabaseFactory(String host, String database, String username, String password) {
        this.host = host;
        this.database = database;
        this.username = username;
        this.password = password;
    }
    
    // Factory method
    public abstract DatabaseConnection createConnection();
    
    // Template method using factory method
    public DatabaseConnection getConnection() {
        DatabaseConnection connection = createConnection();
        connection.connect();
        return connection;
    }
    
    public void executeTransactionalOperation(String... sqlStatements) {
        DatabaseConnection connection = getConnection();
        
        try {
            connection.beginTransaction();
            
            for (String sql : sqlStatements) {
                if (sql.trim().toUpperCase().startsWith("SELECT")) {
                    ResultSet rs = connection.executeQuery(sql);
                    System.out.println("Query result: " + rs);
                } else {
                    int affected = connection.executeUpdate(sql);
                    System.out.println("Rows affected: " + affected);
                }
            }
            
            connection.commit();
            System.out.println("Transaction completed successfully");
            
        } catch (Exception e) {
            System.err.println("Transaction failed: " + e.getMessage());
            connection.rollback();
        } finally {
            connection.disconnect();
        }
    }
}

/**
 * MySQL Database Factory
 */
public class MySQLDatabaseFactory extends DatabaseFactory {
    
    public MySQLDatabaseFactory(String host, String database, String username, String password) {
        super(host, database, username, password);
    }
    
    @Override
    public DatabaseConnection createConnection() {
        System.out.println("Creating MySQL connection using MySQL factory");
        return new MySQLConnection(host, database, username, password);
    }
}

/**
 * PostgreSQL Database Factory
 */
public class PostgreSQLDatabaseFactory extends DatabaseFactory {
    
    public PostgreSQLDatabaseFactory(String host, String database, String username, String password) {
        super(host, database, username, password);
    }
    
    @Override
    public DatabaseConnection createConnection() {
        System.out.println("Creating PostgreSQL connection using PostgreSQL factory");
        return new PostgreSQLConnection(host, database, username, password);
    }
}

/**
 * Database Factory Provider
 */
public class DatabaseFactoryProvider {
    
    public enum DatabaseType {
        MYSQL, POSTGRESQL, ORACLE, MONGODB
    }
    
    public static DatabaseFactory createFactory(DatabaseType type, String host, 
                                              String database, String username, String password) {
        switch (type) {
            case MYSQL:
                return new MySQLDatabaseFactory(host, database, username, password);
            case POSTGRESQL:
                return new PostgreSQLDatabaseFactory(host, database, username, password);
            case ORACLE:
                return new OracleDatabaseFactory(host, database, username, password);
            case MONGODB:
                return new MongoDBDatabaseFactory(host, database, username, password);
            default:
                throw new IllegalArgumentException("Unsupported database type: " + type);
        }
    }
    
    public static DatabaseType detectDatabaseType(String connectionUrl) {
        if (connectionUrl.startsWith("mysql://") || connectionUrl.startsWith("jdbc:mysql://")) {
            return DatabaseType.MYSQL;
        } else if (connectionUrl.startsWith("postgresql://") || connectionUrl.startsWith("jdbc:postgresql://")) {
            return DatabaseType.POSTGRESQL;
        } else if (connectionUrl.contains("oracle")) {
            return DatabaseType.ORACLE;
        } else if (connectionUrl.startsWith("mongodb://")) {
            return DatabaseType.MONGODB;
        } else {
            throw new IllegalArgumentException("Cannot detect database type from URL: " + connectionUrl);
        }
    }
}
```

## Configuration-Based Factory

### Configurable Factory with Properties

```java
/**
 * Configuration-driven factory
 */
@Component
public class ConfigurableDocumentFactory {
    
    @Value("${document.default.type:TEXT}")
    private String defaultDocumentType;
    
    @Value("${document.templates.enabled:false}")
    private boolean templatesEnabled;
    
    private final Map<String, DocumentCreator> creators = new HashMap<>();
    
    @PostConstruct
    public void initializeCreators() {
        creators.put("WORD", this::createWordDocument);
        creators.put("PDF", this::createPDFDocument);
        creators.put("TEXT", this::createTextDocument);
        creators.put("POWERPOINT", this::createPowerPointDocument);
        creators.put("EXCEL", this::createExcelDocument);
    }
    
    public Document createDocument(String type) {
        DocumentCreator creator = creators.get(type.toUpperCase());
        
        if (creator == null) {
            throw new IllegalArgumentException("Unsupported document type: " + type);
        }
        
        Document document = creator.create();
        
        if (templatesEnabled) {
            applyTemplate(document, type);
        }
        
        return document;
    }
    
    public Document createDefaultDocument() {
        return createDocument(defaultDocumentType);
    }
    
    private Document createWordDocument() {
        return new WordDocument();
    }
    
    private Document createPDFDocument() {
        return new PDFDocument();
    }
    
    private Document createTextDocument() {
        return new TextDocument();
    }
    
    private Document createPowerPointDocument() {
        return new PowerPointDocument();
    }
    
    private Document createExcelDocument() {
        return new ExcelDocument();
    }
    
    private void applyTemplate(Document document, String type) {
        System.out.println("Applying " + type + " template to document");
        // Template application logic
    }
    
    @FunctionalInterface
    private interface DocumentCreator {
        Document create();
    }
}

/**
 * Service using configurable factory
 */
@Service
public class DocumentService {
    
    private final ConfigurableDocumentFactory documentFactory;
    
    public DocumentService(ConfigurableDocumentFactory documentFactory) {
        this.documentFactory = documentFactory;
    }
    
    public void createAndProcessDocument(String type, String content) {
        try {
            Document document = documentFactory.createDocument(type);
            
            document.open();
            
            // Type-specific content addition
            addContentToDocument(document, content);
            
            document.save();
            document.print();
            document.close();
            
        } catch (Exception e) {
            System.err.println("Error processing document: " + e.getMessage());
        }
    }
    
    public void createDefaultDocument() {
        Document document = documentFactory.createDefaultDocument();
        document.open();
        System.out.println("Created default document: " + document.getType());
        document.close();
    }
    
    private void addContentToDocument(Document document, String content) {
        if (document instanceof TextDocument) {
            ((TextDocument) document).appendText(content);
        } else if (document instanceof WordDocument) {
            ((WordDocument) document).addFormatting("Content: " + content);
        }
        // Add more type-specific handling as needed
    }
}
```

## Lambda-Based Factory (Modern Approach)

### Functional Factory Implementation

```java
/**
 * Modern functional factory using lambdas
 */
public class FunctionalDocumentFactory {
    
    private final Map<String, Supplier<Document>> documentCreators = new HashMap<>();
    private final Map<String, Consumer<Document>> documentProcessors = new HashMap<>();
    
    public FunctionalDocumentFactory() {
        initializeCreators();
        initializeProcessors();
    }
    
    private void initializeCreators() {
        documentCreators.put("WORD", WordDocument::new);
        documentCreators.put("PDF", PDFDocument::new);
        documentCreators.put("TEXT", TextDocument::new);
        documentCreators.put("POWERPOINT", PowerPointDocument::new);
        documentCreators.put("EXCEL", ExcelDocument::new);
    }
    
    private void initializeProcessors() {
        documentProcessors.put("WORD", doc -> {
            if (doc instanceof WordDocument) {
                ((WordDocument) doc).addFormatting("Default Word formatting");
            }
        });
        
        documentProcessors.put("PDF", doc -> {
            if (doc instanceof PDFDocument) {
                ((PDFDocument) doc).setReadOnly(false);
            }
        });
        
        documentProcessors.put("TEXT", doc -> {
            if (doc instanceof TextDocument) {
                ((TextDocument) doc).appendText("Default text content");
            }
        });
    }
    
    public Document createDocument(String type) {
        return createDocument(type, true);
    }
    
    public Document createDocument(String type, boolean applyProcessing) {
        Supplier<Document> creator = documentCreators.get(type.toUpperCase());
        
        if (creator == null) {
            throw new IllegalArgumentException("Unsupported document type: " + type);
        }
        
        Document document = creator.get();
        
        if (applyProcessing) {
            Consumer<Document> processor = documentProcessors.get(type.toUpperCase());
            if (processor != null) {
                document.open();
                processor.accept(document);
                document.close();
            }
        }
        
        return document;
    }
    
    public <T extends Document> T createDocumentWithCallback(String type, 
                                                           Function<Document, T> customProcessor) {
        Document document = createDocument(type, false);
        document.open();
        
        T processedDocument = customProcessor.apply(document);
        
        document.close();
        return processedDocument;
    }
    
    // Fluent interface for document creation
    public DocumentBuilder builder(String type) {
        return new DocumentBuilder(type, this);
    }
    
    public static class DocumentBuilder {
        private final String type;
        private final FunctionalDocumentFactory factory;
        private boolean withProcessing = true;
        private final List<Consumer<Document>> customProcessors = new ArrayList<>();
        
        public DocumentBuilder(String type, FunctionalDocumentFactory factory) {
            this.type = type;
            this.factory = factory;
        }
        
        public DocumentBuilder withoutDefaultProcessing() {
            this.withProcessing = false;
            return this;
        }
        
        public DocumentBuilder withCustomProcessor(Consumer<Document> processor) {
            this.customProcessors.add(processor);
            return this;
        }
        
        public Document build() {
            Document document = factory.createDocument(type, withProcessing);
            
            if (!customProcessors.isEmpty()) {
                document.open();
                customProcessors.forEach(processor -> processor.accept(document));
                document.close();
            }
            
            return document;
        }
    }
}
```

## Spring Boot Integration

### Spring Configuration

```java
/**
 * Spring configuration for factory pattern
 */
@Configuration
@EnableConfigurationProperties(DocumentProperties.class)
public class FactoryConfig {
    
    @Bean
    @Primary
    public ConfigurableDocumentFactory documentFactory() {
        return new ConfigurableDocumentFactory();
    }
    
    @Bean
    public FunctionalDocumentFactory functionalDocumentFactory() {
        return new FunctionalDocumentFactory();
    }
    
    @Bean
    @ConditionalOnProperty(value = "database.enabled", havingValue = "true")
    public DatabaseFactory databaseFactory(DatabaseProperties dbProperties) {
        return DatabaseFactoryProvider.createFactory(
            DatabaseFactoryProvider.DatabaseType.valueOf(dbProperties.getType().toUpperCase()),
            dbProperties.getHost(),
            dbProperties.getDatabase(),
            dbProperties.getUsername(),
            dbProperties.getPassword()
        );
    }
}

/**
 * Configuration properties
 */
@ConfigurationProperties(prefix = "document")
@Data
public class DocumentProperties {
    private String defaultType = "TEXT";
    private boolean templatesEnabled = false;
    private Map<String, String> templates = new HashMap<>();
}

@ConfigurationProperties(prefix = "database")
@Data
public class DatabaseProperties {
    private String type = "mysql";
    private String host = "localhost";
    private String database = "testdb";
    private String username = "root";
    private String password = "";
}

/**
 * REST Controller using factories
 */
@RestController
@RequestMapping("/api/documents")
public class DocumentController {
    
    private final ConfigurableDocumentFactory documentFactory;
    private final FunctionalDocumentFactory functionalFactory;
    
    public DocumentController(ConfigurableDocumentFactory documentFactory,
                            FunctionalDocumentFactory functionalFactory) {
        this.documentFactory = documentFactory;
        this.functionalFactory = functionalFactory;
    }
    
    @PostMapping("/create/{type}")
    public ResponseEntity<String> createDocument(@PathVariable String type,
                                               @RequestBody(required = false) String content) {
        try {
            Document document = documentFactory.createDocument(type);
            document.open();
            
            String result = String.format("Created document: %s", document.getType());
            document.close();
            
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error creating document: " + e.getMessage());
        }
    }
    
    @PostMapping("/create/functional/{type}")
    public ResponseEntity<String> createFunctionalDocument(@PathVariable String type,
                                                          @RequestParam(defaultValue = "false") boolean withCustomProcessing) {
        try {
            Document document;
            
            if (withCustomProcessing) {
                document = functionalFactory.builder(type)
                    .withCustomProcessor(doc -> System.out.println("Custom processing applied"))
                    .build();
            } else {
                document = functionalFactory.createDocument(type);
            }
            
            String result = String.format("Created functional document: %s", document.getType());
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            return ResponseEntity.badRequest().body("Error creating document: " + e.getMessage());
        }
    }
}
```

## Testing Factory Method Pattern

### Unit Tests

```java
/**
 * Tests for Factory Method implementations
 */
@ExtendWith(MockitoExtension.class)
class FactoryMethodTest {
    
    @Test
    void testDocumentApplicationFactories() {
        // Test different document applications
        DocumentApplication wordApp = new WordApplication();
        DocumentApplication pdfApp = new PDFApplication();
        DocumentApplication textApp = new TextApplication();
        
        // Test document creation
        Document wordDoc = wordApp.createDocument();
        Document pdfDoc = pdfApp.createDocument();
        Document textDoc = textApp.createDocument();
        
        assertThat(wordDoc).isInstanceOf(WordDocument.class);
        assertThat(pdfDoc).isInstanceOf(PDFDocument.class);
        assertThat(textDoc).isInstanceOf(TextDocument.class);
        
        // Test document functionality
        wordDoc.open();
        assertThat(wordDoc.getType()).isEqualTo("Microsoft Word Document");
        wordDoc.close();
    }
    
    @Test
    void testParameterizedFactory() {
        EnhancedDocumentApplication app = new EnhancedDocumentApplication();
        
        // Test all document types
        for (EnhancedDocumentApplication.DocumentType type : 
             EnhancedDocumentApplication.DocumentType.values()) {
            
            Document doc = app.createDocument(type);
            assertThat(doc).isNotNull();
            
            doc.open();
            assertThat(doc.getType()).isNotBlank();
            doc.close();
        }
    }
    
    @Test
    void testDatabaseFactory() {
        DatabaseFactory mysqlFactory = new MySQLDatabaseFactory(
            "localhost", "testdb", "root", "password"
        );
        
        DatabaseFactory pgFactory = new PostgreSQLDatabaseFactory(
            "localhost", "testdb", "postgres", "password"
        );
        
        // Test MySQL connection
        DatabaseConnection mysqlConn = mysqlFactory.createConnection();
        assertThat(mysqlConn).isInstanceOf(MySQLConnection.class);
        assertThat(mysqlConn.getConnectionType()).isEqualTo("MySQL");
        
        // Test PostgreSQL connection
        DatabaseConnection pgConn = pgFactory.createConnection();
        assertThat(pgConn).isInstanceOf(PostgreSQLConnection.class);
        assertThat(pgConn.getConnectionType()).isEqualTo("PostgreSQL");
        
        mysqlConn.disconnect();
        pgConn.disconnect();
    }
    
    @Test
    void testFunctionalFactory() {
        FunctionalDocumentFactory factory = new FunctionalDocumentFactory();
        
        // Test basic creation
        Document doc = factory.createDocument("WORD");
        assertThat(doc).isInstanceOf(WordDocument.class);
        
        // Test builder pattern
        Document builtDoc = factory.builder("TEXT")
            .withCustomProcessor(d -> System.out.println("Custom processing"))
            .build();
        
        assertThat(builtDoc).isInstanceOf(TextDocument.class);
    }
    
    @Test
    void testFactoryErrorHandling() {
        EnhancedDocumentApplication app = new EnhancedDocumentApplication();
        
        // Test invalid document type
        assertThrows(IllegalArgumentException.class, () -> {
            app.createDocument(null);
        });
    }
}

/**
 * Integration test
 */
@SpringBootTest
@TestPropertySource(properties = {
    "document.default.type=WORD",
    "document.templates.enabled=true"
})
class FactoryIntegrationTest {
    
    @Autowired
    private ConfigurableDocumentFactory documentFactory;
    
    @Autowired
    private DocumentService documentService;
    
    @Test
    void testSpringFactoryConfiguration() {
        // Test default document creation
        Document defaultDoc = documentFactory.createDefaultDocument();
        assertThat(defaultDoc).isInstanceOf(WordDocument.class);
        
        // Test service integration
        documentService.createDefaultDocument();
        documentService.createAndProcessDocument("PDF", "Test content");
    }
}
```

## Performance Considerations

### Factory with Object Pooling

```java
/**
 * Factory with object pooling for expensive objects
 */
public class PooledDocumentFactory {
    
    private final Map<String, Queue<Document>> documentPools = new ConcurrentHashMap<>();
    private final int maxPoolSize = 10;
    
    public PooledDocumentFactory() {
        initializePools();
    }
    
    private void initializePools() {
        documentPools.put("WORD", new ConcurrentLinkedQueue<>());
        documentPools.put("PDF", new ConcurrentLinkedQueue<>());
        documentPools.put("TEXT", new ConcurrentLinkedQueue<>());
    }
    
    public Document getDocument(String type) {
        Queue<Document> pool = documentPools.get(type.toUpperCase());
        
        if (pool == null) {
            throw new IllegalArgumentException("Unsupported document type: " + type);
        }
        
        Document document = pool.poll();
        
        if (document == null) {
            document = createNewDocument(type);
            System.out.println("Created new " + type + " document");
        } else {
            System.out.println("Retrieved " + type + " document from pool");
        }
        
        return document;
    }
    
    public void returnDocument(Document document) {
        String type = getDocumentType(document);
        Queue<Document> pool = documentPools.get(type);
        
        if (pool != null && pool.size() < maxPoolSize) {
            // Reset document state before returning to pool
            document.close();
            pool.offer(document);
            System.out.println("Returned " + type + " document to pool");
        }
    }
    
    private Document createNewDocument(String type) {
        switch (type.toUpperCase()) {
            case "WORD":
                return new WordDocument();
            case "PDF":
                return new PDFDocument();
            case "TEXT":
                return new TextDocument();
            default:
                throw new IllegalArgumentException("Unsupported document type: " + type);
        }
    }
    
    private String getDocumentType(Document document) {
        if (document instanceof WordDocument) return "WORD";
        if (document instanceof PDFDocument) return "PDF";
        if (document instanceof TextDocument) return "TEXT";
        throw new IllegalArgumentException("Unknown document type");
    }
    
    public void clearPools() {
        documentPools.values().forEach(Queue::clear);
        System.out.println("All document pools cleared");
    }
    
    public Map<String, Integer> getPoolSizes() {
        return documentPools.entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> entry.getValue().size()
            ));
    }
}
```

## Best Practices

### ✅ **Do's:**

1. **Use abstract base class or interface** for consistent product creation
2. **Implement proper error handling** for unsupported types
3. **Consider using enums** for type safety in parameterized factories
4. **Leverage dependency injection** frameworks for configuration
5. **Document factory capabilities** and supported product types
6. **Make factories thread-safe** when needed
7. **Consider object pooling** for expensive objects

### ❌ **Don'ts:**

1. **Don't create factories for simple objects** - use constructors directly
2. **Don't ignore null checks** and validation
3. **Don't make factory methods too complex** - keep creation logic simple
4. **Don't forget about memory management** in pooled factories
5. **Don't hard-code product types** - use configuration when possible
6. **Don't create deep inheritance hierarchies** unnecessarily

## When to Use Factory Method Pattern

### ✅ **Good Use Cases:**
- **Complex object creation**: When construction involves multiple steps
- **Runtime type determination**: Object type decided at runtime
- **Configuration-based creation**: Different objects based on configuration
- **Testing**: Easy to mock and test different implementations
- **Plugin architectures**: Support for different implementations
- **Framework development**: Allow users to extend with custom types

### ❌ **Avoid Factory Method For:**
- **Simple object creation**: Direct instantiation is sufficient
- **Single concrete product**: No need for factory abstraction
- **Performance-critical code**: Factory overhead might be significant
- **Unchanging requirements**: No need for future extension

## Related Patterns

- **Abstract Factory**: Creates families of related objects
- **Builder**: Constructs complex objects step by step
- **Singleton**: Often used with factories
- **Template Method**: Often used in factory implementations

## Summary

The Factory Method pattern is excellent for:

**Key Benefits:**
- **Flexibility**: Easy to add new product types
- **Decoupling**: Separates creation from usage
- **Testability**: Easy to mock different implementations
- **Configuration**: Runtime selection of object types

**Common Variations:**
- **Simple Factory**: Static method that creates objects
- **Parameterized Factory**: Uses parameters to determine type
- **Configurable Factory**: Uses configuration to determine behavior
- **Functional Factory**: Uses lambda expressions and functional interfaces

Use Factory Method when you need flexibility in object creation and want to support multiple types of products!
