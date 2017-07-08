# DatabaseLib
## How-to
#### Installation
1. Put the .jar in the plugin folder
2. Start the server
3. Stop the server
4. Configure the pool.yml (and optionally the async.yml)
5. Start the server again
#### Usage
1. Create a new class (or use a preexisting one) and `extend  PluginSqlTaskSubmitter`.
2. Create a new instance of your class by either passing a `JavaPlugin` (Bukkit) or a
`Plugin` instance (BungeeCord).

Take a look at the examples below for a complete example of a Bukkit plugin.
## General information
#### Callbacks
When you create or submit tasks, you have to pass a non-null callback.
The type of the callback depends on whether the asynchronously executed method
returns something or not. If the method doesn't return anything (i.e. there is no
`return` statement), the type of the callback is `Consumer<Exception>`.
Otherwise, it is `BiConsumer<ReturnType, Exception>`.

Some examples:
- if there is no `return` statement, pass a `Consumer<Exception>`
- if you return an `Integer`, pass a `BiConsumer<Integer, Exception>`
- if you return a `String`, pass a `BiConsumer<String, Exception>`
- in general: if you return something of type `R`, pass a `BiConsumer<? super R, Exception>`

#### Asynchronous execution of tasks
All tasks that are submitted through one of the different `submit...` methods
are executed asynchronously in whichever thread the library chooses. After
a task completes (either normally or by throwing an exception), its
callback method is executed in the server thread (Bukkit plugins) or in one
of the plugin threads (Bungee plugins). If the task completes
exceptionally, the `Exception` passed to the callback is non-null.
#### Configuring tasks before submission
If you want to configure a task before submitting it (e.g. you want to
increase its priority), you first have to create it by calling one of the
`new...Task` methods, then use its setters to configure it and finally pass
it to the `submit` method for execution.
(see `creatingAndConfiguringTasksBeforeSubmission` method in the examples)

Note that by default all tasks have a `NORMAL` priority and all
statement tasks timeout after 5 seconds.
#### Closing Connections and Statements
All library supplied `Connection`s and `Statement`s are closed automatically,
so you don't have to call `close()` on them. However, you still need to close
all `Statement`s you created yourself (e.g. when using an `SqlConnectionTask`).
The only exception to this rule is when you use `getConnection()` (see next section).
#### Synchronous execution of queries
Sometimes you don't want to execute queries asynchronously. In these cases
you can get a `Connection` directly from the pool by calling `getConnection()`.

**You are responsible for closing the connection after usage.** If you forget
to do so, the pool will run out of `Connection`s, so it's best to use a try-with-resources
block wherever possible when acquiring them this way.

**Be aware that a call to `getConnection()` is blocking.** If no `Connection` is available
(e.g. because of the pool being empty), the thread in which this method
is called will be blocked.
#### Creating your own connection pools
If your application has to submit many long-running tasks or you have some other reason that
makes it necessary for you to manage your own set of `Connection`s, you can instantiate a
connection pool by using the static factory methods of the `SqlConnectionPools` class.
To use these methods you have to pass a `PoolConfig` instance which can be created by using a
`PoolConfig.Builder`. If you want your users to be able to manually configure your pool from
a file, you can use a `PoolConfiguration` to store the options.
## Examples
#### Complete Bukkit plugin example
```java
import de.exlll.asynclib.exec.TaskPriority;
import de.exlll.databaselib.submit.PluginSqlTaskSubmitter;
import de.exlll.databaselib.submit.SqlPreparedStatementTask;
import de.exlll.databaselib.submit.configure.PreparationStrategy;
import org.bukkit.plugin.java.JavaPlugin;

import java.sql.*;
import java.util.function.BiConsumer;
import java.util.function.Consumer;
import java.util.logging.Level;

public class ExamplePlugin extends JavaPlugin {

    // Bungee plugins have to extend BungeeSqlTaskSubmitter instead
    public static final class TestSubmitter extends PluginSqlTaskSubmitter {
        public TestSubmitter(JavaPlugin plugin) {
            super(plugin);
        }

        public void getUserNameById(int userId, BiConsumer<String, Exception> callback) {
            String query = "SELECT `user_name` FROM `users` WHERE `user_id` = ?";
            submitPreparedStatementTask(query, preparedStatement -> {
                preparedStatement.setInt(1, userId);
                ResultSet rs = preparedStatement.executeQuery();

                return rs.next() ? rs.getString("user_name") : "";
            }, callback);
        }

        public void insertUser(String userName, Consumer<Exception> callback) {
            String query = "INSERT INTO `users`(user_name) VALUES (?)";

            SqlPreparedStatementTask<Void> task =
                    newPreparedStatementTask(query, preparedStatement -> {
                        preparedStatement.setString(1, userName);
                        preparedStatement.execute();
                    }, callback);

            int returnKeys = Statement.RETURN_GENERATED_KEYS;
            PreparationStrategy<PreparedStatement> strategy =
                    PreparationStrategy.withAutoGeneratedKeys(returnKeys);
            task.setPriority(TaskPriority.HIGH)
                    .setPreparationStrategy(strategy);
            submit(task);
        }

        public void createTable() {
            // get Connection directly
            try (Connection connection = getConnection();
                 Statement stmt = connection.createStatement()) {

                stmt.execute("CREATE TABLE IF NOT EXISTS `users` (" +
                        "`user_id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY," +
                        "`user_name` VARCHAR(32))");
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public void onEnable() {
        TestSubmitter submitter = new TestSubmitter(this);
        submitter.createTable(); // this call is synchronous!

        submitter.insertUser("User1", exception -> {
            if (exception != null) {
                getLogger().log(Level.SEVERE, "failed to insert user", exception);
            }
        });

        int userId = 1;
        submitter.getUserNameById(userId, (userName, exception) -> {
            if (exception != null) {
                getLogger().log(Level.SEVERE, "failed to get user by id", exception);
            } else {
                String msg = String.format("user with id %d has name %s", userId, userName);
                getLogger().info(msg);
            }
        });
    }
}
```

#### Ways to create, configure and submit tasks
```java
import de.exlll.asynclib.exec.TaskPriority;
import de.exlll.databaselib.submit.*;
import de.exlll.databaselib.submit.configure.PreparationStrategy;
import org.bukkit.plugin.java.JavaPlugin;

import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.function.BiConsumer;
import java.util.function.Consumer;
import java.util.logging.Level;

final class ExampleSubmitter extends PluginSqlTaskSubmitter {
    public ExampleSubmitter(JavaPlugin plugin) {
        super(plugin);
    }

    Consumer<Exception> exceptionCallback = exception ->
            pluginInfo.getLogger().log(Level.SEVERE, exception.getMessage(), exception);

    BiConsumer<Object, Exception> resultExceptionCallback = (result, exception) -> {
        if (exception != null) {
            pluginInfo.getLogger().log(Level.SEVERE, exception.getMessage(), exception);
        } else {
            pluginInfo.getLogger().info("result: " + result);
        }
    };

    public void submittingTasksWhichDoNotReturnResults() {
        submitConnectionTask(connection -> {
            // ...do something with the connection
        }, exceptionCallback);

        submitStatementTask(statement -> {
            // ...do something with the statement
        }, exceptionCallback);

        String query = "UPDATE ...";
        submitPreparedStatementTask(query, preparedStatement -> {
            // ...do something with the preparedStatement
        }, exceptionCallback);

        String call = "{call ... }";
        submitCallableStatementTask(call, callableStatement -> {
            // ...do something with the callableStatement
        }, exceptionCallback);
    }

    public void submittingTasksWhichReturnResults() {
        submitConnectionTask(connection -> {
            // ...do something with the connection
            return "1";
        }, resultExceptionCallback);

        submitStatementTask(statement -> {
            // ...do something with the statement
            return "2";
        }, resultExceptionCallback);

        String query = "SELECT * FROM ...";
        submitPreparedStatementTask(query, preparedStatement -> {
            // ...do something with the preparedStatement
            return "3";
        }, resultExceptionCallback);

        String call = "{call ... }";
        submitCallableStatementTask(call, callableStatement -> {
            // ...do something with the callableStatement
            return "4";
        }, resultExceptionCallback);
    }

    public void creatingAndConfiguringTasksBeforeSubmission() {
        // #####################
        // ### SqlConnectionTask
        // #####################

        // SqlConnectionTask that doesn't return a result
        SqlConnectionTask<Void> voidConnectionTask = newConnectionTask(connection -> {
            // ...do something with the connection
        }, exceptionCallback);

        voidConnectionTask.setPriority(TaskPriority.LOW);
        submit(voidConnectionTask);


        // SqlConnectionTask that returns a result
        SqlConnectionTask<String> resultConnectionTask = newConnectionTask(connection -> {
            // ...do something with the connection
            return "1";
        }, resultExceptionCallback);

        resultConnectionTask.setPriority(TaskPriority.HIGH);
        submit(resultConnectionTask);


        // ####################
        // ### SqlStatementTask
        // ####################

        // SqlStatementTask that doesn't return a result
        SqlStatementTask<Void> voidStatementTask = newStatementTask(statement -> {
            // ...do something with the statement
        }, exceptionCallback);

        voidStatementTask.setPriority(TaskPriority.LOW);
        submit(voidStatementTask);


        // SqlStatementTask that returns a result
        SqlStatementTask<String> resultStatementTask = newStatementTask(statement -> {
            // ...do something with the statement
            return "2";
        }, resultExceptionCallback);

        resultStatementTask.setPriority(TaskPriority.HIGH);
        submit(resultStatementTask);


        // ############################
        // ### SqlPreparedStatementTask
        // ############################

        // SqlPreparedStatementTask that doesn't return a result
        SqlPreparedStatementTask<Void> voidPreparedStatementTask =
                newPreparedStatementTask("UPDATE ...", preparedStatement -> {
                    // ...do something with the preparedStatement
                }, exceptionCallback);

        voidPreparedStatementTask
                .setPreparationStrategy(PreparationStrategy.PREPARED_STATEMENT_DEFAULT)
                .setPriority(TaskPriority.LOW);
        submit(voidPreparedStatementTask);


        // SqlPreparedStatementTask that returns a result
        SqlPreparedStatementTask<String> resultPreparedStatementTask =
                newPreparedStatementTask("SELECT ...", preparedStatement -> {
                    // ...do something with the preparedStatement
                    return "3";
                }, resultExceptionCallback);

        resultPreparedStatementTask
                .setPreparationStrategy(
                        PreparationStrategy.withAutoGeneratedKeys(Statement.RETURN_GENERATED_KEYS)
                )
                .setPriority(TaskPriority.HIGH);
        submit(resultPreparedStatementTask);


        // ############################
        // ### SqlCallableStatementTask
        // ############################

        // SqlCallableStatementTask that doesn't return a result
        SqlCallableStatementTask<Void> voidCallableStatementTask =
                newCallableStatementTask("{call ...}", callableStatement -> {
                    // ...do something with the callableStatement
                }, exceptionCallback);

        voidCallableStatementTask
                .setPreparationStrategy(PreparationStrategy.CALLABLE_STATEMENT_DEFAULT)
                .setPriority(TaskPriority.LOW);
        submit(voidCallableStatementTask);


        // SqlCallableStatementTask that returns a result
        SqlCallableStatementTask<String> resultCallableStatementTask =
                newCallableStatementTask("{call ...}", callableStatement -> {
                    // ...do something with the callableStatement
                    return "4";
                }, resultExceptionCallback);

        resultCallableStatementTask
                .setPreparationStrategy(PreparationStrategy.CALLABLE_STATEMENT_DEFAULT)
                .setPriority(TaskPriority.HIGH);
        submit(resultCallableStatementTask);
    }

    public void usingConnectionsDirectly() {
        try (Connection connection = getConnection()) {
            // ...do something with the connection
        } catch (SQLException e) {
            pluginInfo.getLogger().log(Level.SEVERE, e.getMessage(), e);
        }
    }
}
```
## Import
#### Maven
```xml
<repository>
    <id>de.exlll</id>
    <url>https://repo.exlll.de/artifactory/releases/</url>
</repository>

<!-- for Bukkit plugins -->
<dependency>
    <groupId>de.exlll</groupId>
    <artifactId>databaselib-bukkit</artifactId>
    <version>2.0.0</version>
</dependency>

<!-- for Bungee plugins -->
<dependency>
    <groupId>de.exlll</groupId>
    <artifactId>databaselib-bungee</artifactId>
    <version>2.0.0</version>
</dependency>
```
#### Gradle
```groovy
repositories {
    maven {
        url 'https://repo.exlll.de/artifactory/releases/'
    }
}
dependencies {
    // for Bukkit plugins
    compile group: 'de.exlll', name: 'databaselib-bukkit', version: '2.0.0'

    // for Bungee plugins
    compile group: 'de.exlll', name: 'databaselib-bungee', version: '2.0.0'
}
```
Additionally, you either have to import the Bukkit or BungeeCord API
or disable transitive lookups. This project uses both of these APIs, so if you
need an example of how to import them with Gradle, take a look at the `build.gradle`.

If, for some reason, you have SSL errors that you're unable to resolve, you can
use `http://exlll.de:8081/artifactory/releases/` as the repository instead.
