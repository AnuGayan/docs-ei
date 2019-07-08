# Mitigating Cross Site Request Forgery Attacks

Follow the instructions given below to secure any custom applications in your Micro Integrator from Cross Site Request Forgery (CSRF) attacks.

## How can CSRF attacks be harmful?

Cross Site Request Forgery (CSRF) attacks trick you to send a malicious
request by forcing you to execute unwanted actions on an already
authenticated web browser. The session in which you logged in to the web
application on the browser is used to bypass the authentication step
during this attack. If you are already authenticated on the website, the
site or application cannot distinguish between a forged request and a
legitimate request. Therefore, it is also known as "session riding".

The attack includes maliciously tricking you to click a URL or HTML
content that will consequently send a request to the website. For
example:

-   You send a request to an online banking application to transfer $100
    to another bank account.
-   An example URL including the parameters (i.e. account number and
    transfer amount) of a request is similar to the following:
    `                     https://bank.com/transfer.do?acct=10220048&amount=100                   `
-   The attacker uses this same URL by replacing the actual account
    number with a malicious account number. Then the attacker disguises
    this URL by including it in a clickable image and sends it to you in
    an email with other content.
-   You may unknowingly click on this URL, which will send a transfer
    request to the bank to transfer money to the malicious bank account.

## Mitigating CSRF attacks

[OWASP
CSRFGuard](https://www.owasp.org/index.php/Category:OWASP_CSRFGuard_Project)
is an OWASP flagship project that provides synchronizer token
pattern-based CSRF protection in a comprehensive and customizable
manner. You can use the best practices and configuration recommendations
of OWASP CSRFGuard to mitigate CSRF attacks in applications hosted on
the WSO2 platform. Fine-tuned configuration values of CSRFGuard
increases security, based on the security requirements of the specific
application.

CSRFGuard offers complete protection over CSRF scenarios by covering
HTTP POST, HTTP GET as well as AJAX-based requests. You can protect
forms based on HTTP POST and HTTP GET methods by injecting CSRF tokens
into the “action” of the form, or by embedding a token in a hidden
field.

You can protect HTTP GET requests sent as a result of resource
inclusions and links can by appending a relevant token in the “href” or
“src” attributes. Include these tokens manually using provided JSP tag
library or by using a JavaScript based automated injection mechanism.
AJAX requests are protected by injecting an additional header, which
contains a CSRF token.

## Configuring applications in WSO2 product to mitigate CSRF attacks

See the following for instructions on manually updating CSRF configurations. 

### Securing web applications

Follow the steps below to secure web applications.

1.  Add the following configurations in the
    `web.xml` file of your application.

    ``` xml
    <!-- OWASP CSRFGuard context listener used to read CSRF configuration -->
    <listener>
        <listener-class>org.owasp.csrfguard.CsrfGuardServletContextListener</listener-class>
    </listener>
    <!-- OWASP CSRFGuard session listener used to generate per-session CSRF token -->
    <listener>
        <listener-class>org.owasp.csrfguard.CsrfGuardHttpSessionListener</listener-class>
    </listener>
    <!-- OWASP CSRFGuard per-application configuration property file location-->
    <context-param>
        <param-name>Owasp.CsrfGuard.Config</param-name>
        <param-value>/repository/conf/security/Owasp.CsrfGuard.properties</param-value>
    </context-param>
    <!-- OWASP CSRFGuard filter used to validate CSRF token-->
    <filter>
        <filter-name>CSRFGuard</filter-name>
        <filter-class>org.owasp.csrfguard.CsrfGuardFilter</filter-class>
    </filter>
    <!-- OWASP CSRFGuard filter mapping used to validate CSRF token-->
    <filter-mapping>
        <filter-name>CSRFGuard</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <!-- OWASP CSRFGuard servlet that serves dynamic token injection JavaScript (application can customize the URL pattern as required)-->
    <servlet>
        <servlet-name>JavaScriptServlet</servlet-name>
        <servlet-class>org.owasp.csrfguard.servlet.JavaScriptServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>JavaScriptServlet</servlet-name>
        <url-pattern>/csrf.js</url-pattern>
    </servlet-mapping>
    ```

2.  Include the following JavaScriptServlet as the first JavaScript
    inclusion of the `<head>` element, in the HTML
    template of all pages of the application that you need to protect.

    ``` java
    …
    <html>
        <head>
            …
            <script type=”text/javascript” src=”/csrf.js”></script>

                <!-- other JavaScript inclusions should follow “csrf.js” inclusion -->
                <script type=”text/javascript” src=”/main.js”></script>
                …
        </head>
        <body>
            ...
        </body>
    </html>
    ```

3. Add the following configurations to the ei.toml file to specify the patterns that should be excluded from CSRF protection.
   ```java
    [[owasp.csrfguard.unprotected.service]]
    name = "oauthiwa"
    service = "%servletContext%/commonauth/iwa/*" 
   ```

4. Add the following configurations to the ei.toml file to specify the patterns that should be excluded from CSRF protection.
   ```java
   [owasp.csrfguard]
   create_token_per_page=false 
   token_length=32
   random_number_generator_algo="SHA1PRNG" 
   ```

5. Other..
    ```
    [owasp.csrfguard.js_servlet]
    x_request_with_header = "WSO2 CSRF Protection"
    ```
   
### Securing Jaggery applications

Follow the steps below to secure Jaggery applications.

1.  Add the following configurations in the
    `jaggery.conf` file of your application.

    ``` java
    "listeners" : [
    {
        "class" : "org.owasp.csrfguard.CsrfGuardServletContextListener"
    },
    {
        "class" : "org.owasp.csrfguard.CsrfGuardHttpSessionListener"    
    }
    ],
    "servlets" : [
    {
    "name" : "JavaScriptServlet",
    "class" : "org.owasp.csrfguard.servlet.JavaScriptServlet"
    }
    ],
    "servletMappings" : [
    {
    "name" : "JavaScriptServlet",
    "url" : "/csrf.js"
    }
    ],
    "contextParams" : [
    {
    "name" : "Owasp.CsrfGuard.Config",
    "value" : "/repository/conf/security/Owasp.CsrfGuard.dashboard.properties"
    }
    ]
    ```

2.  Include the following JavaScriptServlet as the first JavaScript
    inclusion of the `           <head>          ` element in the HTML
    template of all pages of the application that you need to protect.

    ``` js
    <html>
        <head>
            …
            <script type=”text/javascript” src=”/csrf.js”></script>

            <!-- other JavaScript inclusions should follow “csrf.js” inclusion -->
            <script type=”text/javascript” src=”/main.js”></script>
            …
        </head>
        <body>
            ...
        </body>
    </html>
    ```

3.  Add the following configurations to the ei.toml file to specify the patterns that should be excluded from CSRF protection.
   ```java
    [[owasp.csrfguard.unprotected.service]]
    name = "oauthiwa"
    service = "%servletContext%/commonauth/iwa/*" 
   ```

4. Add the following configurations to the ei.toml file to specify the patterns that should be excluded from CSRF protection.
   ```java
   [owasp.csrfguard]  
   create_token_per_page=false 
   token_length=32
   random_number_generator_algo="SHA1PRNG" 

   ```
5. Other..
    ```
    [owasp.csrfguard.js_servlet]
    x_request_with_header = "WSO2 CSRF Protection"
    ```
