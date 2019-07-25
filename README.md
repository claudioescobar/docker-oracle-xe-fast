# Fast Oracle-XE 18 docker container for automated tests (fast startup with embedded data)
The oracle-xe docker container for local development has a slow starting time turning hard to use it with automated tests(using https://www.testcontainers.org/ for example). 

Testcontainers has no support for named volumns(https://github.com/testcontainers/testcontainers-java/issues/675) so I couldn't use an initialized database.

So the aproach was to initialize the database on image creation. The bad think on this is that we don't have volume control using docker volume, because the database state was inside the container. For me this was not a problem because my automated tests needs. 

I changed the scripts for oracle 18.4.0 located in OracleDatabase/SingleInstance/docker-files/18.4.0 (you can see the changes on the commit history).

Now lets start the process to generate the image and after that a code sample of how to run it with docker command, docker-compose configuration and TestContainers.

## Image generation
To generate the initialized image:

- put oracle-xe 18.4.0 rpm file(https://www.oracle.com/technetwork/database/database-technologies/express-edition/downloads/index.html) on 

      OracleDatabase/SingleInstance/docker-files/18.4.0
- execute the command below on directory OracleDatabase/SingleInstance/docker-files

      ./buildDockerImage.sh  -v 18.4.0 -x

Now use the command 
   
    docker image ls

and you should see something like

    REPOSITORY                                  TAG                 IMAGE ID            CREATED             SIZE
    oracle/database                             18.4.0-xe           d9fc386259e7        26 hours ago        13.3GB

Note that the image is greater than the original oracle because the volume is generated inside it.

## Notes before running the image
The connection:

    url: jdbc:oracle:thin:@localhost:1521/XE
    user: SYSTEM
    password: oracle

## Running the image on docker

    docker container run -p 1521:1521 -v <your local scripts path>:/opt/oracle/scripts/startup --name my_oracle oracle/database:18.4.0-xe

Note that we have a mapped volumn for 'opt/oracle/scripts/startup' directory because the 'opt/oracle/scripts/setup' is used when the volumn is created (the usual way) and with the custom scripts are executed just after the container initialization and will maintain the state during the container lifetime since we dont have external volumes.

## Running the image on docker-compose
Create your docker-compose.yaml file:

    version: '2'
    services:
    db:
        image: "oracle/database:18.4.0-xe"
        container_name: "my_oracle"
        ports:
        - "1521:1521"
        volumes:
        - <your local scripts path>:/opt/oracle/scripts/startup

Run it:

    docker-compose up

Note that the configuration file dont have the named volums created.

## Running with TestContainers
First create your Singleton instance to create the oracle container just one time:

    import org.testcontainers.containers.BindMode;
    import org.testcontainers.containers.OracleContainer;

    import java.io.IOException;

    public class SingletonOracleContainer extends OracleContainer {
        private static final String IMAGE_VERSION = "oracle/database:18.4.0-xe";
        private static SingletonOracleContainer container;

        private SingletonOracleContainer() {
            super(IMAGE_VERSION);
        }

        public static SingletonOracleContainer getInstance() {
            if (container == null) {
                container = new SingletonOracleContainer();
                container.withPassword("oracle");
                container.withClasspathResourceMapping("<your scripts resource directory>/", "/home/oracle", BindMode.READ_ONLY);
            }
            return container;
        }

        @Override
        public void start() {
            super.start();

            //the tests run before the startup scripts directory are executed so we need to run them inside the container via docker exec
            try {
                this.execInContainer("sqlplus", "system/oracle@//localhost:1521/XE", "@/home/oracle/your-script.sql");
            } catch (IOException e) {
                throw new RuntimeException("Error trying to run the initialization script", e);
            } catch (InterruptedException e) {
                throw new RuntimeException("Error trying to run the initialization script", e);
            }

            System.setProperty("DB_URL", container.getJdbcUrl());
            System.setProperty("DB_USERNAME", "system");
            System.setProperty("DB_PASSWORD", "oracle");
        }

        @Override
        public void stop() {
            //do nothing, JVM handles shut down
        }
    }

Sample of test class using junit 5 and spring test (probably you would need to delete and insert data before tests because the data state, to do that I used spring-test annotations):
@Testcontainers
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = YorTestSpringConfig.class)
@DatabaseTestData
public class YourRepositoryTest {

    @Container
    private static final OracleContainer ORACLE_CONTAINER = SingletonOracleContainer.getInstance();

    @Autowired
    YourRepository yourRepository;

    @Test
    void dbIsRunning() {
        assertTrue(ORACLE_CONTAINER.isRunning());
    }

    @Test
    void getById() {
        # your insert-data.sql scripts should have this id registered
        assertNotNull(yourRepository.getById(90400888000142l));
    }
}    

## Conclusion
With this image on my machine the container creation took around 30 seconds without custom scripts execution. With more adjustments we could reduce the image size and probably the container creation will have the time reduced as well.

### <b>*The base code was taken from official oracle docker images at https://github.com/oracle/docker-images/tree/master/OracleDatabase/SingleInstance<b>
