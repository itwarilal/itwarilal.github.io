---
layout: post
title:  "Cassandra Schema Migration for Spring-Boot Projects"
date:   2018-10-06 11:20:32 +0700
categories: [java, spring-boot]
---
## What is Schema Migration?

Typically if you are developing backend web application or REST APIs there may be need for data storage. There are 
numerous options on database choice ranging from typical RDBMS systems to NoSQL databases. As you develop and iterate on
your application there might be changes to database schema and you would want to migrate these database changes 
across your environments in automated way. Database schema migration solutions do provide such support, for RDBMS 
[Flyway](https://flywaydb.org), [Liquibase](http://www.liquibase.org) and [MyBatis](http://www.mybatis.org/migrations/) 
are proven solutions. In this article we are going to focus on how to manage schema migration for Cassandra.

## Why to use Schema Migration?

Idea behind Schema Migration is that you are can automatically apply schema changes across all your environments in 
consistent fashion. Such solution rules out any inadvertent human error with applying manual changes and it also makes 
rebuilding any environment breeze.    

## How does Schema Migration work?

Schema migration can be done either as a standalone solution which resides outside of application or service. You can
execute those migration before your related application changes are deployed. Alternative approach is to embedded 
schema migration into your application and it gets executed as part of application startup. There are several pros and 
cons around both approaches but I personally prefer the later as it fits better with CI/CD model.

Schema migration frameworks generally work by creating tracking table which keeps tab on which schema migration script 
has been applied on that particular environment. All the schema migration scripts are given unique version number 
or primary identifier to keep track of the progress of migration. Additionally, in some frameworks a locking table is 
used to make sure multiple instances of application does not execute same migration. Since RDBMS systems are strongly 
consistent it guarantees that other application instances see the status of migration and schema changes.  

## Challenges with Schema Migration on Cassandra

Inherently Cassandra does not support strong consistency but uses eventual consistency model. Migration frameworks rely
on JDBC drivers and transaction support to guarantee schema migration consistency. This makes it very challenging for 
the typical schema migration frameworks to work with Cassandra. There have been several attempts to write JDBC wrappers 
around Cassandra driver but they haven't been very successful. 

## Spring Boot and Cassandra Support

spring-boot makes is very easy to integrate Cassandra support into your applications. If you have used spring-data 
module for RDBMS support, then replacing it to use Cassandra support will be breeze. spring-data provides easy to use
Repository pattern which can be easily changed to work with RDMBS or Cassandra very easily. spring-data-cassandra module
provides great flexibility in its approach with 3 different ways to interact with Cassandra - repository pattern for 
domain based interaction, CassandraTemplate for better control over queries using domain objects and CqlTemplate for 
low level cql like interactions.

As it is very common with spring modules, spring-data-cassandra is developed with lot of extensibility and great 
thoughts around development lifecycle. Unfortunately, spring-data-cassandra provides very rudimentary support for
Cassandra Schema migration.

spring-data-cassandra provides support to specify keyspace creation and other data migration scripts. Spring can also 
introspect domain objects or entity classes and create Cassandra table specifications for those.

Generally, speaking I like greater control over generated schema hence I prefer explicitly creating data migration 
scripts for schema changes instead of using entity based auto-generation of scripts. This approach makes it easy to 
enforce any conventions you might have around database changes. spring-data-cassandra also provides script based 
approach but lacks versioning, metadata information and tracking as you would expect from schema migration solutions.

There are couple of good frameworks that make it easy to support schema migration for Cassandra and provide very similar
behavior/functionality as their RDBMS counterpart. We are going to go into more detail into one of such solution
that I have been using for sometime - 
[smartcat labs cassandra migration tool](https://github.com/smartcat-labs/cassandra-migration-tool-java) 

### Requirements

- Schema migrations can be versioned 
- Schema migrations can be migrated with the application changes
- Cassandra is eventually consistent so there should be significant guarantee that there was agreement with-in cluster 
nodes
- For local development in-memory Cassandra would be used so schema migration should work with that setup as well  

### Solution

Simple spring-boot web application with spring-data-cassandra setup. Smartcat lab's migration framework to implement 
schema migration.

Generally, spring provides great setup for local development by bootstrapping in-memory database. But spring lacks such
support when it comes to Cassandra. You can add in-memory Cassandra support by using 
[cassandra-unit](https://github.com/jsevellec/cassandra-unit).

As spring does generally, it provides different ways to modify Cassandra auto-configuration so that we can modify 
auto configured components.   

I have used Java 8, Spring-Boot 2.0.1 for snippets but this also works for Spring-Boot 1.x services.

#### In-Memory Cassandra Support

To start, you can add cassandra-unit to add embedded cassandra server to your project. 

```xml
<dependency>
    <groupId>org.cassandraunit</groupId>
    <artifactId>cassandra-unit-spring</artifactId>
    <version>3.5.0.1</version>
</dependency>
```
Spring's auto-configuration does most of the heavy lifting like instantiating cassandra driver, providing necessary 
defaults for driver and creating beans for Session, CqlTemplate, CassandraTemplate and proxy classes for Repository objects. 

Add below configuration so that embedded cassandra server is started as part of your application start. Adding 
*@Profile("default")* ensures that embedded cassandra is only started when starting the application on local machine. 

```java
@Configuration
@Profile("default")
public class LocalDevConfig extends AbstractCassandraConfiguration {

    private static final Logger LOGGER = LoggerFactory.getLogger(LocalDevConfig.class);

    @Value("${spring.data.cassandra.keyspace-name}")
    private String keyspace;

    @Value("${spring.data.cassandra.port}")
    private int port;

    @PostConstruct
    public void startEmbeddedCassandra() throws InterruptedException, IOException, TTransportException {
        EmbeddedCassandraServerHelper.startEmbeddedCassandra();

        LOGGER.info("Started Embedded Cassandra Cluster");
    }

    @PreDestroy
    public void stopEmbeddedCassandra(){
        EmbeddedCassandraServerHelper.cleanEmbeddedCassandra();

        LOGGER.info("Disposed Embedded Cassandra Cluster");
    }

    @Override
    protected List<CreateKeyspaceSpecification> getKeyspaceCreations() {
        return Collections.singletonList(CreateKeyspaceSpecification.createKeyspace(keyspace)
                .ifNotExists()
                .with(KeyspaceOption.DURABLE_WRITES, true)
                .withSimpleReplication());
    }

    @Override
    protected String getKeyspaceName() {
        return keyspace;
    }

    @Override
    protected int getPort() {
        return port;
    }
}
```

Extending *AbstractCassandraConfiguration* allows you to hook keyspace creation as part of application start. But before 
keySpace is created you also need to make sure that embedded cassandra server is started. This can be achieved by 
specifying that as part of *@PostConstruct* on the configuration itself. This helps with local development setup. 

I am sure there are different ways to achieve similar configuration but this was one that worked out for me.

#### Integrate Smartcat Lab's Migration Manager

Next I wanted to add data migration support which works similar way across both local and deployed environment and it 
works in same way without any special handling between embedded cassandra vs deployed cassandra.

Add below dependency for smartcat lab's migration manager component -

```xml
<dependency>
    <groupId>io.smartcat</groupId>
    <artifactId>cassandra-migration-tool</artifactId>
    <version>2.1.9.0</version>
</dependency>
```

Once you have this dependency you can start adding database migration scripts. These scripts are simple java classes
that extend *SchemaMigration* class. Version number field is used to order the migration of scripts.

```java
public class AddBooksTable extends SchemaMigration {

    public AddBooksTable(final int version) {
        super(version);
    }

    @Override
    public String getDescription() {
        return "Add Books Table";
    }

    @Override
    public void execute() throws MigrationException {
        final String createTable = SchemaBuilder
                .createTable("books")
                .addPartitionKey("id", text())
                .addClusteringColumn("author", text())
                .addColumn("description", text())
                .addColumn("createtime", timestamp())
                .ifNotExists().withOptions()
                .comment("Books table created")
                .buildInternal();

        executeWithSchemaAgreement(new SimpleStatement(createTable));
    }
}
```

Important thing to note is that these scripts are executed using special clause *executeWithSchemaAgreement* which 
ensures that all the nodes in the cluster have agreed on schema change. In absence of transaction support and eventual
consistency model this clause guarantees that schema change has been committed.

I added below configuration to support data migration scripts so that migration scripts are executed as part of 
application startup. I would like to further enhance this such that migration scripts are auto discovered and there
is not need to add scripts in the list when new one is created.

```java
@Component
public class MigrationManager {

    private static final Logger LOGGER = LoggerFactory.getLogger(MigrationManager.class);

    @Value("${spring.data.cassandra.keyspace-name}")
    private String keyspace;

    @Autowired
    private Session session;

    public void doMigration(){
        Assert.notNull(session, "Session object is null");
        Assert.hasText(keyspace, "Keyspace cannot be null or empty");
        printMetadata(session);
        migrateSchema(session);
    }

    private void migrateSchema(final Session session) {
        LOGGER.info("Executing schema migrations");

        final MigrationResources resources = findMigrationResources();

        MigrationEngine.withSession(session).migrate(resources);

        LOGGER.info("Done with schema migrations");
    }

    private MigrationResources findMigrationResources(){
        MigrationResources resources = new MigrationResources();
        resources.addMigration(new InitializeSchema(1));
        resources.addMigration(new AddBooksTable(2));
        return resources;
    }

    private void printMetadata(final Session session) {
        final Metadata metadata = session.getCluster().getMetadata();
        LOGGER.info("Connected to cluster = {}", metadata.getClusterName());

        for (final Host host : metadata.getAllHosts()) {
            LOGGER.info("Datacenter = {} host = {}", host.getDatacenter(), host.getAddress());
        }
    }

    @PostConstruct
    public void setup(){
        doMigration();
    }
}

``` 

One problem that I ran into with this setup was that order of creation of various cassandra configuration classes is 
non-deterministic and it would result in startup failure. After investigation I found that sometimes migration scripts 
will get executed before cassandra session got created. I tried using *@DependsOn* to force creation of session before 
script execution but I did not find good solution.

After digging through some of the Cassandra related auto-configuration I found this gem - 
*AbstractDependsOnBeanFactoryPostProcessor*. So I added below configuration and that worked out perfectly.

```java
@Configuration
public class MigrationConfig {

    @Order
    @Configuration
    public static class CassandraDriverDependsOnBeanFactoryPostProcessor
            extends AbstractDependsOnBeanFactoryPostProcessor {

        public CassandraDriverDependsOnBeanFactoryPostProcessor() {
            super(CqlTemplate.class, CassandraCqlTemplateFactoryBean.class, "migrationManager");
        }
    }
}
```
  
This configuration basically says that instances *CqlTemplate* and *CassandraCqlTemplateFactoryBean* should be created 
before *migrationManager* is created. Which in turn results in Cassandra Session bean getting created as it is needed 
to initialize CqlTemplate.

With little bit of configuration above it makes it easy to setup database migration for cassandra which works in
consistent way across all your environments. Although, it is small and easy setup, I wished spring had provided such 
configuration to make it easy for developers to support cassandra development.
    

 



 
