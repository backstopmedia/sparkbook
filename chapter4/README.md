# Chapter4: Security

What's in this chapter?

- Introduce the concept and fundamental architecture of Spark security 
- Management of ACL of web UI and applications on secure Spark
- Configuring ports for network security and encryption using SSL/TLS 
- Compatibility of external library

The following code samples have been used in this chapter to describe in a more comprehensive way the topics listed above.

Text files

1. Setup Configurations

```scala
    val conf = new SparkConf()
    conf.set(“spark.authenticate”, “false”)
    // Create context object with your configuration
    // This configuration will be reflected all
    // components and executors used your job.
    val sc = new SparkContext(new SparkConf())
```

```bash

    $ ./bin/spark-shell —master <Spark Master URL>  \
        --conf spark.authenticate=false \
        -—conf spark.authenticate.secret=secret
```
        
2. ACLs

```bash
    $ cp $SPARK_HOME/conf/spark-defaults.conf.template \
        $SPARK_HOME/conf/spark-defaults.conf
    $ vim $SPARK_HOME/conf/spark-defaults.conf
    $ cat $SPARK_HOME/conf/spark-defaults.conf
    # ...
    spark.authenticate                 true
    spark.authenticate.secret          mysecret
```
   
```bash
    $ cd $SPARK_HOME
    $ ./sbin/start-master.sh
```
    
3. Web UI

```java
    package my.application.filter

    import com.sun.jersey.core.util.Base64;

    import java.io.IOException;
    import java.io.UnsupportedEncodingException;
    import java.util.StringTokenizer;
    import javax.servlet.FilterConfig;
    import javax.servlet.ServletException;
    import javax.servlet.ServletRequest;
    import javax.servlet.ServletResponse;
    import javax.servlet.http.HttpServletRequest;
    import javax.servlet.http.HttpServletResponse;
    import javax.servlet.Filter;
    import javax.servlet.FilterChain;

    public class BasicAuthFilter implements Filter {
        String username = null;
        String password = null;

        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
            this.username = "spark-user";
            this.password = "spark-password";
        }

        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                             FilterChain filterChain) throws IOException, ServletException {
            HttpServletRequest request = (HttpServletRequest)servletRequest;
            HttpServletResponse response = (HttpServletResponse)servletResponse;

            String authHeader = request.getHeader("Authorization");
            if (authHeader != null) {
                StringTokenizer st = new StringTokenizer(authHeader);
                if (st.hasMoreTokens()) {
                    String basic = st.nextToken();
                    if (basic.equalsIgnoreCase("Basic")) {
                        try {
                            String credentials = new String(Base64.decode(st.nextToken()), "UTF-8");
                            int pos = credentials.indexOf(":");
                            if (pos != -1) {
                                String username = credentials.substring(0, pos).trim();
                                String password = credentials.substring(pos + 1).trim();

                                if (!username.equals(this.username) || 
                                    !password.equals(this.password)) {
                                    unauthorized(response, "Unauthorized:" +
                                            this.getClass().getCanonicalName());
                                }

                                filterChain.doFilter(servletRequest, servletResponse);
                            } else {
                                unauthorized(response, "Unauthorized:" +
                                        this.getClass().getCanonicalName());
                            }
                        } catch (UnsupportedEncodingException e) {
                            throw new Error("Counldn't retrieve authorization information", e);
                        }
                    }
                }
            } else {
                unauthorized(response, "Unauthorized:" + this.getClass().getCanonicalName());
            }
        }

        @Override
        public void destroy() {}

        private void unauthorized(HttpServletResponse response, String message) throws IOException {
            response.setHeader("WWW-Authenticate", "Basic realm=\"Spark Realm\"");
            response.sendError(401, message);
        }

    }
``` 
    
    
```java
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.Arrays;
import java.util.List;

public class UserRoleFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, 
      ServletResponse servletResponse, FilterChain filterChain) 
      throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)servletRequest;
        String user = "spark-userA";
        List<String> userList = Arrays.asList("spark-userA", "spark-userB", "spark-userC");

        filterChain.doFilter(new UserRoleRequestWrapper(user, userList, request), servletResponse);
    }

    @Override
    public void destroy() {

    }
}

```

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.util.List;

public class UserRoleRequestWrapper extends HttpServletRequestWrapper {
    String user;
    List<String> userList = null;
    HttpServletRequest request;

    public UserRoleRequestWrapper(String user, List<String> userList, HttpServletRequest originalRequest) {
        super(originalRequest);
        this.user = user;
        this.userList = userList;
        this.request = originalRequest;
    }

    @Override
    public String getRemoteUser() {
        if (this.userList.contains(this.user)) {
            return this.user;
        }
        return null;
    }
}
```

4. Encryption

```bash
$ keytool -genkey \
          -alias ssltest \          # The secret key name
          -keyalg RSA \              # The algorithm for encryption
          -keysize 2048 \
          -keypass key_password \ 　　# The key password
          -storetype JKS \
          -keystore my_key_store \
           -storepass store_password　#  The key store password
```

```bash
$ keytool -export \
          -alias ssltest \       # The key name
          -file my_cert.cer \    # The cert name
          -keystore my_key_store 
```

```bash
$ keytool -import -v \
          -trustcacerts \
          -alias ssltest \
          -file my_cert.cer \         # The cert name
          -keyStore my_trust_store \　# The name of trust store
          -keypass store_password     # The password of trust store

```


