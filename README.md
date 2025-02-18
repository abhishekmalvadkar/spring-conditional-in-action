## Spring Custom Conditions - Real-World Scenarios

### Scenario 1: Enabling a Bean Based on Environment Variable

#### Use Case:
You want to register a bean **only if a specific environment variable (e.g., `ENABLE_FEATURE_X`) is set**.

#### Step 1: Create the Custom Condition
```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class EnvironmentVariableCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String envVar = System.getenv("ENABLE_FEATURE_X");
        return "true".equalsIgnoreCase(envVar); // Bean will load only if ENABLE_FEATURE_X=true
    }
}
```

#### Step 2: Create a Custom Annotation
```java
import org.springframework.context.annotation.Conditional;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(EnvironmentVariableCondition.class)
public @interface ConditionalOnEnvironmentVariable {
}
```

#### Step 3: Apply the Annotation to a Bean
```java
import org.springframework.stereotype.Service;

@Service
@ConditionalOnEnvironmentVariable
public class FeatureXService {
    public String getFeatureXMessage() {
        return "Feature X is enabled!";
    }
}
```

#### Step 4: Test It
Run the application **without setting the environment variable**, and the bean will not be created.  
Now, **set `ENABLE_FEATURE_X=true`** in your environment and restart your app to see it in action!

---

### Scenario 2: Enabling a Bean Based on Active Spring Profile

#### Use Case:
You want a service to be enabled only if a specific **Spring profile (e.g., `prod`) is active**.

#### Step 1: Create the Custom Condition
```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class ProfileCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        return env.acceptsProfiles("prod"); // Bean will be created only in 'prod' profile
    }
}
```

#### Step 2: Create a Custom Annotation
```java
import org.springframework.context.annotation.Conditional;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(ProfileCondition.class)
public @interface ConditionalOnProdProfile {
}
```

#### Step 3: Apply the Annotation to a Bean
```java
import org.springframework.stereotype.Service;

@Service
@ConditionalOnProdProfile
public class ProductionService {
    public String getProductionMessage() {
        return "Running in production mode!";
    }
}
```

#### Step 4: Test It
Run the app with different profiles:
```sh
# This won't load the bean
mvn spring-boot:run

# This will load the bean
mvn spring-boot:run -Dspring.profiles.active=prod
```

---

### Scenario 3: Load Different Database Configuration Based on DB Type

#### Use Case:
You want to **register different configurations** based on the **database type** (MySQL, PostgreSQL, H2).

#### Step 1: Create the Database Condition
```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class DatabaseTypeCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        String dbType = env.getProperty("database.type"); // Read from properties
        return dbType != null && dbType.equalsIgnoreCase("mysql");
    }
}
```

#### Step 2: Create a Custom Annotation
```java
import org.springframework.context.annotation.Conditional;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(DatabaseTypeCondition.class)
public @interface ConditionalOnMySQL {
}
```

#### Step 3: Define Separate Beans for Different DBs
```java
import org.springframework.stereotype.Service;

@Service
@ConditionalOnMySQL
public class MySQLDatabaseService {
    public String getDBType() {
        return "Using MySQL database";
    }
}
```

```java
import org.springframework.context.annotation.Conditional;
import org.springframework.stereotype.Service;

@Conditional(DatabaseTypeCondition.class)
@Service
public class H2DatabaseService {
    public String getDBType() {
        return "Using H2 database";
    }
}
```

#### Step 4: Set the Database Type in `application.properties`
```properties
database.type=mysql
```
or
```properties
database.type=h2
```
Restart the app and check which database configuration is used.

---

### Scenario 4: Enable a Bean Only on Weekends

#### Step 1: Create the Custom Condition
```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.type.AnnotatedTypeMetadata;
import java.time.DayOfWeek;
import java.time.LocalDate;

public class WeekendCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        DayOfWeek day = LocalDate.now().getDayOfWeek();
        return day == DayOfWeek.SATURDAY || day == DayOfWeek.SUNDAY;
    }
}
```

#### Step 2: Create a Custom Annotation
```java
import org.springframework.context.annotation.Conditional;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(WeekendCondition.class)
public @interface ConditionalOnWeekend {
}
```

#### Step 3: Apply the Annotation to a Bean
```java
import org.springframework.stereotype.Service;

@Service
@ConditionalOnWeekend
public class WeekendService {
    public String getWeekendMessage() {
        return "It's the weekend! Special weekend service is enabled.";
    }
}
```

#### Step 4: Test It
Run the application on a weekday, and the bean won't be created.  
Run it on a weekend (Saturday/Sunday), and it will be available.

---

## Summary
| **Scenario** | **Custom Annotation** | **Condition Logic** |
|-------------|----------------------|---------------------|
| Enable a bean based on an **environment variable** | `@ConditionalOnEnvironmentVariable` | Checks if an env variable is set |
| Enable a bean based on **Spring profile** | `@ConditionalOnProdProfile` | Checks if profile is `prod` |
| Enable different configurations based on **database type** | `@ConditionalOnMySQL` | Checks `database.type` property |
| Enable a bean only on **weekends** | `@ConditionalOnWeekend` | Checks if today is Saturday/Sunday |

---

