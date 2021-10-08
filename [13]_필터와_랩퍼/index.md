# Chapter 13. 필터와 랩퍼

## 1. 필터

- 전체 웹 애플리케이션 기능을 확장 ex> 보안, 요청에 대한 로그
- 일반 자바 컴포넌트. 서블릿으로 요청 처리 전, 클라이언트에게 넘겨주기전 응답을 가로채어 처리할 수 있다.

### Request/Response 필터

---

- Request 필터로 처리할만한 것들
    - 보안 관련 내용을 체크
    - 요청 헤더와 바디 포맷팅 수정
    - 요청을 감시하거나 기록으로 남김
- Response 필터로 처리할만한 것들
    - 응답 스트림을 압축
    - 응답 스트림에 내용을 추가하거나 수정함
    - 완전 다른 새로운 응답을 만듬

### 필터와 서블릿은 유사하다

---

- 컨테이너가 이들의 API 를 알고있음
- 컨테이너가 생명주기를 관리함
- DD 에 설정함

구현 → Filter 인터페이스

### Filter Interface

---

- `javax.servlet.Filter`
    
    ```java
    public interface Filter {
        // init() 메소드는 반드시 구현
        // Container가 Filter를 인스턴스화 할 때 실행
        // Filter 호출 전 뭔가 설정할 것이 있는 경우 (일반적으로 FilterConfig 객체를 내부 field에 저장)
        public void init(FilterConfig) throws ServletException;
    
        // 실제 filter를 구현
        public void doFilter(ServletRequest, ServletResponse, FilterChain) throws IOException, ServletException;
    
        // Conteriner가 Filter의 인스턴스를 제거할 때 호출 (삭제 전 수행할 코드)
        public void destroy();
    }
    ```
    
- 필터의 생명주기
    - `init()`: `FilterConfig` 객체 저장 역할이 주
    - `doFiler()`: 처리할 것
    - `destroy()` : 필터 삭제 전에 할 작업

### FilterConfig Interface

---

- 메소드: `getFilterName`, `getInitParameter`, `getInitParameterNames`, `getServletContext`

### FilterChain Interface

---

- `javax.servlet.FilterChain`
    
    ```java
    public interface FilterChain {
        public void doFilter (ServletRequest, ServletResponse) throws IOException, ServletException;
    }
    ```
    
1. DD 를 통해 filter 의 순서를 지정할 수 있고 `FilterChain` 이 다음에 실행할 filter 를 판단함으로써 순차적 진행을 가능하게 함
2. `Filter.doFilter()`와 다름
3. `FilterChain.doFilter()`는
    - 다음에 호출할 filter 가 무엇인지 파악하여 `doFilter()`를 실행
    - 체인의 마지막 filter인 경우 Servlet 의 `service()` 메소드 호출

### 개념적인 스택 모델

---

- filter 수도코드 (p.743)
    
    ```java
    class MyCompressionFilter implements Filter {
        init();
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
            // 요청을 처리
            chain.doFilter(request, response);
            // 응답을 처리
        }
        destroy();
    }
    ```
    
1. 요청이 들어오면 `Filter` A의 `doFilter()` 메소드 호출
    - `doFilter()`에서 `FilterChain` 인스턴스의 `doFilter()`를 만날 때 까지 수행
2. 컨테이너는 `Filter` B의 `doFilter()`를 Stack 의 제일 위에 쌓음
    - `doFilter()`에서 `FilterChain` 인스턴스의 `doFilter()`를 만날 때까지 수행
3. 컨테이너는 `Servlet`의 `service()`를 Stack 의 제일 위에 쌓음
4. 컨테이너는 `service()` 실행 완료 시 Stack 에서 제거
5. 컨테이너는 `Filter` B의 `doFilter()` 메소드로 돌아옴
    - `doFilter()`에서 `FilterChain` 인스턴스의 `doFilter()` 이후의 로직을 수행
        - 이후 로직에서 response 관련 작업
6. 컨테이너는 `Filter` B의 `doFilter()` 실행 완료시 Stack 에서 제거
7. 컨테이너는 `Filter` A의 `doFilter()`로 돌아옴
    - `doFilter()`에서 `FilterChain` 인스턴스의 `doFilter()` 이후의 로직을 수행
        - 이후 로직에서 response 관련 작업
8. 컨테이너는 `Filter` A의 `doFilter()` 실행 완료시 Stack 에서 제거
9. 컨테이너는 응답 완료함


## 2. Request Filter

- 컨테이너가 실행 순서 지정하는 규칙
    1. `<url-pattern>`으로 일치하는 필터가 체인에 순서대로 들어감
        - 일치하는 모든 필터를 모두 체인에 넣고 실행
    2. 이후 `<servlet-name>`으로 일치하는 필터를 찾아 그 정의된 순서대로 체인에 등록

- `web.xml`
    - 필터 정의: Client 로부터 직접 들어오는 요청에 대해서 필터 적용 가능
        
        ```xml
        필터 정의
        <filter>
          <filter-name>BeerRequest</filter-name>
          -------------------------------------- 
          필수 - 필터 이름
          <filter-class>com.example.web.BeerRequestFilter</filter-class>
          -------------------------------------------------------------
          필수 - 필터 클래스 이름(패키지명 포함)
          <dispatcher>REQUEST | INCLUDE | FORWARD | ERROR </dispatcher>
          -------------------------------------------------------------
          <init-param>
             <param-name>LogFileName</param-name>
             <param-value>UserLog.txt</param-value>
          </init-param>
          -------------
          선택 - 여러개 존재해도 됨.
        </filter>
        ```
        
    - URL 패턴과 필터 맵핑 선언하기
        
        ```xml
        <filter-mapping>
          <filter-name>BeerRequest</filter-name>
          --------------------------------------
          필수 - filter의 <filter-name>에 존재해야 함.
          <url-pattern>*.do</url-pattern>
          -------------------------------
          필수 - <url-pattern> 혹은 <servlet-name> 중 하나는 존재해야 함.
        </filter-mapping>
        ```
        
    - 서블릿 이름에 필터 맵핑 선언하기
        
        ```xml
        <filter-mapping>
          <filter-name>BeerRequest</filter-name>
          <servlet-name>AdviceServlet</servlet-name>
        </filter-mapping>
        ```
        
    
    ```xml
    <web-app ...>
        <servlet> ... </servlet>
    
        <!-- filter-name, filter-class는 반드시 필요한 항목 -->
        <!-- init-param은 옵션 -->
        <filter>
            <filter-name>BeerRequest</filter-name>
            <filter-class>com.example.web.BeerRequestFilter</filter-class>
            <init-param>
                <param-name>LogFileName</param-name>
                <param-value>UserLog.txt</param-value>
            </init-param>
        </filter>
    
        <!-- filter-mapping 선언규칙 -->
        <!-- filter-name은 반드시 필요하며, filter 선언을 찾기 위해 필요 -->
        <!-- url-pattern, servlet-name 중 하나는 반드시 필요 -->
        <filter-mapping>
            <filter-name>BeerRequest</filter-name>
            <!-- url-pattern은 어떤 웹 어플리케이션 리소들에 필터를 적용할지 정의 -->
            <url-patterm>*.do</url-pattern>
        </filter-mapping>
    
        <filter-mapping>
            <filter-name>BeerRequest</filter-name>
            <!-- 필터를 적용할 웹 어플리케이션 하나를 정의 -->
            <servlet-name>AdviceServlet</servlet-name>
        </filter-mapping>
    </web-app>
    ```
    

- (서블릿 스펙 2.4 부터) RequestDispatcher 로 들어오는 요청에도 필터 적용 가능
    - 서블릿 스펙 2.4 부터
        - `RequestDispatcher` 요청 시 특별한 정보를 제공하도록 다섯가지 새로운 요청 속성(attribute)을 추가하였다.
            - REQUEST, FORWARD, INCLUDE, ERROR와 같은 가능한 값을 사용하여 새 `<dispatcher>` 요소를 정의할 수 있다.
    - 서블릿 스펙 3.0 부터: ASYNC, REQUEST, FORWARD, INCLUDE, ERROR와 같은 가능한 값을 사용하여 새 `<dispatcher>` 요소를 정의할 수 있다.
    - forward, include, request dispatch, error 를 통해 웹 리소스를 요청하는 경우에도 필터 적용 가능
        
        ```xml
        <filter-mapping>
            <filter-name>MonitorFilter</filter-name>
            <url-pattern>*.do</url-pattern>
            <!-- dispatcher 항목을 0..4까지 포함 가능 -->
            <!-- request는 client가 요청할 때 적용한다는 의미 -->
            <!-- dispatcher 항목이 없는 경우 request가 DEFAULT -->
            <dispatcher>request</dispatcher>
                그리고 / 또는
            <!-- include는 client가 포함될 때 적용 -->
            <dispatcher>include</dispatcher>
                그리고 / 또는
            <!-- forward는 client 요청을 다른 곳으로 넘길 때 적용 -->
            <dispatcher>forward</dispatcher>
                그리고 / 또는
            <!-- error는 error 핸들러는 호출했을 때 적용 -->
            <dispatcher>error</dispatcher>
        </filter-mapping>
        ```
        
    - REQUEST: 클라이언트의 요청할 때 적용 (기본값: `<dispatcher>` 항목이 없는 경우)
    - INCLUDE: 클라이언트가 포함(include)할 때 적용
    - FORWARD: 클라이언트의 요청을 다른 곳으로 넘길 때(forward) 적용
    - ERROR: error 핸들러를 호출했을 때

### RequestDispatcher

---

- `RequestDispatcher`
    - 클라이언트로부터 최초에 들어온 요청을 JSP/Servlet 내에서 원하는 자원으로 요청을 넘기는(보내는) 역할을 수행하거나, 특정 자원에 처리를 요청하고 처리 결과를 얻어오는 기능을 수행하는 클래스
    - 즉 /a.jsp 로 들어온 요청을 /a.jsp 내에서 `RequestDispatcher`를 사용하여 b.jsp로 요청을 보낼 수 있다.
    - 또는 a.jsp에서 b.jsp로 처리를 요청하고 b.jsp에서 처리한 결과 내용을 a.jsp의 결과에 포함시킬 수 있습니다.
- 메소드
    - `forward()`
    - `include()`
- `RequstDispatcher`를 얻는 방법 2가지
    - `Request`
    - `Context`: `ServletContext` 로부터 리턴받는 경우
        
        ```java
        RequestDispatcher view = getServletContext().getRequestDispatcher("/result.jsp");
        ```
        
        - 상대 경로를 사용할 수 없다. 반드시 "/" 시작으로 경로를 명시해야 한다.

- `HttpServletResponse.sendRedirect()` 와의 차이점
    
    ![Untitled](Chapter%2013%20%E1%84%91%E1%85%B5%E1%86%AF%E1%84%90%E1%85%A5%E1%84%8B%E1%85%AA%20%E1%84%85%E1%85%A2%E1%86%B8%E1%84%91%E1%85%A5%201756a5a91dc74fb5beabe99969718b3d/Untitled.png)
    
    - `sendRedirect()`는 HTTP 리다이렉션을 이용하기 때문에 하나의 요청 범위 안에서 처리를 하는것이 아니라 브라우저에게 Response 후 브라우저 측에서 지정받은 요청 경로로 다시 재요청을 하는 방식
        
        → 두 번의 HTTP 트랜잭션이 발생
        
    - 서버측에서는 최초에 받은 요청중에 처리한 내용을 리다이렉트 된 요청안에서 공유할 수 없는 문제가 있다.
    - `HttpServletResponse`를 통해 리다이렉트 하는 방식은 현재 어플리케이션 이외에 다른 자원의 경로를 요청할 수 있는 반면, `RequestDispatcher`는 현재 처리중인 서블릿이 속해 있는 웹 어플리케이션 범위 내에서만 요청을 제어할 수 있다.


## 3. Response Filter

- 심각한 문제점 (p.749) → 서블릿에 넘겨주는 response 객체를 서블릿이 사용하게 되면, 필터를 거치지 않고 바로 클라이언트로 response 하게 된다.
    1. `Filter`는 Servlet 으로 request, response객체를 전달
        - Servlet 에게 다시 response 객체를 전달받기를 기다림
    2. Servlet 은 자신이 받은 인자가 filtering 되었다는 사실을 모름
        - 출력에 대한 제어는 컨테이너가 가져감
        - 컨테이너는 결과를 Client 에게 전달
    3. `Filter`는 response 객체를 전달받기 위해 계속 기다림

- 해결책
    - 4가지의 wrapper class를 사용하여 Response 객체를 새롭게 정의한 후 Servlet에게 전달
    - Servlet 에서 새 Response 객체에 기록 시 Client 로 전달되기 전에 제어 가능
- 실제 압축 필터 코드
    
    ```java
    ...
    public class CompressionFilter implements Filter {
      private ServletContext ctx;
      private FilterConfig cfg;
    
      public void init(FilterConfig cfg) throws ServletException {
        this.cfg = cfg;
        ctx = cfg.getServletContext();
        ctx.log(cfg.getFiltername() + " initialized.");
      }
    
      public void doFilter(ServletRequest req, ServletResponse resp, FilterChain fc) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)req;
        HttpServletResponse response = (HttpServletResponse)resp;
    
        String valid_encodings = request.getHeader("Accept-Encoding");
    
        if(valid_encodings.indexOf("gzip") > -1) {
          CompressionResponseWrapper wrappedResp = new CompressionResponseWrapper(response);
          //-------------------------------------------
          // 압축 랩퍼 클래스 생성
          wrappedResp.setHeader("Content-Encoding", "gzip");
          //-------------------------------------------------
          // 응답 컨텐츠가 gzip으로 인코딩됨을 선언하는 부분
          fc.doFilter(request, wrappedResp);
          //---------------------------------
          //체인에 있는 다음 컴포넌트를 호출합니다.
    
          GZIPOutputStream gzos = wrappedResp.getGZIPOutputStream();
          gzos.flush();
          //-------------
          // 원본 스트림으로 모든 데이터를 전송합니다.
          ctx.log(cfg.getFilterName() + ": finished the request.");
        } else{
          ctx.log(cfg.getFilterName() + ": no encoding performed.");
        }
      }
    
      public void destroy() {
        // 인스턴스 변수들을 null 처리 한다.
        cfg = null; ctx = null;
      }
    }
    ```
    

## 4. Wrapper

- 랩퍼 클래스는 현재 구현하려는 타입의 객체를 내부에 가짐
- 모든 메소드 호출을 위임할 동일한 타입의 클래스 객체에 대한 참조를 내부에서 가짐
- 서블릿에서 제공하는 4개의 랩퍼 클래스
    - `ServletRequestWrapper`
    - `HttpServletRequestWrapper`
    - `ServletResponseWraper`
    - `HttpServletResponseWraper`

- 랩퍼클래스 코드
    
    ```java
    import java.io.IOException;
    import java.io.OutputStreamWriter;
    import java.io.PrintWriter;
    import java.util.zip.GZIPOutputStream;
    
    import javax.servlet.ServletOutputStream;
    import javax.servlet.http.HttpServletResponse;
    import javax.servlet.http.HttpServletResponseWrapper;
    
    public class CompressionResponseWrapper extends HttpServletResponseWrapper {
        private GZIPServletOutputStream servletGzipOS = null;
        private PrintWriter pw = null;
    
        public CompressionResponseWrapper(HttpServletResponse response) {
            super(response);
        }
       
        public void setContentLength(int len) {}
       
        public GZIPOutputStream getGZIPOutputStream() {
            return this.servletGzipOS.internalGzipOS;
        }
       
        private Object streamUsed = null;
       
        public ServletOutputStream getOutputStream() throws IOException {
            if((streamUsed != null) && (streamUsed != pw)) {
                throw new IllegalStateException();
            }
           
            if(servletGzipOS == null) {
                servletGzipOS = new GZIPServletOutputStream(getResponse().getOutputStream());
                streamUsed = servletGzipOS;
            }
            return servletGzipOS;
        }
       
        public PrintWriter getWriter() throws IOException {
            if((streamUsed != null) && (streamUsed != servletGzipOS)) {
                throw new IllegalStateException();
            }
           
            if(pw == null) {
                servletGzipOS = new GZIPServletOutputStream(getResponse().getOutputStream());
                OutputStreamWriter osw = new OutputStreamWriter(servletGzipOS, getResponse().getCharacterEncoding());
               
                pw = new PrintWriter(osw);
                streamUsed = pw;
            }
            return pw;
        }
    }
    ```
    
- `GZIPServletOutputStream` 코드
    
    ```java
    import java.io.IOException;
    import java.util.zip.GZIPOutputStream;
    
    import javax.servlet.ServletOutputStream;
    
    public class GZIPServletOutputStream extends ServletOutputStream {
        GZIPOutputStream internalGzipOS;
       
        GZIPServletOutputStream(ServletOutputStream sos) throws IOException {
            // TODO Auto-generated constructor stub
            this.internalGzipOS = new GZIPOutputStream(sos);
        }
       
        @Override
        public void write(int param) throws IOException {
            // TODO Auto-generated method stub
            internalGzipOS.write(param);
        }
    
    }
    ```
    

- 압축이 HTTP 를 만났을 때
    - 브라우저가 보내는 헤더 중 하나인 `Accept-Endcoing: gzip` 은 서버에게 이런 컨텐츠를 다룰 수 있다고 알려주는 것
    - 브라우저가 압축 데이터를 처리할 수 있으면 서버는 헤더에 `Content-Encoding: gzip`을 추가해서 컨텐츠가 압축되었음을 알려줘야 한다.
    - 브라우저는 위 정보를 확인하고 화면을 출력하면 압축을 풀어야함을 알게된다.

- 참고
    - [http://freeminderhuni.blogspot.com/2013/09/head-firstservlets-jsp-13.html](http://freeminderhuni.blogspot.com/2013/09/head-firstservlets-jsp-13.html)
    - [https://dololak.tistory.com/502](https://dololak.tistory.com/502)
