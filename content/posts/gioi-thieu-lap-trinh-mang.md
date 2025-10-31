---
title: "Lập trình mạng - Hành trình từ cơ bản đến thành thạo"
date: 2025-10-23
draft: false
tags: ["Networking", "Java", "TCP/IP", "Socket", "Programming", "HTTP"]
description: "Hướng dẫn toàn diện về lập trình mạng từ những khái niệm cơ bản đến việc xây dựng ứng dụng mạng thực tế với Java"
---

## Giới thiệu

Trong thế giới công nghệ hiện đại, **lập trình mạng** (Network Programming) là một kỹ năng không thể thiếu đối với mọi developer. Từ việc gửi một email đơn giản đến xây dựng hệ thống phân tán phức tạp, tất cả đều dựa trên các nguyên lý lập trình mạng.

Bài viết này sẽ đưa bạn qua hành trình từ những khái niệm cơ bản nhất đến việc triển khai các ứng dụng mạng thực tế, với trọng tâm sử dụng Java.

## Lập trình mạng là gì?

**Lập trình mạng** là việc phát triển các chương trình máy tính có khả năng giao tiếp với nhau qua mạng máy tính. Điều này bao gồm:

- **Truyền dữ liệu** giữa các máy tính
- **Chia sẻ tài nguyên** như file, database, services
- **Xây dựng hệ thống phân tán** với multiple components
- **Tạo các giao thức** cho communication patterns

### Tại sao lập trình mạng quan trọng?

```
🌐 Internet của vạn vật (IoT)
📱 Mobile applications
☁️  Cloud computing
🏢 Enterprise systems
🎮 Online gaming
💰 Fintech và blockchain
```

## Kiến thức nền tảng

### 1. Mô hình OSI và TCP/IP

#### Mô hình OSI (7 layers)

```
7. Application Layer  ← HTTP, FTP, SMTP
6. Presentation Layer ← SSL/TLS, encryption
5. Session Layer      ← Session management
4. Transport Layer    ← TCP, UDP
3. Network Layer      ← IP, routing
2. Data Link Layer    ← Ethernet, WiFi
1. Physical Layer     ← Cables, radio waves
```

#### Mô hình TCP/IP (4 layers)

```
4. Application Layer  ← HTTP, FTP, DNS
3. Transport Layer    ← TCP, UDP
2. Internet Layer     ← IP, ICMP
1. Network Access     ← Ethernet, WiFi
```

### 2. Địa chỉ IP và Port

```java
// IPv4 address examples
192.168.1.100    // Private network
8.8.8.8          // Google DNS
127.0.0.1        // Localhost (loopback)

// IPv6 address example
2001:0db8:85a3:0000:0000:8a2e:0370:7334

// Port examples
80    // HTTP
443   // HTTPS
22    // SSH
21    // FTP
25    // SMTP
3306  // MySQL
5432  // PostgreSQL
```

### 3. Giao thức TCP vs UDP

| Đặc điểm | TCP | UDP |
|----------|-----|-----|
| **Reliability** | Đáng tin cậy | Không đảm bảo |
| **Connection** | Connection-oriented | Connectionless |
| **Speed** | Chậm hơn | Nhanh hơn |
| **Overhead** | Cao | Thấp |
| **Use cases** | Web, Email, File transfer | Gaming, Video streaming, DNS |

## Lập trình Socket với Java

### 1. Socket cơ bản

```java
import java.net.*;
import java.io.*;

public class BasicSocketExample {
    
    // TCP Client example
    public static void tcpClient() {
        try (Socket socket = new Socket("www.google.com", 80)) {
            
            // Gửi HTTP request
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true);
            out.println("GET / HTTP/1.1");
            out.println("Host: www.google.com");
            out.println("Connection: close");
            out.println(); // Empty line để kết thúc request
            
            // Đọc response
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            
            String line;
            while ((line = in.readLine()) != null) {
                System.out.println(line);
            }
            
        } catch (IOException e) {
            System.err.println("Client error: " + e.getMessage());
        }
    }
    
    // TCP Server example
    public static void tcpServer() {
        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            System.out.println("Server listening on port 8080");
            
            while (true) {
                Socket clientSocket = serverSocket.accept();
                
                // Handle client in separate thread
                new Thread(() -> handleClient(clientSocket)).start();
            }
            
        } catch (IOException e) {
            System.err.println("Server error: " + e.getMessage());
        }
    }
    
    private static void handleClient(Socket clientSocket) {
        try (BufferedReader in = new BufferedReader(
                new InputStreamReader(clientSocket.getInputStream()));
             PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true)) {
            
            String inputLine;
            while ((inputLine = in.readLine()) != null) {
                System.out.println("Received: " + inputLine);
                out.println("Echo: " + inputLine);
                
                if ("bye".equalsIgnoreCase(inputLine)) {
                    break;
                }
            }
            
        } catch (IOException e) {
            System.err.println("Error handling client: " + e.getMessage());
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                System.err.println("Error closing client socket: " + e.getMessage());
            }
        }
    }
}
```

### 2. UDP Programming

```java
import java.net.*;
import java.io.*;

public class UDPExample {
    
    // UDP Server
    public static class UDPServer {
        private DatagramSocket socket;
        private boolean running;
        private byte[] buffer = new byte[1024];
        
        public UDPServer(int port) throws SocketException {
            socket = new DatagramSocket(port);
        }
        
        public void start() {
            running = true;
            
            while (running) {
                try {
                    DatagramPacket packet = new DatagramPacket(
                        buffer, buffer.length);
                    socket.receive(packet);
                    
                    String received = new String(
                        packet.getData(), 0, packet.getLength());
                    System.out.println("Received: " + received);
                    
                    // Echo back
                    String response = "Echo: " + received;
                    byte[] responseData = response.getBytes();
                    
                    DatagramPacket responsePacket = new DatagramPacket(
                        responseData, responseData.length,
                        packet.getAddress(), packet.getPort());
                    
                    socket.send(responsePacket);
                    
                } catch (IOException e) {
                    if (running) {
                        System.err.println("Server error: " + e.getMessage());
                    }
                }
            }
        }
        
        public void stop() {
            running = false;
            socket.close();
        }
    }
    
    // UDP Client
    public static class UDPClient {
        private DatagramSocket socket;
        private InetAddress address;
        private int port;
        private byte[] buffer;
        
        public UDPClient(String hostname, int port) throws Exception {
            socket = new DatagramSocket();
            address = InetAddress.getByName(hostname);
            this.port = port;
        }
        
        public String sendMessage(String message) throws IOException {
            buffer = message.getBytes();
            
            DatagramPacket packet = new DatagramPacket(
                buffer, buffer.length, address, port);
            socket.send(packet);
            
            // Receive response
            packet = new DatagramPacket(new byte[1024], 1024);
            socket.receive(packet);
            
            return new String(packet.getData(), 0, packet.getLength());
        }
        
        public void close() {
            socket.close();
        }
    }
    
    public static void main(String[] args) throws Exception {
        // Start server in separate thread
        new Thread(() -> {
            try {
                UDPServer server = new UDPServer(9876);
                server.start();
            } catch (SocketException e) {
                e.printStackTrace();
            }
        }).start();
        
        // Give server time to start
        Thread.sleep(1000);
        
        // Client usage
        UDPClient client = new UDPClient("localhost", 9876);
        String response = client.sendMessage("Hello UDP Server!");
        System.out.println("Server response: " + response);
        client.close();
    }
}
```

## HTTP Programming

### 1. HTTP Client với Java 11+

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;
import java.util.concurrent.CompletableFuture;

public class ModernHTTPClient {
    
    private final HttpClient httpClient;
    
    public ModernHTTPClient() {
        this.httpClient = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(Duration.ofSeconds(10))
            .build();
    }
    
    // Synchronous GET request
    public String syncGet(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .GET()
            .uri(URI.create(url))
            .setHeader("User-Agent", "Java HTTP Client")
            .build();
        
        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString());
        
        System.out.println("Status: " + response.statusCode());
        return response.body();
    }
    
    // Asynchronous GET request
    public CompletableFuture<String> asyncGet(String url) {
        HttpRequest request = HttpRequest.newBuilder()
            .GET()
            .uri(URI.create(url))
            .setHeader("User-Agent", "Java HTTP Client")
            .build();
        
        return httpClient.sendAsync(request, 
            HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body);
    }
    
    // POST request with JSON
    public String postJson(String url, String jsonBody) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
            .uri(URI.create(url))
            .setHeader("Content-Type", "application/json")
            .setHeader("User-Agent", "Java HTTP Client")
            .build();
        
        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString());
        
        return response.body();
    }
    
    // Example usage
    public static void main(String[] args) throws Exception {
        ModernHTTPClient client = new ModernHTTPClient();
        
        // Sync request
        String result = client.syncGet("https://api.github.com/users/octocat");
        System.out.println("Sync result: " + result);
        
        // Async request
        CompletableFuture<String> asyncResult = client.asyncGet(
            "https://api.github.com/users/octocat");
        
        asyncResult.thenAccept(response -> {
            System.out.println("Async result: " + response);
        });
        
        // Wait for async to complete
        Thread.sleep(2000);
    }
}
```

### 2. Simple HTTP Server

```java
import com.sun.net.httpserver.*;
import java.net.InetSocketAddress;
import java.io.IOException;
import java.io.OutputStream;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.Executors;

public class SimpleHTTPServer {
    
    private HttpServer server;
    private final int port;
    
    public SimpleHTTPServer(int port) {
        this.port = port;
    }
    
    public void start() throws IOException {
        server = HttpServer.create(new InetSocketAddress(port), 0);
        
        // Create contexts (routes)
        server.createContext("/", new RootHandler());
        server.createContext("/api/hello", new HelloAPIHandler());
        server.createContext("/api/echo", new EchoHandler());
        server.createContext("/health", new HealthHandler());
        
        // Set executor for handling requests
        server.setExecutor(Executors.newFixedThreadPool(10));
        
        server.start();
        System.out.println("🚀 HTTP Server started on port " + port);
    }
    
    public void stop() {
        if (server != null) {
            server.stop(1);
            System.out.println("🔴 HTTP Server stopped");
        }
    }
    
    // Root handler
    static class RootHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            String response = """
                <html>
                <head><title>Simple Java HTTP Server</title></head>
                <body>
                    <h1>Welcome to Java HTTP Server!</h1>
                    <p>Available endpoints:</p>
                    <ul>
                        <li><a href="/api/hello">/api/hello</a></li>
                        <li><a href="/api/echo">/api/echo</a></li>
                        <li><a href="/health">/health</a></li>
                    </ul>
                </body>
                </html>
                """;
            
            sendResponse(exchange, 200, response, "text/html");
        }
    }
    
    // API hello handler
    static class HelloAPIHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            String method = exchange.getRequestMethod();
            
            if ("GET".equals(method)) {
                String jsonResponse = """
                    {
                        "message": "Hello from Java HTTP Server!",
                        "timestamp": "%s",
                        "method": "%s"
                    }
                    """.formatted(java.time.Instant.now(), method);
                
                sendResponse(exchange, 200, jsonResponse, "application/json");
            } else {
                sendResponse(exchange, 405, "Method not allowed", "text/plain");
            }
        }
    }
    
    // Echo handler
    static class EchoHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            String method = exchange.getRequestMethod();
            
            if ("POST".equals(method)) {
                // Read request body
                String requestBody = new String(
                    exchange.getRequestBody().readAllBytes(),
                    StandardCharsets.UTF_8);
                
                String jsonResponse = """
                    {
                        "echo": "%s",
                        "length": %d,
                        "timestamp": "%s"
                    }
                    """.formatted(requestBody, requestBody.length(), 
                                  java.time.Instant.now());
                
                sendResponse(exchange, 200, jsonResponse, "application/json");
            } else {
                sendResponse(exchange, 405, "Method not allowed", "text/plain");
            }
        }
    }
    
    // Health check handler
    static class HealthHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) throws IOException {
            String healthResponse = """
                {
                    "status": "UP",
                    "timestamp": "%s",
                    "uptime": "%d ms"
                }
                """.formatted(java.time.Instant.now(), 
                             System.currentTimeMillis());
            
            sendResponse(exchange, 200, healthResponse, "application/json");
        }
    }
    
    // Utility method to send response
    private static void sendResponse(HttpExchange exchange, int statusCode, 
                                   String response, String contentType) 
                                   throws IOException {
        exchange.getResponseHeaders().set("Content-Type", contentType);
        exchange.getResponseHeaders().set("Access-Control-Allow-Origin", "*");
        
        byte[] bytes = response.getBytes(StandardCharsets.UTF_8);
        exchange.sendResponseHeaders(statusCode, bytes.length);
        
        try (OutputStream os = exchange.getResponseBody()) {
            os.write(bytes);
        }
    }
    
    public static void main(String[] args) throws IOException {
        SimpleHTTPServer server = new SimpleHTTPServer(8080);
        
        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(server::stop));
        
        server.start();
    }
}
```

## Lập trình mạng nâng cao

### 1. Non-blocking I/O với NIO

```java
import java.nio.*;
import java.nio.channels.*;
import java.net.InetSocketAddress;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    
    private Selector selector;
    private ServerSocketChannel serverChannel;
    private final int port;
    
    public NIOServer(int port) {
        this.port = port;
    }
    
    public void start() throws Exception {
        // Initialize selector and server channel
        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.configureBlocking(false);
        
        // Register server channel with selector
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        
        System.out.println("🚀 NIO Server started on port " + port);
        
        while (true) {
            // Wait for events
            selector.select();
            
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectedKeys.iterator();
            
            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                
                if (key.isAcceptable()) {
                    handleAccept(key);
                } else if (key.isReadable()) {
                    handleRead(key);
                }
                
                iter.remove();
            }
        }
    }
    
    private void handleAccept(SelectionKey key) throws Exception {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        SocketChannel clientChannel = serverChannel.accept();
        
        if (clientChannel != null) {
            clientChannel.configureBlocking(false);
            clientChannel.register(selector, SelectionKey.OP_READ);
            
            System.out.println("✅ New client connected: " + 
                             clientChannel.getRemoteAddress());
        }
    }
    
    private void handleRead(SelectionKey key) throws Exception {
        SocketChannel clientChannel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        
        try {
            int bytesRead = clientChannel.read(buffer);
            
            if (bytesRead == -1) {
                // Client disconnected
                System.out.println("❌ Client disconnected: " + 
                                 clientChannel.getRemoteAddress());
                clientChannel.close();
                key.cancel();
                return;
            }
            
            if (bytesRead > 0) {
                buffer.flip();
                String message = new String(buffer.array(), 0, bytesRead);
                
                System.out.println("📩 Received: " + message.trim());
                
                // Echo back
                String response = "Echo: " + message;
                ByteBuffer responseBuffer = ByteBuffer.wrap(response.getBytes());
                clientChannel.write(responseBuffer);
            }
            
        } catch (Exception e) {
            System.err.println("Error reading from client: " + e.getMessage());
            clientChannel.close();
            key.cancel();
        }
    }
    
    public void stop() throws Exception {
        if (selector != null) selector.close();
        if (serverChannel != null) serverChannel.close();
        System.out.println("🔴 NIO Server stopped");
    }
    
    public static void main(String[] args) throws Exception {
        NIOServer server = new NIOServer(9090);
        
        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try {
                server.stop();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }));
        
        server.start();
    }
}
```

### 2. WebSocket Implementation

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.WebSocket;
import java.nio.ByteBuffer;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CountDownLatch;

public class WebSocketExample {
    
    // WebSocket Client
    public static class WebSocketClient {
        private WebSocket webSocket;
        private final CountDownLatch connectionLatch = new CountDownLatch(1);
        
        public void connect(String uri) throws Exception {
            HttpClient client = HttpClient.newHttpClient();
            
            WebSocket.Builder builder = client.newWebSocketBuilder();
            webSocket = builder.buildAsync(URI.create(uri), new WebSocketListener())
                              .join();
            
            connectionLatch.await(); // Wait for connection
        }
        
        public void sendMessage(String message) {
            if (webSocket != null) {
                webSocket.sendText(message, true);
            }
        }
        
        public void close() {
            if (webSocket != null) {
                webSocket.sendClose(WebSocket.NORMAL_CLOSURE, "Goodbye");
            }
        }
        
        private class WebSocketListener implements WebSocket.Listener {
            @Override
            public void onOpen(WebSocket webSocket) {
                System.out.println("🔗 WebSocket connected");
                connectionLatch.countDown();
                WebSocket.Listener.super.onOpen(webSocket);
            }
            
            @Override
            public CompletionStage<?> onText(WebSocket webSocket, 
                                           CharSequence data, boolean last) {
                System.out.println("📩 Received: " + data);
                return WebSocket.Listener.super.onText(webSocket, data, last);
            }
            
            @Override
            public CompletionStage<?> onBinary(WebSocket webSocket, 
                                             ByteBuffer data, boolean last) {
                System.out.println("📦 Received binary data: " + data.remaining() + " bytes");
                return WebSocket.Listener.super.onBinary(webSocket, data, last);
            }
            
            @Override
            public CompletionStage<?> onClose(WebSocket webSocket, 
                                            int statusCode, String reason) {
                System.out.println("❌ WebSocket closed: " + reason);
                return WebSocket.Listener.super.onClose(webSocket, statusCode, reason);
            }
            
            @Override
            public void onError(WebSocket webSocket, Throwable error) {
                System.err.println("🚨 WebSocket error: " + error.getMessage());
                WebSocket.Listener.super.onError(webSocket, error);
            }
        }
    }
    
    public static void main(String[] args) throws Exception {
        // Example WebSocket client usage
        WebSocketClient client = new WebSocketClient();
        
        // Connect to a WebSocket echo server
        client.connect("wss://echo.websocket.org");
        
        // Send some messages
        client.sendMessage("Hello WebSocket!");
        client.sendMessage("This is a test message");
        
        // Keep alive for a while
        Thread.sleep(3000);
        
        client.close();
    }
}
```

## Security trong lập trình mạng

### 1. SSL/TLS Implementation

```java
import javax.net.ssl.*;
import java.security.cert.X509Certificate;
import java.io.*;

public class SSLExample {
    
    // SSL Client
    public static class SSLClient {
        public void connectToSSLServer(String hostname, int port) throws Exception {
            // Create SSL context
            SSLContext sslContext = SSLContext.getInstance("TLS");
            sslContext.init(null, new TrustManager[]{new TrustAllCertificates()}, 
                          new java.security.SecureRandom());
            
            // Create SSL socket factory
            SSLSocketFactory factory = sslContext.getSocketFactory();
            
            try (SSLSocket socket = (SSLSocket) factory.createSocket(hostname, port)) {
                
                // Enable all supported protocols
                socket.setEnabledProtocols(socket.getSupportedProtocols());
                
                // Start handshake
                socket.startHandshake();
                
                // Get session info
                SSLSession session = socket.getSession();
                System.out.println("🔐 SSL Session Info:");
                System.out.println("Protocol: " + session.getProtocol());
                System.out.println("Cipher Suite: " + session.getCipherSuite());
                System.out.println("Peer Principal: " + session.getPeerPrincipal());
                
                // Send HTTP request
                PrintWriter out = new PrintWriter(
                    new OutputStreamWriter(socket.getOutputStream()), true);
                out.println("GET / HTTP/1.1");
                out.println("Host: " + hostname);
                out.println("Connection: close");
                out.println();
                
                // Read response
                BufferedReader in = new BufferedReader(
                    new InputStreamReader(socket.getInputStream()));
                
                String line;
                int lineCount = 0;
                while ((line = in.readLine()) != null && lineCount < 10) {
                    System.out.println(line);
                    lineCount++;
                }
            }
        }
    }
    
    // Trust all certificates (for testing only!)
    private static class TrustAllCertificates implements X509TrustManager {
        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType) {
            // Trust all client certificates
        }
        
        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType) {
            // Trust all server certificates
        }
        
        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return new X509Certificate[0];
        }
    }
    
    public static void main(String[] args) throws Exception {
        SSLClient client = new SSLClient();
        client.connectToSSLServer("www.google.com", 443);
    }
}
```

### 2. Secure Authentication

```java
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Base64;
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;

public class NetworkSecurity {
    
    // Password hashing with salt
    public static class PasswordSecurity {
        
        public static String generateSalt() {
            SecureRandom random = new SecureRandom();
            byte[] salt = new byte[16];
            random.nextBytes(salt);
            return Base64.getEncoder().encodeToString(salt);
        }
        
        public static String hashPassword(String password, String salt) throws Exception {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            md.update(Base64.getDecoder().decode(salt));
            
            byte[] hashedPassword = md.digest(password.getBytes("UTF-8"));
            return Base64.getEncoder().encodeToString(hashedPassword);
        }
        
        public static boolean verifyPassword(String password, String salt, 
                                           String hashedPassword) throws Exception {
            String newHash = hashPassword(password, salt);
            return newHash.equals(hashedPassword);
        }
    }
    
    // Simple encryption/decryption
    public static class SimpleEncryption {
        private static final String ALGORITHM = "AES";
        
        public static SecretKey generateKey() throws Exception {
            KeyGenerator keyGenerator = KeyGenerator.getInstance(ALGORITHM);
            keyGenerator.init(256);
            return keyGenerator.generateKey();
        }
        
        public static String encrypt(String data, SecretKey key) throws Exception {
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, key);
            
            byte[] encryptedData = cipher.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(encryptedData);
        }
        
        public static String decrypt(String encryptedData, SecretKey key) throws Exception {
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, key);
            
            byte[] decodedData = Base64.getDecoder().decode(encryptedData);
            byte[] decryptedData = cipher.doFinal(decodedData);
            
            return new String(decryptedData);
        }
    }
    
    // Token-based authentication
    public static class TokenAuth {
        
        public static String generateToken(String username) {
            String timestamp = String.valueOf(System.currentTimeMillis());
            String tokenData = username + ":" + timestamp;
            
            return Base64.getEncoder().encodeToString(tokenData.getBytes());
        }
        
        public static boolean validateToken(String token, long maxAge) {
            try {
                String decoded = new String(Base64.getDecoder().decode(token));
                String[] parts = decoded.split(":");
                
                if (parts.length != 2) return false;
                
                long timestamp = Long.parseLong(parts[1]);
                long currentTime = System.currentTimeMillis();
                
                return (currentTime - timestamp) <= maxAge;
                
            } catch (Exception e) {
                return false;
            }
        }
        
        public static String getUserFromToken(String token) {
            try {
                String decoded = new String(Base64.getDecoder().decode(token));
                return decoded.split(":")[0];
            } catch (Exception e) {
                return null;
            }
        }
    }
    
    public static void main(String[] args) throws Exception {
        // Password security example
        System.out.println("=== Password Security ===");
        String password = "mySecurePassword123";
        String salt = PasswordSecurity.generateSalt();
        String hashedPassword = PasswordSecurity.hashPassword(password, salt);
        
        System.out.println("Original: " + password);
        System.out.println("Salt: " + salt);
        System.out.println("Hashed: " + hashedPassword);
        System.out.println("Verified: " + 
            PasswordSecurity.verifyPassword(password, salt, hashedPassword));
        
        // Encryption example
        System.out.println("\n=== Encryption ===");
        SecretKey key = SimpleEncryption.generateKey();
        String originalData = "Sensitive network data";
        String encrypted = SimpleEncryption.encrypt(originalData, key);
        String decrypted = SimpleEncryption.decrypt(encrypted, key);
        
        System.out.println("Original: " + originalData);
        System.out.println("Encrypted: " + encrypted);
        System.out.println("Decrypted: " + decrypted);
        
        // Token authentication example
        System.out.println("\n=== Token Authentication ===");
        String token = TokenAuth.generateToken("user123");
        System.out.println("Generated token: " + token);
        System.out.println("User from token: " + TokenAuth.getUserFromToken(token));
        System.out.println("Token valid: " + TokenAuth.validateToken(token, 60000)); // 1 minute
    }
}
```

## Performance và Monitoring

### 1. Connection Pooling

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class ConnectionPool {
    
    private final BlockingQueue<Connection> pool;
    private final int maxPoolSize;
    private final String url;
    private final String username;
    private final String password;
    
    public ConnectionPool(String url, String username, String password, 
                         int maxPoolSize) throws Exception {
        this.url = url;
        this.username = username;
        this.password = password;
        this.maxPoolSize = maxPoolSize;
        this.pool = new LinkedBlockingQueue<>(maxPoolSize);
        
        // Initialize pool
        for (int i = 0; i < maxPoolSize; i++) {
            pool.offer(createConnection());
        }
    }
    
    private Connection createConnection() throws Exception {
        return DriverManager.getConnection(url, username, password);
    }
    
    public Connection getConnection() throws InterruptedException {
        return pool.poll(10, TimeUnit.SECONDS);
    }
    
    public void releaseConnection(Connection connection) {
        if (connection != null) {
            pool.offer(connection);
        }
    }
    
    public void closeAll() throws Exception {
        Connection connection;
        while ((connection = pool.poll()) != null) {
            connection.close();
        }
    }
    
    public int getAvailableConnections() {
        return pool.size();
    }
}
```

### 2. Network Monitoring

```java
import java.net.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class NetworkMonitor {
    
    private final AtomicLong totalConnections = new AtomicLong(0);
    private final AtomicLong activeConnections = new AtomicLong(0);
    private final AtomicLong totalBytesReceived = new AtomicLong(0);
    private final AtomicLong totalBytesSent = new AtomicLong(0);
    private final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(1);
    
    public void startMonitoring() {
        scheduler.scheduleAtFixedRate(this::printStats, 0, 10, TimeUnit.SECONDS);
    }
    
    public void recordConnection() {
        totalConnections.incrementAndGet();
        activeConnections.incrementAndGet();
    }
    
    public void recordDisconnection() {
        activeConnections.decrementAndGet();
    }
    
    public void recordBytesReceived(long bytes) {
        totalBytesReceived.addAndGet(bytes);
    }
    
    public void recordBytesSent(long bytes) {
        totalBytesSent.addAndGet(bytes);
    }
    
    private void printStats() {
        System.out.println("=== Network Statistics ===");
        System.out.println("Total Connections: " + totalConnections.get());
        System.out.println("Active Connections: " + activeConnections.get());
        System.out.println("Total Bytes Received: " + formatBytes(totalBytesReceived.get()));
        System.out.println("Total Bytes Sent: " + formatBytes(totalBytesSent.get()));
        System.out.println("Memory Usage: " + getMemoryUsage());
        System.out.println();
    }
    
    private String formatBytes(long bytes) {
        if (bytes < 1024) return bytes + " B";
        int exp = (int) (Math.log(bytes) / Math.log(1024));
        String pre = "KMGTPE".charAt(exp - 1) + "";
        return String.format("%.1f %sB", bytes / Math.pow(1024, exp), pre);
    }
    
    private String getMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        long used = runtime.totalMemory() - runtime.freeMemory();
        long max = runtime.maxMemory();
        return String.format("%.1f/%.1f MB (%.1f%%)", 
                           used / 1024.0 / 1024.0,
                           max / 1024.0 / 1024.0,
                           (used * 100.0) / max);
    }
    
    public void shutdown() {
        scheduler.shutdown();
    }
}
```

## Best Practices và Design Patterns

### 1. Observer Pattern cho Network Events

```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

// Event definitions
interface NetworkEvent {
    String getType();
    long getTimestamp();
}

class ConnectionEvent implements NetworkEvent {
    private final String type;
    private final String clientId;
    private final long timestamp;
    
    public ConnectionEvent(String type, String clientId) {
        this.type = type;
        this.clientId = clientId;
        this.timestamp = System.currentTimeMillis();
    }
    
    @Override
    public String getType() { return type; }
    
    @Override
    public long getTimestamp() { return timestamp; }
    
    public String getClientId() { return clientId; }
}

class MessageEvent implements NetworkEvent {
    private final String sender;
    private final String content;
    private final long timestamp;
    
    public MessageEvent(String sender, String content) {
        this.sender = sender;
        this.content = content;
        this.timestamp = System.currentTimeMillis();
    }
    
    @Override
    public String getType() { return "MESSAGE"; }
    
    @Override
    public long getTimestamp() { return timestamp; }
    
    public String getSender() { return sender; }
    public String getContent() { return content; }
}

// Observer interface
interface NetworkEventListener {
    void onNetworkEvent(NetworkEvent event);
}

// Observable network manager
class NetworkEventManager {
    private final List<NetworkEventListener> listeners = 
        new CopyOnWriteArrayList<>();
    
    public void addListener(NetworkEventListener listener) {
        listeners.add(listener);
    }
    
    public void removeListener(NetworkEventListener listener) {
        listeners.remove(listener);
    }
    
    public void fireEvent(NetworkEvent event) {
        for (NetworkEventListener listener : listeners) {
            try {
                listener.onNetworkEvent(event);
            } catch (Exception e) {
                System.err.println("Error in event listener: " + e.getMessage());
            }
        }
    }
}

// Example listeners
class LoggingListener implements NetworkEventListener {
    @Override
    public void onNetworkEvent(NetworkEvent event) {
        System.out.println("[LOG] " + event.getType() + " at " + 
                          new java.util.Date(event.getTimestamp()));
    }
}

class MetricsListener implements NetworkEventListener {
    private long messageCount = 0;
    private long connectionCount = 0;
    
    @Override
    public void onNetworkEvent(NetworkEvent event) {
        switch (event.getType()) {
            case "MESSAGE" -> messageCount++;
            case "CONNECT", "DISCONNECT" -> connectionCount++;
        }
        
        if (messageCount % 100 == 0) {
            System.out.println("[METRICS] Messages: " + messageCount + 
                             ", Connections: " + connectionCount);
        }
    }
}
```

## Kết luận và Roadmap học tập

### Roadmap học lập trình mạng

```
📚 Giai đoạn 1: Nền tảng (2-3 tháng)
├── Mô hình OSI/TCP-IP
├── Socket programming cơ bản
├── HTTP/HTTPS protocols
└── Basic security concepts

🔧 Giai đoạn 2: Thực hành (3-4 tháng)
├── Xây dựng chat application
├── RESTful API development
├── Database integration
└── Error handling & logging

⚡ Giai đoạn 3: Nâng cao (4-6 tháng)
├── Non-blocking I/O (NIO)
├── WebSocket programming
├── Microservices architecture
└── Performance optimization

🚀 Giai đoạn 4: Chuyên sâu (6+ tháng)
├── Distributed systems
├── Message queues (RabbitMQ, Kafka)
├── Load balancing & clustering
└── Container deployment (Docker, K8s)
```

### Tài nguyên học tập được khuyến nghị

**📖 Sách:**
- "Java Network Programming" - Elliotte Rusty Harold
- "TCP/IP Illustrated" - W. Richard Stevens
- "Designing Data-Intensive Applications" - Martin Kleppmann

**🌐 Online Resources:**
- Oracle Java Networking Documentation
- RFC specifications (RFC 793 for TCP, RFC 791 for IP)
- Baeldung Java tutorials

**🛠️ Tools cần thành thạo:**
- Wireshark (packet analysis)
- Postman (API testing)
- JProfiler (performance monitoring)
- Docker (containerization)

### Dự án thực tế nên làm

1. **Chat Application** - Học socket và threading
2. **HTTP Proxy Server** - Hiểu HTTP protocol
3. **File Transfer System** - Xử lý binary data
4. **REST API Gateway** - Microservices communication
5. **Real-time Game Server** - Low-latency networking

Lập trình mạng là một lĩnh vực rộng lớn và thú vị. Với nền tảng vững chắc và thực hành đều đặn, bạn sẽ có thể xây dựng những hệ thống mạng mạnh mẽ và hiệu quả!

---

*Bài viết này là phần đầu tiên trong series "Lập trình mạng với Java". Hãy theo dõi blog để cập nhật những bài viết chi tiết về các chủ đề cụ thể!*