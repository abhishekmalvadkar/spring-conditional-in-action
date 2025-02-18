## Real-World Example: Conditional Email Sender (SMTP vs SendGrid)

### **Scenario:**
We configure **JavaMailSender (SMTP)** and **SendGrid** based on a property (`mail.provider`). If the property is `smtp`, JavaMailSender is used; if it's `sendgrid`, SendGrid is used. If `mail.provider` is not set, **SMTP is used as the default**.

---

### **Step 1: Create the Email Service Interface**

```java
public interface EmailService {
    String sendMail(String to, String subject, String body);
}
```

---

### **Step 2: Create the Custom Condition**

```java
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

public class MailProviderCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();
        String provider = env.getProperty("mail.provider", "smtp"); // Default to SMTP
        String expectedProvider = (String) metadata.getAnnotationAttributes(ConditionalOnMailProvider.class.getName()).get("value");
        return expectedProvider.equalsIgnoreCase(provider);
    }
}
```

---

### **Step 3: Create a Custom Annotation**

```java
import org.springframework.context.annotation.Conditional;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Conditional(MailProviderCondition.class)
public @interface ConditionalOnMailProvider {
    String value(); // Takes the expected provider value ("smtp" or "sendgrid")
}
```

---

### **Step 4: Implement JavaMailSender Service (SMTP)**

```java
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.JavaMailSenderImpl;
import org.springframework.stereotype.Service;
import java.util.Properties;

@Service
@ConditionalOnMailProvider("smtp")
public class SmtpMailService implements EmailService {
    
    private final JavaMailSender mailSender;

    public SmtpMailService() {
        this.mailSender = configureJavaMailSender();
    }

    private JavaMailSender configureJavaMailSender() {
        JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
        mailSender.setHost("smtp.gmail.com");
        mailSender.setPort(587);
        mailSender.setUsername("your-email@gmail.com");
        mailSender.setPassword("your-password");

        Properties props = mailSender.getJavaMailProperties();
        props.put("mail.transport.protocol", "smtp");
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");

        return mailSender;
    }

    @Override
    public String sendMail(String to, String subject, String body) {
        return "Email sent via SMTP to " + to;
    }
}
```

---

### **Step 5: Implement SendGrid Mail Service**

```java
import com.sendgrid.Method;
import com.sendgrid.Request;
import com.sendgrid.Response;
import com.sendgrid.SendGrid;
import com.sendgrid.Mail;
import com.sendgrid.Email;
import com.sendgrid.Content;
import org.springframework.stereotype.Service;
import java.io.IOException;

@Service
@ConditionalOnMailProvider("sendgrid")
public class SendGridMailService implements EmailService {
    
    private final SendGrid sendGrid;

    public SendGridMailService() {
        this.sendGrid = new SendGrid("YOUR_SENDGRID_API_KEY");
    }

    @Override
    public String sendMail(String to, String subject, String body) {
        Email from = new Email("your-email@example.com");
        Email recipient = new Email(to);
        Content content = new Content("text/plain", body);
        Mail mail = new Mail(from, subject, recipient, content);

        Request request = new Request();
        try {
            request.setMethod(Method.POST);
            request.setEndpoint("mail/send");
            request.setBody(mail.build());
            Response response = sendGrid.api(request);
            return "Email sent via SendGrid to " + to + " with status: " + response.getStatusCode();
        } catch (IOException e) {
            return "Failed to send email via SendGrid: " + e.getMessage();
        }
    }
}
```

---

### **Step 6: Expose an API to Test the Mail Services**

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/mail")
public class MailController {
    
    private final EmailService emailService;

    public MailController(EmailService emailService) {
        this.emailService = emailService;
    }

    @PostMapping("/send")
    public String sendMail(@RequestParam String to, @RequestParam String subject, @RequestParam String body) {
        return emailService.sendMail(to, subject, body);
    }
}
```

---

### **Step 7: Configure in `application.properties`**

To use **SMTP** (default):
```properties
# mail.provider not set (SMTP will be picked automatically)
```

To explicitly use **SendGrid**:
```properties
mail.provider=sendgrid
```

---

### **Step 8: Test the API**

#### **Test with Default SMTP (when `mail.provider` is not set)**
```sh
curl -X POST "http://localhost:8080/mail/send" -d "to=test@example.com" -d "subject=Hello" -d "body=This is a test email"
```
**Expected Output:**  
`"Email sent via SMTP to test@example.com"`

#### **Test with SendGrid**
Change `mail.provider=sendgrid` in `application.properties` and restart the application.
```sh
curl -X POST "http://localhost:8080/mail/send" -d "to=test@example.com" -d "subject=Hello" -d "body=This is a test email"
```
**Expected Output:**  
`"Email sent via SendGrid to test@example.com with status: 202"`

---

### **Daily-Life Use Case**
- **Corporate Email Services**: Automatically switch between SMTP and SendGrid depending on availability.
- **Multi-tenant Applications**: Different customers use different email providers.
- **Failover System**: If one email provider is down, you can switch to another.
- **Default Behavior**: If no provider is explicitly configured, SMTP is used.

This example demonstrates **real-world Spring Conditional Annotations** to dynamically select an **email provider** based on an interface-driven design with a default fallback mechanism. 🚀
