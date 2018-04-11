Ethercis
========


What is it?
-----------

An [openEHR](http://www.openehr.org/) CDR essentially based on SQL that exposes its services *via* a REST API.

More documentation about the concepts and architecture of EtherCIS is located [here](http://docs.ethercis.org/)

Please also have a look to our [roadmap](https://github.com/ethercis/ethercis/blob/master/ethercis-roadmap.md) for more details

What's new?
-----------
- Posted a new (a first prelease) of the installation package. It contains scripts and libraries to perform a complete installation of Postgresql 10 and EtherCIS server with preconfiguration of the DB. It contains also a set of openEHR operational templates to get you started. The package is in the "Releases" [section](https://github.com/ethercis/ethercis/releases/tag/v1.1.0-beta). NB. This installation has been tested on CentOS 7 (should work on RHEL 7).

Changes
======

v1.1.2 (Apr 10 2018)
--------------------
This version merges Sheref's PR to allow CI using Travis.

There are several changes including:

- Tests are more or less operational but nevertheless work is needed to make them more meaningful (as well as coverage). To disable the tests, set maven skip test flag to ```true``` in the respective POMs:

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.19</version>
                <configuration>
                    <skipTests>true</skipTests>
                </configuration>
            </plugin>

- This version now uses PostgreSQL v10+. This is due mainly to better support returning compositions from AQL under
a (canonical) JSON format. PostgreSQL 10 comes with interesting jsonb functions that makes this part easier since 
JSON encoding can be partially done at DB level (NB. in the future this encoding shall be totally performed at DB level). The corresponding DB functions are in a flyway migration [script](https://github.com/ethercis/ehrservice/blob/remote-github/db/src/main/resources/db/migration/V5__raw_json_encoding.sql)
which can be run manually

- To run the tests, it is expected that a DB is installed locally and contains test data. The test data can be restored
from a [backup file](https://github.com/ethercis/VirtualEhr/blob/master/file_repo/db_test/testdb-pg10.backup). The restore
can be done using PGAdmin4 (since we use PostgreSQL 10). An easy way to proceed is to CASCADE DELETE schema 'ehr' and perform the restore using pg_restore as described in this [document](https://github.com/ethercis/ethercis/blob/master/doc/DB%20administration.md). Please note that the referential integrity trigger must be disabled. 

The DB installation can be done using the script found [here](https://github.com/ethercis/deploy-n-scripts/blob/master/ethercis-install/v1.1.0/install-db.sh). The install process is described in the [deploy-n-scripts](https://github.com/ethercis/deploy-n-scripts) section.


Spring-is-almost-there Edition Feb 2018
---------------------------------------

##### Changes in library structure

Few changes in the Uber jar generation to remove pesky dependencies on, yet-to-be-removed, org.openehr legacy classes. This has an impact on the classpath of the launch script to hold few more jars not included into the Uber jars anymore. Please note this will be modified soon as we are migrating to a continuous integration framework with Docker image generation.

The changes consists in the following classpath addition in ecis-server script:

```
${APPLIB}/CompositionTemplate.jar:\
${APPLIB}/openEHR.v1.Template.jar:\
${APPLIB}/composition_xml.jar:\
${APPLIB}/openEHR.v1.OperationalTemplate.jar
```
The above libraries have been added to lib/application repository.

The main repository lib/deploy is updated with the latest changes.

##### Operational Template Introspection (CR #74)

A new feature now support OPT introspection. Useful to automate some client UI construct or others. It is also opening the door to further data analytics potential as introspection results can be used to further support complex DB queries. See documentation in [OPT introspection](https://github.com/ethercis/ethercis/blob/master/doc/OPT%20introspection.md "OPT introspection")

##### Full template querying returning a JSON object (CR #73)

This changes allows to get a whole composition from a template in JSON format. 

To integrate this feature, a number of steps are required:

- Migration of PostgreSQL to at least 9.6 (10 is recommended)
- Installation of the functions supporting JSON encoding at DB level. A script is provided to help in this process. See in [resources/raw_json_encoding](https://github.com/ethercis/ehrservice/tree/remote-github/db/src/main/resources/raw_json_encoding)

In the future, we plan to support most of encoding/retrieval/querying at DB level only (by-passing most of the middleware logic) for performance reason.

##### Multiple fixes and enhancements

Please see the list of closed/in-test CRs for more details.


Project Structure
---
To allow various deployment and integration, the project is partitioned in two parts:

- core components: this part deals with OpenEhr object handling, serialization, deserialization, knowledge management and persistence
- service wrappers: this part encapsulate core components into a service framework with a REST API and JMX instrumentalization.

**Core Components**

The core modules are located in the repository [ehrservice](https://github.com/ethercis/ehrservice):

- core: fundamental operations and encoding of OpenEhr entities
- ehrdao: persistence of OpenEhr entities using a mixed model (relational/NoSql)
- knowledge-cache: caching of OpenEhr knowledge models (operational templates in particular)
- aql-processor: two passes SQL translation and query execution
- jooq-pg: utility module, binds ethercis table to jOOQ/Postgresql 9.4
- transform is mainly used to deal with raw json
- validation is responsible to check data input in relation to an openEHR template

**Service Wrappers**

The services and framework are located in [VirtualEhr](https://github.com/ethercis/VirtualEhr)

- `ServiceManager` service management framework
- `VEhrService` Query gateway of a running instance
- `ResourceAccessService` a common service to access external resources (DB, knowledge etc.)
- `PartyIdentifiedService` wrapper to interact with OpenEhr PartyIdentified entities
- `LogonService` controls user login/logout and sessions
- `AuthenticationService` wrap a security policy provider
- `CacheKnowledgeService` wrapper of knowledge-cache to allow user queries
- `EhrService` deals with user queries on OpenEhr Ehr and Ehr Status objects
- `CompositionService` deals with user queries on Composition objects

Please refer to the respective component's README for more details on the above

**Database**

The database is based on bi-temporal tables keeping records history. See [pgsql_ehr.ddl](https://github.com/ethercis/ehrservice/blob/remote-github/ehrdao/resources/ddls/pgsql_ehr.ddl) for more details on the actual structure and triggers.

The DB can be generated by running the above ddl script. Schema `ethercis` should exist.

Tables TERRITORY, LANGUAGE and CONCEPT should be populated from openEHR local terminology definition contained in `terminology.xml`. Script `populate-concept` is provided to perform this task (see [ethercis/examples/scripts](https://github.com/ethercis/ethercis/tree/master/examples))

###### Required PostgreSQL extensions

<html>
<body>
<table border="1" style="border-collapse:collapse">
<tr><td>plpgsql</td><td>1.0</td><td></td></tr>
<tr><td>jsquery</td><td>1.0</td><td>https://github.com/postgrespro/jsquery</td></tr>
<tr><td>ltree</td><td>1.0</td><td>https://www.postgresql.org/docs/9.4/static/ltree.html</td></tr>
<tr><td>temporal_tables</td><td>1.0.2</td><td>http://pgxn.org/dist/temporal_tables/</td></tr>
<tr><td>uuid-ossp</td><td>1.0</td><td>https://www.postgresql.org/docs/9.5/static/uuid-ossp.html</td></tr>
</table>
</body>
</html>

How To Build It?
----------------
- You need to compile each module as indicated in their respective README. A global setting for the assembly of *uber* jars should be done in your `settings.xml`. An example is given at ethercis/examples/
- The sample launch script (ecis-server) assumes some jar assemblies to simplify the classpath. At the moment, there is no assembly provided in the pom's.
- Locally, you should install the 'exotic' libraries required by Maven. These jars are located in directory 'libraries'. Local installation can be achieved with the following command for example:

		mvn org.apache.maven.plugins:maven-install-plugin:2.5.2:install-file  
		    -Dfile=/Development/Dropbox/eCIS_Development/eCIS-LIB/compositionTemplate.jar 
		    -DgroupId=org.openehr 
		    -DartifactId=org.openehr.openEHR.v1 
		    -Dversion=1.0.0 
		    -Dpackaging=jar 
		    -DlocalRepositoryPath=/Development/Dropbox/eCIS_Development/eCIS-LIB/local-maven-repo

- Gradle is now also supported

## How To Run It?

- Script `ecis-server` should be adapted to get the right classpath, path to required configuration, network parameters etc.
- Ditto for all configuration files.

The scripts and configuration samples are in directory `examples` 

Script `ecis-server` uses *uber* jars to keep the modularity of the platform as well as to ease the production of patches. The jars are posted at [libraries](https://github.com/ethercis/ethercis/tree/master/libraries) until a better file repository is identified.

Documentation And Examples
-

In this section you will find:

- [examples](https://github.com/ethercis/ethercis/tree/master/examples) scripts and configuration files to run ethercis on a Linux box. Scripts can be adapted to launch the server on  Windows if required.
- [libraries](https://github.com/ethercis/ethercis/tree/master/libraries) some pre-compiled libraries to make life a bit easier (mostly xml bindings classes and one to avoid conflicts with the patches from the core module
- [installation notes](https://github.com/ethercis/ethercis/tree/master/installation) documentation and readme's, mostly to install a system
- installation update in [Deploy n Scripts](https://github.com/ethercis/deploy-n-scripts)  
- [REST API](https://github.com/ethercis/ethercis/blob/master/doc/rest%20api.md) and [FLAT JSON](https://github.com/ethercis/ethercis/blob/master/doc/flat%20json.md)
- [Composition Serialization and Query](https://github.com/ethercis/ethercis/blob/master/doc/serialization.md)


## Product/Project Support
This product /project is supported by the Ripple Foundation, who aim to enhance the EtherCIS solution. 
We are working to fund as many of the enhancements of EtherCIS as we can based on projects that our non profit organisation supports.

We will try to fix any key bugs and documentation errors ourselves. Other issues, requests for enhancements or feature additions, will be added to the project backlog.

The Ripple Foundation is committed to offering free and open software, with quality, free and open documentation, but unfortunately is unable to offer free support for all issues/pull requests.

(Our latest thinking on the best model to support our open platform mission in healthcare may best be understood by reading this article. https://opensource.com/business/16/4/refactoring-open-source-business-models

If you would like to offer some of your energy/ suggest other ideas towards progressing an open platform in healthcare, please contact us at info@ripple.foundation )

If you need support with a particular issue/pull request, please let us know and we can consider a bounty source (https://www.bountysource.com/), or indeed a formal project/support arrangement to get particular issues/requirements reviewed/ addressed.

Thanks for your interest in EtherCIS

The Ripple Foundation
http://ripple.foundation/
 
