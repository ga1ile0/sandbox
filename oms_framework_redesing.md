# Playwright + JUnit5 Test Framework Redesign

## Purpose of this document

This document is a design specification for rewriting an existing Selenium/TestNG UI
automation framework into a Playwright (Java) + JUnit5 framework. It is intended to be
handed to an engineer (human or AI agent) who has access to the current framework
codebase and is expected to **migrate/rewrite it** according to the architecture
described here.

The document explains:
1. What problems exist in the current framework
2. What architecture replaces it, and why
3. Concrete, ready-to-adapt Java code for every component
4. How the pieces compose together
5. Migration guidance and edge cases

---

## 1. Background — problems with the current framework

The current framework (TestNG-based, inherited codebase) has the following structural
problems that this redesign fixes:

1. **Static shared browser/session state.** A `BrowserContext` (or similar) is stored as
   a `static` field, shared across all threads. When tests run in parallel, one thread
   can clear cookies/cache (relogin) while another thread is mid-test using the same
   context. This causes flaky, order-dependent failures and makes parallelization
   unsafe.
2. **Deep, rigid inheritance for login strategy.** There is one `BaseTest` class,
   extended by several "Login*" classes (e.g. `LoginBeforeEachTest`, `LoginOnce`), each
   hardcoding a different relogin cadence. Test classes extend whichever `Login*` class
   matches the behavior they want. This means:
   - Login cadence and page-object access are entangled in a single inheritance chain.
   - You can only ever have *one* axis of variation per test class (single inheritance),
     so combining "these specific pages" with "this specific login cadence" forces a
     combinatorial explosion of subclasses as the app grows.
3. **God-object `BaseTest`.** `BaseTest` eagerly instantiates and holds a field for
   *every single Page Object in the entire application*, whether or not a given test
   class uses it. This is slow, hides real dependencies (you can't tell what a test
   actually touches by reading its class), and grows without bound as the app grows.
4. **Ad-hoc environment branching inside test bodies.** The application under test has
   two deployment instances — "internal" and "SaaS" — that are almost identical but
   differ in small ways (an extra field here, a different validation rule there). The
   current code handles this with inline conditionals scattered through test methods:
   ```java
   public void someTest() {
       if (instance.equals("internal")) {
           doSomething();
       }
       doSomethingElse();
   }
   ```
   This is unreadable, easy to get wrong, untraceable (you can't find all
   instance-specific behavior in one place), and duplicated across many tests.
5. **Shared test accounts with no concurrency control.** The application under test logs
   a user out if the same account logs in from elsewhere. With parallel test execution
   and shared static login state, two threads can log in as the same user concurrently,
   causing one thread's session to be silently invalidated by the other.

---

## 2. High-level architecture

The new framework is built on **composition over inheritance**, using JUnit5's
extension model (`Extension`, `ParameterResolver`, `BeforeEachCallback`,
`BeforeAllCallback`, etc.) as the composition mechanism, instead of a deep class
hierarchy.

There are **four independent, orthogonal concerns**, each solved by its own small
component. None of them know about each other; they are combined per test class simply
by attaching multiple JUnit5 extensions (or a meta-annotation that bundles a few
together). This is the core structural change from the old framework:

| Concern | Old approach | New approach |
|---|---|---|
| Browser/page lifecycle | Static fields, shared across threads | Per-test scoped, via Playwright's `@UsePlaywright` or a custom extension |
| Login cadence (relogin every test vs. once per class) | Subclass chain (`LoginBeforeEachTest`, `LoginOnce`, ...) | Small JUnit5 extensions + meta-annotations (`@ReloginPerTest`, `@ReloginPerClass`) |
| Which Page Objects a test needs | `BaseTest` holds one instance of every page | Generic `ParameterResolver` injects only the page types the test method declares as parameters |
| App instance differences (internal vs. SaaS) | `if (instance.equals(...))` inline in tests | Small class hierarchy *at the Page Object level only*, isolated to the handful of pages that actually differ |
| Shared test accounts | Static reused login, no coordination | Per-role account pool with acquire/release leasing |

Each of these is described in detail below, with code.

---

## 3. Browser & context lifecycle — no static state, per-test isolation

### Problem being solved
Static `Browser`/`BrowserContext`/`Page` fields are shared mutable state across threads.
Playwright's own model assumes a `BrowserContext` (cookies, storage, cache) is
disposable and scoped to a single logical session — reusing one across concurrent tests
is inherently unsafe.

### Design
- One `Playwright` + `Browser` instance **per thread** (Playwright driver connections are
  not meant to be shared across threads), held in a `ThreadLocal`.
- One fresh `BrowserContext` + `Page` **per test** (or per test class, if using the
  "relogin once" strategy — see section 5), created and torn down by extensions, never
  stored as a `static` field.

### Code

```java
public class BrowserFactory {
    private static final ThreadLocal<Playwright> PLAYWRIGHT =
        ThreadLocal.withInitial(Playwright::create);

    private static final ThreadLocal<Browser> BROWSER =
        ThreadLocal.withInitial(() ->
            PLAYWRIGHT.get().chromium().launch(
                new BrowserType.LaunchOptions().setHeadless(true)));

    public static Browser getBrowser() {
        return BROWSER.get();
    }

    public static void closeThreadResources() {
        BROWSER.get().close();
        PLAYWRIGHT.get().close();
        BROWSER.remove();
        PLAYWRIGHT.remove();
    }
}
```

If you adopt Playwright's native JUnit5 integration (`com.microsoft.playwright.junit`),
you can instead implement `OptionsFactory` to control launch options, and let
`@UsePlaywright` manage the `Playwright`/`Browser`/`BrowserContext`/`Page` lifecycle for
you (it already does per-test scoping correctly). The custom `BrowserFactory` above is
useful if you need more control (e.g. custom context creation tied to `storageState`,
described next) than the built-in integration allows.

### Parallel execution config

**JUnit5** (`junit-platform.properties` on the classpath):
```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.config.strategy=dynamic
```

**TestNG** (if you end up keeping TestNG instead — see section 9): use
`parallel="methods"` or `parallel="classes"` with `thread-count` in `testng.xml`, and
rely on the `ThreadLocal` browser factory above, since TestNG has no first-class
Playwright resource management.

---

## 4. Authentication — `storageState`, not "relogin by clearing cookies"

### Problem being solved
The old framework's "relogin" strategies work by re-running the login UI flow and/or
clearing cookies/cache on a *shared* context. This is slow (full UI login every time)
and, combined with static shared state, unsafe under parallelism.

### Design
Playwright supports capturing an authenticated session as a JSON file
(`storageState`) and seeding a *brand-new* `BrowserContext` with it. This means:
- The actual login UI flow runs once (or rarely, e.g. when the state file is missing/
  stale).
- Every test/class gets its own independent `BrowserContext` pre-loaded with that
  session — no shared object, no race condition, and no need to clear/relogin a shared
  context.

### Code

```java
public class LoginHelper {

    /** Performs the real UI login flow and persists the resulting session to disk. */
    public static Path performLoginAndSaveState(Browser browser, TestUser user) {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();
        page.navigate(user.getLoginUrl());
        page.fill("#username", user.getUsername());
        page.fill("#password", user.getPassword());
        page.click("#submit");
        page.waitForURL("**/dashboard");

        Path stateFile = Paths.get("auth/" + user.getId() + ".json");
        context.storageState(new BrowserContext.StorageStateOptions().setPath(stateFile));
        context.close();
        return stateFile;
    }

    /** Creates a new context/page seeded with a previously saved session. */
    public static BrowserContext createAuthenticatedContext(Browser browser, TestUser user) {
        Path stateFile = Paths.get("auth/" + user.getId() + ".json");
        if (!Files.exists(stateFile)) {
            performLoginAndSaveState(browser, user);
        }
        return browser.newContext(
            new Browser.NewContextOptions().setStorageStatePath(stateFile));
    }
}
```

Notes:
- If a role/user needs re-authentication periodically (token expiry, etc.), simply
  delete/regenerate the state file on a schedule, or check its age before reuse.
- If different roles/users need different starting states, store one file per user
  (`auth/admin.json`, `auth/standard-user.json`, ...) and select the file based on which
  user the test pool hands out (see section 6).

---

## 5. Login cadence — composable JUnit5 extensions, not subclassing

### Problem being solved
The old framework encodes "relogin before every test" vs. "relogin once per class" as
two different classes in an inheritance chain (`LoginBeforeEachTest`, `LoginOnce`),
which a test class must extend. Because Java only allows single inheritance, this
locks the test class into one specific combination of "how do I log in" and "what pages
do I have access to" and cannot be mixed independently.

### Design
Model each login cadence as its own **JUnit5 extension** (a plain class implementing a
lifecycle callback interface). A test class attaches the extension(s) it wants via
`@ExtendWith`, or via a **meta-annotation** that bundles the extension with the page
resolver (section 6) for a clean one-line declaration on the test class.

Because these are independent plug-ins (not a class hierarchy), any test class can use
any login cadence without needing a new subclass to exist for that combination.

### Code

```java
/** Key used to pass the current Playwright Page between extensions (must be shared). */
public final class TestContextKeys {
    public static final ExtensionContext.Namespace NAMESPACE =
        ExtensionContext.Namespace.create("playwright-framework");
    public static final String PAGE_KEY = "page";
    public static final String USER_KEY = "user";
    public static final String CONTEXT_KEY = "browserContext";
}

/** Relogs in before every single test method. Use for tests that must not share any
 *  session/browser state with other tests (e.g. tests that intentionally mutate
 *  account-level state). */
public class ReloginBeforeEachExtension implements BeforeEachCallback, AfterEachCallback {

    @Override
    public void beforeEach(ExtensionContext ctx) {
        TestUser user = UserPools.forCurrentRole().acquire();
        BrowserContext browserContext =
            LoginHelper.createAuthenticatedContext(BrowserFactory.getBrowser(), user);
        Page page = browserContext.newPage();

        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.USER_KEY, user);
        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.CONTEXT_KEY, browserContext);
        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.PAGE_KEY, page);
    }

    @Override
    public void afterEach(ExtensionContext ctx) {
        BrowserContext browserContext =
            ctx.getStore(TestContextKeys.NAMESPACE).get(TestContextKeys.CONTEXT_KEY, BrowserContext.class);
        browserContext.close();

        TestUser user = ctx.getStore(TestContextKeys.NAMESPACE).get(TestContextKeys.USER_KEY, TestUser.class);
        UserPools.forCurrentRole().release(user);
    }
}

/** Logs in once before all tests in the class, and reuses that session/context for
 *  every test method in the class. Use for read-heavy or workflow tests that don't
 *  need a clean session per test. */
public class ReloginOncePerClassExtension implements BeforeAllCallback, AfterAllCallback {

    @Override
    public void beforeAll(ExtensionContext ctx) {
        TestUser user = UserPools.forCurrentRole().acquire();
        BrowserContext browserContext =
            LoginHelper.createAuthenticatedContext(BrowserFactory.getBrowser(), user);
        Page page = browserContext.newPage();

        // Use the root context store scoped to the class, so it's visible to all
        // @BeforeEach/@Test invocations within the class, and cleaned up in afterAll.
        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.USER_KEY, user);
        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.CONTEXT_KEY, browserContext);
        ctx.getStore(TestContextKeys.NAMESPACE).put(TestContextKeys.PAGE_KEY, page);
    }

    @Override
    public void afterAll(ExtensionContext ctx) {
        BrowserContext browserContext =
            ctx.getStore(TestContextKeys.NAMESPACE).get(TestContextKeys.CONTEXT_KEY, BrowserContext.class);
        browserContext.close();

        TestUser user = ctx.getStore(TestContextKeys.NAMESPACE).get(TestContextKeys.USER_KEY, TestUser.class);
        UserPools.forCurrentRole().release(user);
    }
}
```

### Meta-annotations (clean call-site syntax)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith({PageObjectResolver.class, ReloginBeforeEachExtension.class})
public @interface ReloginPerTest {}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith({PageObjectResolver.class, ReloginOncePerClassExtension.class})
public @interface ReloginPerClass {}
```

### Usage

```java
@ReloginPerTest
public class FirstFormValidationTest {
    @Test
    void requiredFieldShowsError(FirstFormPage form) {
        form.submitEmpty();
        assertThat(form.getErrorMessage()).isEqualTo("Field required");
    }
}

@ReloginPerClass
public class FirstFormWorkflowTest {
    @Test
    void step1FillsBasicInfo(FirstFormPage form) {
        form.fillStep1();
    }

    @Test
    void step2FillsAdvancedInfo(FirstFormPage form) {
        form.fillStep2();
    }
}
```

Both test classes use the exact same `FirstFormPage` Page Object (no duplication), but
independently choose their login cadence via a single annotation. Adding a third
strategy later (e.g. `@ReloginPerSuite`) means writing one new extension + annotation —
**zero changes** to any existing test class.

### Ordering guarantee
JUnit5 runs `@ExtendWith` extensions in declaration order for lifecycle callbacks, and
parameter resolution (section 6) happens immediately before test method invocation,
after `beforeEach`/`beforeAll` have run. As long as the login extension is a
`BeforeEachCallback`/`BeforeAllCallback` (not itself trying to be a `ParameterResolver`),
the Playwright `Page` will already be stored and available by the time
`PageObjectResolver` needs it.

---

## 6. Page Object injection — generic `ParameterResolver`, replacing `BaseTest`

### Problem being solved
`BaseTest` holds a field for every Page Object class in the app, instantiated eagerly
regardless of whether a given test needs it. This wastes setup time, obscures real
dependencies, and doesn't scale as the app (and its page count) grows. It also forces
every test class into one giant inheritance chain just to get access to page objects.

### Design
A single, generic `ParameterResolver` that:
1. Recognizes any parameter whose type is a Page Object (e.g. implements/extends a
   marker interface or base class `BasePage`).
2. Reads the current Playwright `Page` from the extension store (populated by whichever
   login extension ran for this test/class — section 5).
3. Instantiates the requested Page Object type on demand, generically, via reflection —
   no per-page, no per-package factory code required.
4. Caches the instance per test invocation so multiple parameters of the same page type
   resolve to the same object within one test.

This works for the **entire application** — there is no need for one factory class per
package. Test classes simply declare, as method parameters, exactly the pages they use.

### Marker type

```java
/** Marker interface implemented by all Page Object classes so the resolver can
 *  recognize them. */
public interface BasePage {
}
```

### Resolver

```java
public class PageObjectResolver implements ParameterResolver {

    @Override
    public boolean supportsParameter(ParameterContext paramCtx, ExtensionContext extCtx) {
        return BasePage.class.isAssignableFrom(paramCtx.getParameter().getType());
    }

    @Override
    public Object resolveParameter(ParameterContext paramCtx, ExtensionContext extCtx) {
        Class<?> type = paramCtx.getParameter().getType();
        Page page = extCtx.getStore(TestContextKeys.NAMESPACE)
                           .get(TestContextKeys.PAGE_KEY, Page.class);

        if (page == null) {
            throw new IllegalStateException(
                "No Playwright Page available in the test context. Did you forget to " +
                "attach a login/lifecycle extension (e.g. @ReloginPerTest) to this test class?");
        }

        // Cache per-test so requesting the same page type twice returns the same instance.
        return extCtx.getStore(TestContextKeys.NAMESPACE)
                      .getOrComputeIfAbsent(type, t -> instantiate((Class<?>) t, page));
    }

    private Object instantiate(Class<?> type, Page page) {
        try {
            Constructor<?> constructor = type.getConstructor(Page.class);
            return constructor.newInstance(page);
        } catch (Exception e) {
            throw new RuntimeException("Failed to instantiate page object: " + type.getName(), e);
        }
    }
}
```

If you would rather avoid reflection entirely, replace `instantiate(...)` with an
explicit registry populated once at startup:

```java
public class PageRegistry {
    private static final Map<Class<?>, Function<Page, ?>> FACTORIES = new HashMap<>();

    static {
        FACTORIES.put(LoginPage.class, LoginPage::new);
        FACTORIES.put(FirstFormPage.class, FirstFormPage::new);
        // one line per page class that needs to be resolvable ...
    }

    @SuppressWarnings("unchecked")
    public static <T> T create(Class<T> type, Page page) {
        Function<Page, ?> factory = FACTORIES.get(type);
        if (factory == null) {
            throw new IllegalStateException("No factory registered for: " + type.getName());
        }
        return (T) factory.apply(page);
    }
}
```
This trades a small amount of one-time boilerplate (one line per page class, in one
central place) for no reflection and full compile-time safety. Either approach is fine;
pick the registry if the team is uncomfortable with reflection-based construction.

### Usage recap

```java
@ReloginPerTest
public class CheckoutFlowTest {
    @Test
    void appliesDiscountCode(LoginPage login, CheckoutPage checkout) {
        login.loginAsStandardUser();
        checkout.applyDiscountCode("SAVE10");
        assertThat(checkout.getTotal()).isEqualTo("90.00");
    }
}
```
`login` and `checkout` are both resolved automatically by `PageObjectResolver`, on
demand, with no `BaseTest` and no manual construction anywhere in the test.

---

## 7. Handling internal vs. SaaS instance differences

### Problem being solved
The application under test ships as two instances — "internal" and "SaaS" — that are
almost identical, differing only in small, page-local ways (an extra field, a different
validation rule). The old framework handles this with `if (instance.equals("internal"))`
scattered inside test bodies.

### Design — decision tree

Use the smallest tool that fits the size of the difference. In order of preference:

1. **Difference is presence/absence of a UI element** → don't encode "which instance"
   anywhere; just query the DOM directly.
   ```java
   public void login(String user, String pass) {
       Locator department = page.locator("#department");
       if (department.count() > 0) {
           department.fill("IT");
       }
       page.fill("#username", user);
       page.fill("#password", pass);
       page.click("#submit");
   }
   ```
   This is self-adapting and requires no environment awareness at all. Prefer this
   whenever the DOM itself can answer the question.

2. **Difference is a data value** (a limit, a regex, expected text) that the DOM can't
   reveal → externalize it to a small, per-environment config, injected where needed.
   Use sparingly — this should stay small; it is not meant to become a dumping ground
   for every difference in the app (see point 3 for that case).
   ```yaml
   # config/internal.yml
   maxUsers: 500
   usernamePattern: "^[A-Z0-9]+$"

   # config/saas.yml
   maxUsers: unlimited
   usernamePattern: "^[a-zA-Z0-9_-]+$"
   ```

3. **Difference is structural/behavioral and lives inside a specific page's methods**
   (the common case described for this app) → **base class + thin subclasses per page**,
   using the Template Method pattern. Only pages that actually differ get this treatment;
   a page with no differences stays a single plain class.

   ```java
   public abstract class DropdownPage {
       protected final Page page;

       protected DropdownPage(Page page) {
           this.page = page;
       }

       // Shared logic — written once, lives only in the base class.
       public void submitForm() {
           page.click("#submit");
       }

       // The only method that differs between instances.
       public abstract void selectOptionFromDropdownA(String option);
   }

   public class InternalDropdownPage extends DropdownPage {
       public InternalDropdownPage(Page page) {
           super(page);
       }

       @Override
       public void selectOptionFromDropdownA(String option) {
           page.fill("#department", "IT"); // internal-only extra step
           page.selectOption("#dropdownA", option);
       }
   }

   public class SaasDropdownPage extends DropdownPage {
       public SaasDropdownPage(Page page) {
           super(page);
       }

       @Override
       public void selectOptionFromDropdownA(String option) {
           page.selectOption("#dropdownA", option);
       }
   }
   ```

   The decision of "which subclass to build" lives in **exactly one place** — the
   factory/instantiation logic used by `PageObjectResolver` (or `PageRegistry`) —  not
   scattered through tests or page logic:
   ```java
   DropdownPage page = currentInstance == Instance.INTERNAL
       ? new InternalDropdownPage(rawPage)
       : new SaasDropdownPage(rawPage);
   ```

4. **Difference is that an entire test simply does not apply to one instance** (not a
   partial difference — the whole scenario is instance-specific) → a custom conditional
   execution annotation, so the test shows as **skipped** in reports rather than being
   silently no-op'd inside the method body.

   ```java
   @Target(ElementType.METHOD)
   @Retention(RetentionPolicy.RUNTIME)
   @ExtendWith(InstanceCondition.class)
   public @interface InternalOnly {}

   public class InstanceCondition implements ExecutionCondition {
       @Override
       public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext ctx) {
           boolean isInternalOnlyTest =
               AnnotationSupport.isAnnotated(ctx.getElement(), InternalOnly.class);
           Instance current = TestEnvironment.getCurrentInstance();

           if (isInternalOnlyTest && current != Instance.INTERNAL) {
               return ConditionEvaluationResult.disabled(
                   "Internal-only test, skipped on instance: " + current);
           }
           return ConditionEvaluationResult.enabled("Applicable to current instance");
       }
   }
   ```
   ```java
   @Test
   @InternalOnly
   void adminPanelHasExtraAuditTab(AdminPage admin) {
       assertThat(admin.getAuditTab()).isVisible();
   }
   ```

### Why not a single shared "god config" for everything
A single large config file that captures every instance difference across a huge
application becomes its own maintenance burden — a monolithic, ever-growing sheet of
flags nobody fully understands or dares modify, and every page becomes implicitly
coupled to it. Base-class-plus-subclass keeps each page's differences local to that
page's own small hierarchy: if 15 out of 200 pages differ, you have 15 small subclass
pairs, not one 500-line config everyone is scared to touch. Reserve config files for
genuine small **data** differences (point 2), not structural/behavioral ones.

### Why not annotations + reflection to pick behavior at runtime
An `@InternalOnly`-style annotation on a *method implementation* (as opposed to on a
whole test, per point 4) is just metadata — it does not by itself cause a different
method body to run. Making it drive dispatch would require reflection to scan for
annotated overloads and invoke the correct one manually, which is reimplementing what
Java's virtual dispatch (method overriding) already does natively, faster, with full IDE
support (find-usages, go-to-implementation, compile-time checking). Prefer plain
overriding (section above) for behavioral differences.

### Why not method overloading
Overloading is resolved by the **compiler** based on argument types/count at compile
time; it has no notion of "which environment is running," which is a **runtime**
decision. The correct mechanism for a runtime-dependent choice of behavior is
**overriding** (different subclass, same method signature, resolved by which object
you're holding), not overloading. This is why the base-class/subclass design is used
instead.

---

## 8. Shared test account contention (login collision) → per-role account pool

### Problem being solved
The application under test logs out any session when the same account logs in
elsewhere. With multiple threads running tests in parallel, two threads might use the
same test account concurrently, invalidating each other's sessions mid-test.

### Design
Treat test accounts as a limited, shared resource pool (analogous to a DB connection
pool). A test (or test class, depending on login cadence — section 5) must **acquire**
an account before it may use it, and **release** it afterward. If no account is
currently free, the acquiring thread blocks (or times out) until one becomes available.
Size each pool with at least as many accounts as your parallel thread count to avoid
contention/queuing.

### Code

```java
public class UserPool {
    private final BlockingQueue<TestUser> available;

    public UserPool(List<TestUser> users) {
        this.available = new LinkedBlockingQueue<>(users);
    }

    /** Blocks until a user becomes available. */
    public TestUser acquire() throws InterruptedException {
        return available.take();
    }

    /** Fails fast instead of hanging forever if a lease was leaked somewhere. */
    public TestUser acquire(Duration timeout) throws InterruptedException {
        TestUser user = available.poll(timeout.toMillis(), TimeUnit.MILLISECONDS);
        if (user == null) {
            throw new IllegalStateException(
                "Timed out waiting for an available test user. Check for leaked leases.");
        }
        return user;
    }

    public void release(TestUser user) {
        available.offer(user);
    }
}

public final class UserPools {
    public static final UserPool STANDARD = new UserPool(loadUsers("standard-users.yml"));
    public static final UserPool ADMIN = new UserPool(loadUsers("admin-users.yml"));

    public static UserPool forCurrentRole() {
        // Extend with real role-selection logic (e.g. based on a test tag/annotation).
        return STANDARD;
    }

    private static List<TestUser> loadUsers(String configFile) {
        // Load from YAML/properties/whatever config format the project already uses.
        ...
    }
}
```

The login extensions in section 5 already call `UserPools.forCurrentRole().acquire()`
in their `beforeEach`/`beforeAll` and `.release(user)` in their `afterEach`/`afterAll` —
this guarantees the release always runs (even on test failure), because JUnit5 always
invokes `afterEach`/`afterAll` regardless of test outcome.

### Practical guidance
- **Pool size vs. thread count**: pool size ≥ thread count avoids queuing entirely; less
  than that is still correct, just slower (tests wait their turn).
- **Always release in `afterEach`/`afterAll`**, never conditionally — an exception
  mid-test must not leak a permanently-held account.
- Once acquired, the account is exclusively owned for the duration of the lease, so it's
  now safe to cache/reuse that user's `storageState` file (section 4) without
  collision risk.
- Consider a library like **Apache Commons Pool2** instead of hand-rolling the
  `BlockingQueue` if you want built-in validation-on-borrow, eviction policies, or more
  configurable max-wait behavior.

---

## 9. Runner choice: JUnit5 vs. TestNG

This design assumes **JUnit5**, because:
- Its `ParameterResolver`/`Extension` model is exactly the composition mechanism this
  architecture depends on (sections 5 and 6).
- Playwright's official Java integration (`@UsePlaywright`, `OptionsFactory`) is built
  specifically for JUnit5 and provides free, correct per-test browser/page lifecycle
  management out of the box.

If the team is unable to move off TestNG:
- All the *conceptual* pieces still apply (account pool, `storageState`, per-page
  subclassing for instance differences), but the **composition mechanism does not** —
  TestNG has no equivalent to `ParameterResolver`/`@ExtendWith` for ad-hoc, mixable
  cross-cutting concerns.
- You would fall back to manual calls to acquire/release and page construction inside
  `@BeforeMethod`/`@BeforeClass`/`@AfterMethod`/`@AfterClass`, and likely still need some
  shared base class — meaning some of the God-class/rigid-hierarchy problems this
  redesign is meant to solve would partially persist. Migrating to JUnit5 is strongly
  recommended to get the full benefit of this design.

---

## 10. Summary of the end-to-end example

Putting every piece together, a finished test class looks like this:

```java
@ReloginPerTest
public class FirstFormValidationTest {

    @Test
    void requiredFieldShowsError(FirstFormPage form) {
        form.submitEmpty();
        assertThat(form.getErrorMessage()).isEqualTo("Field required");
    }
}

@ReloginPerClass
public class FirstFormWorkflowTest {

    @Test
    void step1FillsBasicInfo(FirstFormPage form) {
        form.fillStep1();
    }

    @Test
    void step2FillsAdvancedInfo(FirstFormPage form) {
        form.fillStep2();
    }
}
```

What happens when `requiredFieldShowsError` runs:
1. `@ReloginPerTest` expands to `@ExtendWith({PageObjectResolver.class, ReloginBeforeEachExtension.class})`.
2. `ReloginBeforeEachExtension.beforeEach` acquires a `TestUser` from `UserPools`,
   creates an authenticated `BrowserContext` via `LoginHelper` (using a cached
   `storageState` file, or performing a real login if none exists yet), opens a `Page`,
   and stores both in the JUnit5 extension store.
3. JUnit5 resolves the `form` parameter: `PageObjectResolver.supportsParameter` matches
   `FirstFormPage` (implements `BasePage`), then `resolveParameter` reads the `Page`
   from the store and constructs a `FirstFormPage` (choosing `InternalDropdownPage` vs.
   `SaasDropdownPage`-style subclass automatically wherever a page needs
   instance-specific behavior — section 7).
4. The test body runs using that `form` instance.
5. `ReloginBeforeEachExtension.afterEach` closes the `BrowserContext` and releases the
   `TestUser` back to the pool, regardless of whether the test passed or failed.

`FirstFormWorkflowTest` behaves identically except steps 2 and 5 run once for the whole
class (`beforeAll`/`afterAll`) instead of once per test method, purely because of the
different annotation — no other code differs.

---

## 11. Migration checklist for the agent performing this rewrite

1. Introduce `BasePage` marker interface; have all existing Page Object classes
   implement it, and normalize their constructors to accept a Playwright `Page`.
2. Build `BrowserFactory` (or adopt `@UsePlaywright`/`OptionsFactory`) for thread-scoped
   browser management. Remove all `static Browser`/`static BrowserContext` fields.
2. Build `LoginHelper` with `storageState` capture/reuse per test user/role.
3. Build `UserPool`/`UserPools` and wire real account lists from existing test data
   config.
4. Build `PageObjectResolver` (or `PageRegistry` if avoiding reflection) and
   `TestContextKeys`.
5. Build `ReloginBeforeEachExtension`, `ReloginOncePerClassExtension`, and their
   `@ReloginPerTest`/`@ReloginPerClass` meta-annotations.
6. For each existing Page Object, check whether it actually differs between "internal"
   and "SaaS" instances:
   - No difference → leave as a single class, have it implement `BasePage`.
   - Presence/absence of an element → inline DOM check, no subclassing needed.
   - Structural/behavioral difference → introduce `abstract` base + two thin
     subclasses (Template Method), update the resolver's instantiation logic to pick the
     correct subclass based on the current instance.
7. Delete `BaseTest`, `LoginBeforeEachTest`, `LoginOnce`, and any other classes in the
   old login-subclass chain once all tests have been migrated to use
   `@ReloginPerTest`/`@ReloginPerClass` + constructor-injected Page Objects.
8. Replace any remaining `if (instance.equals("internal"))` blocks inside test bodies
   using the decision tree in section 7 — do not leave any inline instance branching in
   test methods.
9. Enable JUnit5 parallel execution config and validate no test failures appear that
   didn't appear in serial execution (a strong signal that some shared state was missed).
10. Add tests/spot-checks confirming: (a) two parallel tests using the same role never
    collide (pool blocks correctly), (b) `storageState` reuse actually skips the login UI
    flow on subsequent runs, (c) instance-specific subclasses are picked correctly when
    running against each of "internal" and "SaaS".
