## 리마인드

- @Target : 해당 타입(메서드, 변수같은)을 가진 곳에만 어노테이션 붙일 수 있음
- @Retention : 어노테이션 유지 정책(코드, .class, 런타임)

<br>

## Spring Annotation

- 스프링의 기능을 어노테이션으로 만든 것


### 컴포넌트, 빈

@Required : Bean 생성시 필수 프로퍼티임을 알림

<details>
<summary>@ComponentScan : @Component를 상속받는 클래스를 찾아 빈에 등록</summary>

```java 

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {

    // value 파라미터로 들어오면 basePackages로 처리
	@AliasFor("basePackages")
	String[] value() default {};

    // 파라미터가 생략되면 value와 같은 기능을 함
    // 경로 설정
	@AliasFor("value")
	String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};

    // 빈 이름 생성기
	Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    // 범위
	Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;

    // 프록시패턴 사용하는지
	ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;

    // 필터 설정
	String resourcePattern() default ClassPathScanningCandidateComponentProvider.DEFAULT_RESOURCE_PATTERN;
	boolean useDefaultFilters() default true;
	Filter[] includeFilters() default {};
	Filter[] excludeFilters() default {};

    // 지연 생성
	boolean lazyInit() default false;

	@Retention(RetentionPolicy.RUNTIME)
	@Target({})
	@interface Filter {
		FilterType type() default FilterType.ANNOTATION;

		@AliasFor("classes")
		Class<?>[] value() default {};

        @AliasFor("value")
		Class<?>[] classes() default {};

		String[] pattern() default {};

	}

}


```
</details>

<details>
<summary>@Indexed : 탐색이 가능하도록 만드는 어노테이션</summary>

```java 

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Indexed {
}

```
</details>

<details>
<summary>@Component : 컴포넌트 스캔에 탐지되도록 만듦</summary>

```java 

@Documented
@Indexed
public @interface Component {
	String value() default "";
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Repository {
	@AliasFor(annotation = Component.class)
	String value() default "";
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {
	@AliasFor(annotation = Component.class)
	String value() default "";
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
	@AliasFor(annotation = Component.class)
	String value() default "";

	boolean proxyBeanMethods() default true;
}

```
</details>

<details>
<summary>@EnableAutoConfiguration : Spring Application Context를 자동으로 설정해줌</summary>

```java 

// @Import내 파일을 파싱하여 컴파일

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class) // 환경설정
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

    // 스코프 설정
	String[] basePackages() default {};
	Class<?>[] basePackageClasses() default {};
}

public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

    private static final AutoConfigurationEntry EMPTY_ENTRY = new AutoConfigurationEntry();
	private static final String[] NO_IMPORTS = {};
	private static final Log logger = LogFactory.getLog(AutoConfigurationImportSelector.class);
	private static final String PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = "spring.autoconfigure.exclude";
	private ConfigurableListableBeanFactory beanFactory;
	private Environment environment;
	private ClassLoader beanClassLoader;
	private ResourceLoader resourceLoader;
	private ConfigurationClassFilter configurationClassFilter;

    // getter, setter...
}

public abstract class AutoConfigurationPackages {

	private static final Log logger = LogFactory.getLog(AutoConfigurationPackages.class);
	private static final String BEAN = AutoConfigurationPackages.class.getName();

     // 빈 생성, 조회, 등록
	public static boolean has(BeanFactory beanFactory) {}
	public static List<String> get(BeanFactory beanFactory) {}
	public static void register(BeanDefinitionRegistry registry, String... packageNames) {}

    // 메타데이터 기록
	static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
		}

		public Set<Object> determineImports(AnnotationMetadata metadata) {
			return Collections.singleton(new PackageImports(metadata));
		}
	}

    // 패키지 import
	private static final class PackageImports {

		private final List<String> packageNames;

		PackageImports(AnnotationMetadata metadata) {
			AnnotationAttributes attributes = AnnotationAttributes
					.fromMap(metadata.getAnnotationAttributes(AutoConfigurationPackage.class.getName(), false));
			List<String> packageNames = new ArrayList<>(Arrays.asList(attributes.getStringArray("basePackages")));
			for (Class<?> basePackageClass : attributes.getClassArray("basePackageClasses")) {
				packageNames.add(basePackageClass.getPackage().getName());
			}
			if (packageNames.isEmpty()) {
				packageNames.add(ClassUtils.getPackageName(metadata.getClassName()));
			}
			this.packageNames = Collections.unmodifiableList(packageNames);
		}
	}

    // BasePackages 기록
	static final class BasePackages {
		private final List<String> packages;
		private boolean loggedBasePackageInfo;
		BasePackages(String... names) {}
		List<String> get() {}
	}

    // BasePackages 관련 세팅
	static final class BasePackagesBeanDefinition extends GenericBeanDefinition {

		private final Set<String> basePackages = new LinkedHashSet<>();

		BasePackagesBeanDefinition(String... basePackages) {
			setBeanClass(BasePackages.class);
			setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			addBasePackages(basePackages);
		}

		public Supplier<?> getInstanceSupplier() {
			return () -> new BasePackages(StringUtils.toStringArray(this.basePackages));
		}

		private void addBasePackages(String[] additionalBasePackages) {
			this.basePackages.addAll(Arrays.asList(additionalBasePackages));
		}
	}
}

```
</details>

- @SpringBootApplication : @Configuration, @EnableAutoConfiguration, @ComponentScan 합침


<details>
<summary>@Bean : 빈에 등록</summary>

```java 

@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
	@AliasFor("name")
	String[] value() default {};

	@AliasFor("value")
	String[] name() default {};

    // 다른 빈에 연결되면 false
	boolean autowireCandidate() default true;

    // 생명 주기
	String initMethod() default "";
	String destroyMethod() default AbstractBeanDefinition.INFER_METHOD;

}


```
</details>

<details><summary>@Required : Bean 생성시 필수 프로퍼티임을 알림</summary></details>


### 의존성 주입

- @Qualifier("id123") : 같은 타입의 빈을 만들기 위해 별명을 붙여줌
- @Autowired : 의존성 주입1. 타입->이름->@Qualifier. \<context:annotation-config/\> 필요
- @Resource : 의존성 주입2. 이름->타입->@Qualifier. \<context:annotation-config/\> 필요
- @Inject : 의존성 주입3. 타입->@Qualifier->이름. javax import



### Response

- @ResponseBody : 자바 객체를 body형식으로 보냄
- @RestController : @Controller에 @ResponseBody가 붙음
- @ResponseStatus : status code, reason을 반환

### Request

- @RequestBody : body를 자바 객체에 매핑
- @RequestHeader : http header를 request에서 설정
- @RequestParam : 쿼리스트링을 변수에 할당
- @PathVariable : 파라미터를 변수에 할당
- @GetMapping : @RequestMapping에 메서드를 붙임. 4대 메서드(GET, POST, PUT, DELETE)사용가능
- @ModelAttribute : 쿼리스트링 변수를 DTO로 만듦
- @RequestAttribute : Request에 설정되어 있는 속성 값
- @RequestPart : MultipartFile을 바인딩

<details>
<summary>@RequestMapping : 경로에 메서드 매핑(mapper handler)</summary>

```java 

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {

    // 매핑 이름(#{name})
	String name() default "";

    // 경로 매핑
	@AliasFor("path")
	String[] value() default {};

    // 안씀
	@AliasFor("value")
	String[] path() default {};

    // http method
	RequestMethod[] method() default {};

    // 쿼리스트링 처리(!= 지원)
	String[] params() default {};

    // 헤더 처리(!= 지원)
	String[] headers() default {};

    // 경로 깊이 증가
	String[] consumes() default {};

    // Content-Type 읽기
	String[] produces() default {};

}
```
</details>

### 설정

- @PropertySource : properties파일을 파싱하여 Enviroment객체로 만듦
- @ConfigurationProperties : properties파일을 파싱하여 메서드로 만듦
- @Value : properties파일을 파싱하여 변수로 만듦

### 검증

- @Valid : 유효성 검증이 필요한 객체임을 지정한다.
- @InitBinder : @Valid 애노테이션으로 유효성 검증이 필요하다고 한 객체를 가져오기전에 수행해야할 method를 지정한다.


### AOP

- @ExceptionHandler(ExceptionClassName.class) : 해당 클래스의 예외를 캐치
- @ControllerAdvice : @ExceptionHandler(ExceptionClassName.class)의 어드바이스
- @RestControllerAdvice : @ControllerAdvice + @ResponseBody
- @Transactional :

- @PostConstruct, @PreConstruct : 객체 생성 전후 실행
- @PreDestroy : 객체 제거 전 실행

- @Transactional : 데이터베이스 트랜잭션

### Cache

- @Cacheable : 캐시 사용(순수 함수만)
- @CachePut : 캐시 업데이트
- @CacheEvict : 캐시 삭제
- @CacheConfig : 클래스 내 캐시 설정 공유

### 기타

- @Test : 어노테이션이 붙은 메서드를 테스트
- @Scheduled : 설정한 시간에 실행
- @Lazy : 지연로딩. 사용할 때 로딩함
- @CookieValue : 쿠키 값을 파라미터로 전달, 없으면 500 에러 반환
- @SessionAttributes : 세션에 데이터 할당

<details>
<summary>@CrossOrigin : CORS 허용</summary>

```java 

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CrossOrigin {

	@Deprecated
	String[] DEFAULT_ORIGINS = {"*"};
	@Deprecated
	String[] DEFAULT_ALLOWED_HEADERS = {"*"};
	@Deprecated
	boolean DEFAULT_ALLOW_CREDENTIALS = false;
	@Deprecated
	long DEFAULT_MAX_AGE = 1800;

	@AliasFor("origins")
	String[] value() default {};

	@AliasFor("value")
	String[] origins() default {};

    // 기본 패턴
	String[] originPatterns() default {};

	// 기본 허용 헤더
    //Cache-Control, Content-Language, Content-Type, Expires, Last-Modified, Pragma
	String[] allowedHeaders() default {};

	// 기본 이외 헤더(자동으로 넘어가지 않음)
	String[] exposedHeaders() default {};

	RequestMethod[] methods() default {};

	// 자격 증명
	String allowCredentials() default "";

	// 캐싱 유지 시간
	long maxAge() default -1;

}

```
</details>

