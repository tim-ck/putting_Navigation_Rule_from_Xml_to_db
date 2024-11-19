# Combining XML and Database Navigation Rules

This guide provides a step-by-step approach to integrate database-stored navigation rules with existing XML-based configurations in a Java application.

## Step-by-Step Guide

### 1. Database Setup
Create a table to store navigation rules in your database.
```sql
CREATE TABLE navigation_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    from_view_id VARCHAR(255),
    to_view_id VARCHAR(255),
    condition VARCHAR(255)
);
```

### 2. Create Entity Class
Define an entity to map the `navigation_rules` table.
```java
@Entity
@Table(name = "navigation_rules")
public class NavigationRule {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String fromViewId;
    private String toViewId;
    private String condition;

    // Getters and setters
}
```

### 3. Create Repository
Create a repository interface to access the navigation rules.
```java
public interface NavigationRuleRepository extends JpaRepository<NavigationRule, Long> {
    List<NavigationRule> findByFromViewId(String fromViewId);
}
```

### 4. Create Service
Create a service to fetch navigation rules from the database.
```java
@Service
public class NavigationRuleService {
    @Autowired
    private NavigationRuleRepository repository;

    public List<NavigationRule> getNavigationRules(String fromViewId, Strign outcome) {
        NavigationRule criteria = new NavigationRule();
        //set criteria...
        return repository.firstResultByCriteria(criteria);
    }
}
```

### 5. Custom Navigation Handler
Extend `ConfigurableNavigationHandler` to create a custom navigation handler.
```java
@ManagedBean
public class CustomNavigationHandler extends ConfigurableNavigationHandler {
    
    private ConfigurableNavigationHandler parent;
    private NavigationRuleService navigationRuleService;

    private ConfigurableNavigationHandler parent;

    public CustomNavigationHandler(ConfigurableNavigationHandler parent) {
        this.parent = parent;
    }

    @Override
    public NavigationCase getNavigationCase(FacesContext context, String fromAction, String outcome) {
        NavigationCase navCase = parent.getNavigationCase(context, fromAction, outcome);
        if (navCase != null) {
            return navCase;
        }
        // If no result found in local XML, check the database
        if (navigationRuleService == null) {
            navigationRuleService = FacesContextUtils.getWebApplicationContext(context).getBean(NavigationRuleService.class);
        }
        
        String fromViewId = context.getViewRoot().getViewId();
        NavigationRule rules = navigationRuleService.getNavigationRules(fromViewId, outcome);

        for (NavigationRule rule : rules) {
            if (rule.getCondition().equals(outcome)) {
                return new NavigationCase(fromViewId, fromAction, outcome, null, rule.getToViewId(), null, false);
            }
        }
        return null;
    }
    @Override
    public Map<String, Set<NavigationCase>> getNavigationCases() {
        return parent.getNavigationCases();
    }
    @Override
    public void handleNavigation(FacesContext context, String fromAction, String outcome) {
        parent.handleNavigation(context, fromAction, outcome);

        // Check if navigation was handled by the parent
        if (!context.getResponseComplete()) {
            // If no result found in local XML, check the database
            if (tmpDynamicPageService == null) {
                tmpDynamicPageService = FacesContextUtils.getWebApplicationContext(context).getBean(TmpDynamicPageService.class);
            }

            String fromViewId = context.getViewRoot().getViewId();
            TmpSenNavigationRule rule = getNavigationCase(context, fromViewId, outcome);
            if(rule!= null) {
            	ViewControllerUtil.setByPassNavigationValidationFlag(true);
            	parent.handleNavigation(context, fromAction, rule.getToViewId());
            }
        }
    }
}
```

### 6. Register Custom Navigation Handler
Register your custom navigation handler in `faces-config.xml`.
```xml
<application>
    <navigation-handler>com.example.CustomNavigationHandler</navigation-handler>
    <managed-property>
        <property-name>facesConfigFiles</property-name>
        <list-entries>
            <value-class>java.lang.String</value-class>
            <value>/WEB-INF/external-navigation.xml</value>
            <value>/WEB-INF/internal-navigation.xml</value>
        </list-entries>
    </managed-property>
</application>
```

### 7. Bean Configuration
Ensure your managed beans can interact with the custom navigation handler.
```java
@Configuration
public class AppConfig {

    @Bean
    public NavigationHandler customNavigationHandler() {
        return new CustomNavigationHandler(FacesContext.getCurrentInstance().getApplication().getNavigationHandler());
    }
}
```

### 8. Example Managed Bean
Create a managed bean that returns the navigation outcome condition.
```java
@ManagedBean
@RequestScoped
public class MyBean {
    @Autowired
    private NavigationRuleService navigationRuleService;

    public String navigate() {
        String currentViewId = FacesContext.getCurrentInstance().getViewRoot().getViewId();
        String outcome = determineOutcome(); // Your logic to determine the outcome
        NavigationRule rule = navigationRuleService.findByFromViewIdAndCondition(currentViewId, outcome);
        return rule != null ? rule.getToViewId() : "defaultOutcome";
    }

    private String determineOutcome() {
        // Example logic to determine outcome
        return "someCondition";
    }
}
```

