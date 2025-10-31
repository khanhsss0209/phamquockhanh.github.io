---
title: "Xây dựng ứng dụng Chat đa người dùng với Java Socket - Hướng dẫn chi tiết"
date: 2025-10-23
draft: false
tags: ["Java", "Socket", "Chat", "Networking", "Multi-threading", "Programming"]
description: "Hướng dẫn chi tiết xây dựng ứng dụng chat đa người dùng sử dụng Java Socket với multi-threading, GUI và các tính năng nâng cao"
---

## Giới thiệu

Lập trình mạng với Java Socket là một kỹ năng quan trọng giúp chúng ta hiểu cách các ứng dụng giao tiếp qua mạng. Trong bài viết này, chúng ta sẽ xây dựng một ứng dụng chat đa người dùng hoàn chỉnh từ cơ bản đến nâng cao.

## Kiến thức cần có

Trước khi bắt đầu, bạn nên có kiến thức về:
- **Java cơ bản**: OOP, Exception handling
- **Multi-threading**: Thread, Runnable interface
- **Java Swing**: GUI components
- **TCP/IP**: Hiểu cơ bản về giao thức mạng

## Kiến trúc tổng quan

### Mô hình Client-Server

```
[Client 1] ----\
                \
[Client 2] ----> [Server] ----> [Clients Management]
                /                      |
[Client 3] ----/                [Message Broadcasting]
```

### Các thành phần chính

1. **Server**: Quản lý kết nối và broadcast tin nhắn
2. **Client**: Giao diện người dùng và gửi/nhận tin nhắn
3. **Protocol**: Định nghĩa format tin nhắn
4. **Threading**: Xử lý đồng thời nhiều client

## Phát triển Server

### 1. ChatServer - Lớp chính

```java
package com.chatapp.server;

import java.io.*;
import java.net.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.logging.*;

public class ChatServer {
    private static final Logger LOGGER = Logger.getLogger(ChatServer.class.getName());
    private static final int PORT = 12345;
    private static final int MAX_CLIENTS = 50;
    
    private ServerSocket serverSocket;
    private volatile boolean running = false;
    private ExecutorService clientPool;
    private Map<String, ClientHandler> connectedClients;
    private List<String> chatHistory;
    
    public ChatServer() {
        this.clientPool = Executors.newFixedThreadPool(MAX_CLIENTS);
        this.connectedClients = new ConcurrentHashMap<>();
        this.chatHistory = new ArrayList<>();
        setupLogging();
    }
    
    private void setupLogging() {
        try {
            FileHandler fileHandler = new FileHandler("server.log", true);
            fileHandler.setFormatter(new SimpleFormatter());
            LOGGER.addHandler(fileHandler);
            LOGGER.setLevel(Level.INFO);
        } catch (IOException e) {
            System.err.println("Could not setup logging: " + e.getMessage());
        }
    }
    
    public void start() {
        try {
            serverSocket = new ServerSocket(PORT);
            running = true;
            
            LOGGER.info("Chat Server started on port " + PORT);
            System.out.println("🚀 Chat Server is running on port " + PORT);
            System.out.println("Waiting for clients to connect...");
            
            // Accept client connections
            while (running) {
                try {
                    Socket clientSocket = serverSocket.accept();
                    
                    if (connectedClients.size() >= MAX_CLIENTS) {
                        rejectClient(clientSocket, "Server is full");
                        continue;
                    }
                    
                    ClientHandler clientHandler = new ClientHandler(clientSocket, this);
                    clientPool.submit(clientHandler);
                    
                } catch (IOException e) {
                    if (running) {
                        LOGGER.warning("Error accepting client connection: " + e.getMessage());
                    }
                }
            }
        } catch (IOException e) {
            LOGGER.severe("Could not start server: " + e.getMessage());
            System.err.println("❌ Could not start server: " + e.getMessage());
        }
    }
    
    private void rejectClient(Socket clientSocket, String reason) {
        try (PrintWriter out = new PrintWriter(
                clientSocket.getOutputStream(), true)) {
            out.println("REJECTED:" + reason);
            clientSocket.close();
            LOGGER.info("Client rejected: " + reason);
        } catch (IOException e) {
            LOGGER.warning("Error rejecting client: " + e.getMessage());
        }
    }
    
    public synchronized void addClient(String username, ClientHandler handler) {
        if (connectedClients.containsKey(username)) {
            handler.sendMessage("ERROR:Username already taken");
            return;
        }
        
        connectedClients.put(username, handler);
        
        // Send welcome message
        handler.sendMessage("WELCOME:Welcome to the chat, " + username + "!");
        
        // Send chat history
        sendChatHistory(handler);
        
        // Send current users list
        sendUsersList(handler);
        
        // Notify all clients about new user
        broadcastMessage("USER_JOINED:" + username + " joined the chat", null);
        broadcastUsersList();
        
        LOGGER.info("User " + username + " connected. Total users: " + 
                   connectedClients.size());
        System.out.println("✅ " + username + " joined. Active users: " + 
                          connectedClients.size());
    }
    
    public synchronized void removeClient(String username) {
        ClientHandler removed = connectedClients.remove(username);
        if (removed != null) {
            broadcastMessage("USER_LEFT:" + username + " left the chat", null);
            broadcastUsersList();
            
            LOGGER.info("User " + username + " disconnected. Total users: " + 
                       connectedClients.size());
            System.out.println("❌ " + username + " left. Active users: " + 
                              connectedClients.size());
        }
    }
    
    public synchronized void broadcastMessage(String message, String sender) {
        // Add to history (except system messages)
        if (sender != null) {
            String timestamp = new Date().toString();
            String historyEntry = "[" + timestamp + "] " + sender + ": " + message;
            chatHistory.add(historyEntry);
            
            // Keep only last 100 messages
            if (chatHistory.size() > 100) {
                chatHistory.remove(0);
            }
        }
        
        // Broadcast to all clients
        String fullMessage = sender != null ? 
            "MESSAGE:" + sender + ":" + message : message;
            
        List<String> disconnectedUsers = new ArrayList<>();
        
        for (Map.Entry<String, ClientHandler> entry : connectedClients.entrySet()) {
            if (!entry.getValue().sendMessage(fullMessage)) {
                disconnectedUsers.add(entry.getKey());
            }
        }
        
        // Remove disconnected users
        for (String user : disconnectedUsers) {
            removeClient(user);
        }
        
        if (sender != null) {
            LOGGER.info("Broadcast message from " + sender + ": " + message);
        }
    }
    
    public synchronized void sendPrivateMessage(String from, String to, String message) {
        ClientHandler recipient = connectedClients.get(to);
        ClientHandler sender = connectedClients.get(from);
        
        if (recipient != null) {
            recipient.sendMessage("PRIVATE:" + from + ":" + message);
            sender.sendMessage("PRIVATE_SENT:" + to + ":" + message);
            
            LOGGER.info("Private message from " + from + " to " + to + ": " + message);
        } else {
            sender.sendMessage("ERROR:User " + to + " not found");
        }
    }
    
    private void sendChatHistory(ClientHandler handler) {
        for (String message : chatHistory) {
            handler.sendMessage("HISTORY:" + message);
        }
    }
    
    private void sendUsersList(ClientHandler handler) {
        StringBuilder users = new StringBuilder("USERS:");
        for (String username : connectedClients.keySet()) {
            users.append(username).append(",");
        }
        handler.sendMessage(users.toString());
    }
    
    private void broadcastUsersList() {
        StringBuilder users = new StringBuilder("USERS:");
        for (String username : connectedClients.keySet()) {
            users.append(username).append(",");
        }
        
        String usersList = users.toString();
        for (ClientHandler handler : connectedClients.values()) {
            handler.sendMessage(usersList);
        }
    }
    
    public void stop() {
        running = false;
        
        try {
            // Notify all clients
            broadcastMessage("SERVER_SHUTDOWN:Server is shutting down", null);
            
            // Close all client connections
            for (ClientHandler handler : connectedClients.values()) {
                handler.close();
            }
            
            // Shutdown thread pool
            clientPool.shutdown();
            if (!clientPool.awaitTermination(5, TimeUnit.SECONDS)) {
                clientPool.shutdownNow();
            }
            
            // Close server socket
            if (serverSocket != null && !serverSocket.isClosed()) {
                serverSocket.close();
            }
            
            LOGGER.info("Chat Server stopped");
            System.out.println("🔴 Chat Server stopped");
            
        } catch (IOException | InterruptedException e) {
            LOGGER.severe("Error stopping server: " + e.getMessage());
        }
    }
    
    public static void main(String[] args) {
        ChatServer server = new ChatServer();
        
        // Graceful shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(server::stop));
        
        server.start();
    }
}
```

### 2. ClientHandler - Xử lý từng client

```java
package com.chatapp.server;

import java.io.*;
import java.net.*;
import java.util.logging.Logger;

public class ClientHandler implements Runnable {
    private static final Logger LOGGER = Logger.getLogger(ClientHandler.class.getName());
    
    private Socket clientSocket;
    private ChatServer server;
    private BufferedReader in;
    private PrintWriter out;
    private String username;
    private volatile boolean running = true;
    
    public ClientHandler(Socket socket, ChatServer server) {
        this.clientSocket = socket;
        this.server = server;
    }
    
    @Override
    public void run() {
        try {
            setupStreams();
            handleClientCommunication();
        } catch (IOException e) {
            LOGGER.warning("Client communication error: " + e.getMessage());
        } finally {
            cleanup();
        }
    }
    
    private void setupStreams() throws IOException {
        in = new BufferedReader(new InputStreamReader(
            clientSocket.getInputStream()));
        out = new PrintWriter(clientSocket.getOutputStream(), true);
    }
    
    private void handleClientCommunication() throws IOException {
        // Wait for username
        String firstMessage = in.readLine();
        if (firstMessage == null || !firstMessage.startsWith("USERNAME:")) {
            sendMessage("ERROR:Please provide username");
            return;
        }
        
        username = firstMessage.substring(9).trim();
        
        if (username.isEmpty() || username.contains(":") || username.contains(",")) {
            sendMessage("ERROR:Invalid username format");
            return;
        }
        
        // Add to server
        server.addClient(username, this);
        
        // Listen for messages
        String inputLine;
        while (running && (inputLine = in.readLine()) != null) {
            processMessage(inputLine);
        }
    }
    
    private void processMessage(String message) {
        try {
            if (message.startsWith("MESSAGE:")) {
                String content = message.substring(8);
                server.broadcastMessage(content, username);
                
            } else if (message.startsWith("PRIVATE:")) {
                String[] parts = message.substring(8).split(":", 2);
                if (parts.length == 2) {
                    String recipient = parts[0];
                    String content = parts[1];
                    server.sendPrivateMessage(username, recipient, content);
                }
                
            } else if (message.equals("DISCONNECT")) {
                running = false;
                
            } else {
                sendMessage("ERROR:Unknown command format");
            }
        } catch (Exception e) {
            LOGGER.warning("Error processing message from " + username + ": " + 
                          e.getMessage());
        }
    }
    
    public boolean sendMessage(String message) {
        try {
            out.println(message);
            return !out.checkError();
        } catch (Exception e) {
            LOGGER.warning("Error sending message to " + username + ": " + 
                          e.getMessage());
            return false;
        }
    }
    
    public void close() {
        running = false;
        cleanup();
    }
    
    private void cleanup() {
        try {
            if (username != null) {
                server.removeClient(username);
            }
            
            if (in != null) in.close();
            if (out != null) out.close();
            if (clientSocket != null && !clientSocket.isClosed()) {
                clientSocket.close();
            }
        } catch (IOException e) {
            LOGGER.warning("Error during cleanup: " + e.getMessage());
        }
    }
    
    public String getUsername() {
        return username;
    }
}
```

## Phát triển Client

### 1. ChatClient - Lớp kết nối

```java
package com.chatapp.client;

import javax.swing.*;
import java.io.*;
import java.net.*;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class ChatClient {
    private static final String SERVER_HOST = "localhost";
    private static final int SERVER_PORT = 12345;
    
    private Socket socket;
    private BufferedReader in;
    private PrintWriter out;
    private String username;
    private ChatGUI gui;
    private volatile boolean connected = false;
    private Thread messageListener;
    private BlockingQueue<String> messageQueue;
    
    public ChatClient() {
        this.messageQueue = new LinkedBlockingQueue<>();
    }
    
    public boolean connect(String username) {
        this.username = username;
        
        try {
            socket = new Socket(SERVER_HOST, SERVER_PORT);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(), true);
            
            // Send username
            out.println("USERNAME:" + username);
            
            // Wait for server response
            String response = in.readLine();
            if (response != null && response.startsWith("ERROR:")) {
                JOptionPane.showMessageDialog(null, 
                    response.substring(6), "Connection Error", 
                    JOptionPane.ERROR_MESSAGE);
                disconnect();
                return false;
            }
            
            connected = true;
            startMessageListener();
            return true;
            
        } catch (IOException e) {
            JOptionPane.showMessageDialog(null, 
                "Could not connect to server: " + e.getMessage(), 
                "Connection Error", JOptionPane.ERROR_MESSAGE);
            return false;
        }
    }
    
    private void startMessageListener() {
        messageListener = new Thread(() -> {
            try {
                String message;
                while (connected && (message = in.readLine()) != null) {
                    messageQueue.offer(message);
                    
                    SwingUtilities.invokeLater(() -> {
                        processServerMessage(messageQueue.poll());
                    });
                }
            } catch (IOException e) {
                if (connected) {
                    SwingUtilities.invokeLater(() -> {
                        gui.showSystemMessage("❌ Connection lost: " + e.getMessage());
                        disconnect();
                    });
                }
            }
        });
        messageListener.start();
    }
    
    private void processServerMessage(String message) {
        if (message == null) return;
        
        try {
            if (message.startsWith("WELCOME:")) {
                gui.showSystemMessage("🎉 " + message.substring(8));
                
            } else if (message.startsWith("MESSAGE:")) {
                String[] parts = message.substring(8).split(":", 2);
                if (parts.length == 2) {
                    gui.displayMessage(parts[0], parts[1]);
                }
                
            } else if (message.startsWith("PRIVATE:")) {
                String[] parts = message.substring(8).split(":", 2);
                if (parts.length == 2) {
                    gui.displayPrivateMessage(parts[0], parts[1], false);
                }
                
            } else if (message.startsWith("PRIVATE_SENT:")) {
                String[] parts = message.substring(13).split(":", 2);
                if (parts.length == 2) {
                    gui.displayPrivateMessage(parts[0], parts[1], true);
                }
                
            } else if (message.startsWith("USER_JOINED:")) {
                gui.showSystemMessage("✅ " + message.substring(12));
                
            } else if (message.startsWith("USER_LEFT:")) {
                gui.showSystemMessage("❌ " + message.substring(10));
                
            } else if (message.startsWith("USERS:")) {
                String usersList = message.substring(6);
                if (!usersList.isEmpty()) {
                    String[] users = usersList.split(",");
                    gui.updateUsersList(users);
                }
                
            } else if (message.startsWith("HISTORY:")) {
                gui.displayHistoryMessage(message.substring(8));
                
            } else if (message.startsWith("ERROR:")) {
                gui.showSystemMessage("⚠️ " + message.substring(6));
                
            } else if (message.startsWith("SERVER_SHUTDOWN:")) {
                gui.showSystemMessage("🔴 " + message.substring(16));
                disconnect();
                
            } else if (message.startsWith("REJECTED:")) {
                JOptionPane.showMessageDialog(gui, 
                    message.substring(9), "Connection Rejected", 
                    JOptionPane.ERROR_MESSAGE);
                disconnect();
            }
        } catch (Exception e) {
            gui.showSystemMessage("⚠️ Error processing message: " + e.getMessage());
        }
    }
    
    public void sendMessage(String message) {
        if (connected && out != null) {
            out.println("MESSAGE:" + message);
        }
    }
    
    public void sendPrivateMessage(String recipient, String message) {
        if (connected && out != null) {
            out.println("PRIVATE:" + recipient + ":" + message);
        }
    }
    
    public void disconnect() {
        connected = false;
        
        try {
            if (out != null) {
                out.println("DISCONNECT");
            }
            
            if (messageListener != null && messageListener.isAlive()) {
                messageListener.interrupt();
            }
            
            if (in != null) in.close();
            if (out != null) out.close();
            if (socket != null && !socket.isClosed()) {
                socket.close();
            }
            
            if (gui != null) {
                gui.onDisconnected();
            }
            
        } catch (IOException e) {
            System.err.println("Error during disconnect: " + e.getMessage());
        }
    }
    
    public boolean isConnected() {
        return connected;
    }
    
    public String getUsername() {
        return username;
    }
    
    public void setGUI(ChatGUI gui) {
        this.gui = gui;
    }
}
```

### 2. ChatGUI - Giao diện người dùng

```java
package com.chatapp.client;

import javax.swing.*;
import javax.swing.border.EmptyBorder;
import javax.swing.text.*;
import java.awt.*;
import java.awt.event.*;
import java.text.SimpleDateFormat;
import java.util.Date;

public class ChatGUI extends JFrame {
    private ChatClient client;
    private JTextPane chatArea;
    private JTextField messageField;
    private JButton sendButton;
    private JList<String> usersList;
    private DefaultListModel<String> usersModel;
    private JButton connectButton;
    private JButton disconnectButton;
    private JTextField usernameField;
    private SimpleDateFormat timeFormat;
    private StyledDocument chatDocument;
    
    // Styles for different message types
    private Style defaultStyle;
    private Style systemStyle;
    private Style userStyle;
    private Style privateStyle;
    private Style timestampStyle;
    
    public ChatGUI() {
        this.client = new ChatClient();
        this.timeFormat = new SimpleDateFormat("HH:mm:ss");
        
        client.setGUI(this);
        initializeComponents();
        setupStyles();
        setupLayout();
        setupEventHandlers();
        
        setTitle("Java Chat Client");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(800, 600);
        setLocationRelativeTo(null);
        
        // Window close handler
        addWindowListener(new WindowAdapter() {
            @Override
            public void windowClosing(WindowEvent e) {
                if (client.isConnected()) {
                    client.disconnect();
                }
                System.exit(0);
            }
        });
    }
    
    private void initializeComponents() {
        // Chat area
        chatArea = new JTextPane();
        chatArea.setEditable(false);
        chatArea.setBackground(new Color(248, 249, 250));
        chatDocument = chatArea.getStyledDocument();
        
        // Message input
        messageField = new JTextField();
        sendButton = new JButton("Send");
        sendButton.setPreferredSize(new Dimension(80, 30));
        
        // Users list
        usersModel = new DefaultListModel<>();
        usersList = new JList<>(usersModel);
        usersList.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        usersList.setBorder(BorderFactory.createTitledBorder("Online Users"));
        
        // Connection controls
        usernameField = new JTextField("User" + (int)(Math.random() * 1000));
        connectButton = new JButton("Connect");
        disconnectButton = new JButton("Disconnect");
        disconnectButton.setEnabled(false);
    }
    
    private void setupStyles() {
        // Default style
        defaultStyle = chatDocument.addStyle("default", null);
        StyleConstants.setFontFamily(defaultStyle, "SansSerif");
        StyleConstants.setFontSize(defaultStyle, 12);
        StyleConstants.setForeground(defaultStyle, Color.BLACK);
        
        // System messages style
        systemStyle = chatDocument.addStyle("system", defaultStyle);
        StyleConstants.setForeground(systemStyle, new Color(128, 128, 128));
        StyleConstants.setItalic(systemStyle, true);
        
        // User messages style
        userStyle = chatDocument.addStyle("user", defaultStyle);
        StyleConstants.setForeground(userStyle, new Color(37, 99, 235));
        StyleConstants.setBold(userStyle, true);
        
        // Private messages style
        privateStyle = chatDocument.addStyle("private", defaultStyle);
        StyleConstants.setForeground(privateStyle, new Color(139, 69, 19));
        StyleConstants.setItalic(privateStyle, true);
        
        // Timestamp style
        timestampStyle = chatDocument.addStyle("timestamp", defaultStyle);
        StyleConstants.setForeground(timestampStyle, new Color(156, 163, 175));
        StyleConstants.setFontSize(timestampStyle, 10);
    }
    
    private void setupLayout() {
        setLayout(new BorderLayout());
        
        // Top panel - Connection controls
        JPanel topPanel = new JPanel(new FlowLayout(FlowLayout.LEFT));
        topPanel.setBorder(new EmptyBorder(5, 5, 5, 5));
        topPanel.add(new JLabel("Username:"));
        topPanel.add(usernameField);
        topPanel.add(connectButton);
        topPanel.add(disconnectButton);
        add(topPanel, BorderLayout.NORTH);
        
        // Center panel - Chat area
        JScrollPane chatScrollPane = new JScrollPane(chatArea);
        chatScrollPane.setVerticalScrollBarPolicy(JScrollPane.VERTICAL_SCROLLBAR_ALWAYS);
        chatScrollPane.setPreferredSize(new Dimension(500, 400));
        
        // Right panel - Users list
        JScrollPane usersScrollPane = new JScrollPane(usersList);
        usersScrollPane.setPreferredSize(new Dimension(150, 400));
        
        // Split pane for chat and users
        JSplitPane splitPane = new JSplitPane(JSplitPane.HORIZONTAL_SPLIT, 
                                            chatScrollPane, usersScrollPane);
        splitPane.setDividerLocation(550);
        add(splitPane, BorderLayout.CENTER);
        
        // Bottom panel - Message input
        JPanel bottomPanel = new JPanel(new BorderLayout());
        bottomPanel.setBorder(new EmptyBorder(5, 5, 5, 5));
        bottomPanel.add(messageField, BorderLayout.CENTER);
        bottomPanel.add(sendButton, BorderLayout.EAST);
        add(bottomPanel, BorderLayout.SOUTH);
    }
    
    private void setupEventHandlers() {
        // Connect button
        connectButton.addActionListener(e -> {
            String username = usernameField.getText().trim();
            if (username.isEmpty()) {
                JOptionPane.showMessageDialog(this, 
                    "Please enter a username", "Error", 
                    JOptionPane.ERROR_MESSAGE);
                return;
            }
            
            if (client.connect(username)) {
                connectButton.setEnabled(false);
                disconnectButton.setEnabled(true);
                usernameField.setEnabled(false);
                messageField.requestFocus();
            }
        });
        
        // Disconnect button
        disconnectButton.addActionListener(e -> {
            client.disconnect();
        });
        
        // Send button
        sendButton.addActionListener(e -> sendMessage());
        
        // Enter key in message field
        messageField.addActionListener(e -> sendMessage());
        
        // Double-click on user for private message
        usersList.addMouseListener(new MouseAdapter() {
            @Override
            public void mouseClicked(MouseEvent e) {
                if (e.getClickCount() == 2) {
                    String selectedUser = usersList.getSelectedValue();
                    if (selectedUser != null && !selectedUser.equals(client.getUsername())) {
                        String message = JOptionPane.showInputDialog(
                            ChatGUI.this, 
                            "Private message to " + selectedUser + ":",
                            "Private Message",
                            JOptionPane.PLAIN_MESSAGE);
                        
                        if (message != null && !message.trim().isEmpty()) {
                            client.sendPrivateMessage(selectedUser, message.trim());
                        }
                    }
                }
            }
        });
    }
    
    private void sendMessage() {
        String message = messageField.getText().trim();
        if (!message.isEmpty() && client.isConnected()) {
            client.sendMessage(message);
            messageField.setText("");
        }
    }
    
    public void displayMessage(String sender, String message) {
        SwingUtilities.invokeLater(() -> {
            try {
                String timestamp = timeFormat.format(new Date());
                
                // Add timestamp
                chatDocument.insertString(chatDocument.getLength(), 
                    "[" + timestamp + "] ", timestampStyle);
                
                // Add sender name
                chatDocument.insertString(chatDocument.getLength(), 
                    sender + ": ", userStyle);
                
                // Add message
                chatDocument.insertString(chatDocument.getLength(), 
                    message + "\n", defaultStyle);
                
                scrollToBottom();
            } catch (BadLocationException e) {
                e.printStackTrace();
            }
        });
    }
    
    public void displayPrivateMessage(String otherUser, String message, boolean sent) {
        SwingUtilities.invokeLater(() -> {
            try {
                String timestamp = timeFormat.format(new Date());
                String prefix = sent ? "To " + otherUser : "From " + otherUser;
                
                // Add timestamp
                chatDocument.insertString(chatDocument.getLength(), 
                    "[" + timestamp + "] ", timestampStyle);
                
                // Add private message indicator
                chatDocument.insertString(chatDocument.getLength(), 
                    "[PRIVATE] " + prefix + ": ", privateStyle);
                
                // Add message
                chatDocument.insertString(chatDocument.getLength(), 
                    message + "\n", defaultStyle);
                
                scrollToBottom();
            } catch (BadLocationException e) {
                e.printStackTrace();
            }
        });
    }
    
    public void displayHistoryMessage(String message) {
        SwingUtilities.invokeLater(() -> {
            try {
                chatDocument.insertString(chatDocument.getLength(), 
                    message + "\n", systemStyle);
                scrollToBottom();
            } catch (BadLocationException e) {
                e.printStackTrace();
            }
        });
    }
    
    public void showSystemMessage(String message) {
        SwingUtilities.invokeLater(() -> {
            try {
                String timestamp = timeFormat.format(new Date());
                
                chatDocument.insertString(chatDocument.getLength(), 
                    "[" + timestamp + "] " + message + "\n", systemStyle);
                
                scrollToBottom();
            } catch (BadLocationException e) {
                e.printStackTrace();
            }
        });
    }
    
    public void updateUsersList(String[] users) {
        SwingUtilities.invokeLater(() -> {
            usersModel.clear();
            for (String user : users) {
                if (!user.trim().isEmpty()) {
                    usersModel.addElement(user.trim());
                }
            }
        });
    }
    
    public void onDisconnected() {
        SwingUtilities.invokeLater(() -> {
            connectButton.setEnabled(true);
            disconnectButton.setEnabled(false);
            usernameField.setEnabled(true);
            usersModel.clear();
            showSystemMessage("🔴 Disconnected from server");
        });
    }
    
    private void scrollToBottom() {
        SwingUtilities.invokeLater(() -> {
            chatArea.setCaretPosition(chatDocument.getLength());
        });
    }
    
    public static void main(String[] args) {
        // Set system look and feel
        try {
            UIManager.setLookAndFeel(UIManager.getSystemLookAndFeel());
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        SwingUtilities.invokeLater(() -> {
            new ChatGUI().setVisible(true);
        });
    }
}
```

## Tính năng nâng cao

### 1. Emoji và Formatting

```java
// Trong ChatGUI, thêm method để xử lý emoji
private String processEmojis(String message) {
    return message
        .replace(":)", "😊")
        .replace(":(", "😢")
        .replace(":D", "😃")
        .replace(";)", "😉")
        .replace("<3", "❤️")
        .replace(":p", "😛");
}
```

### 2. File Transfer

```java
// Thêm vào ClientHandler
private void handleFileTransfer(String message) {
    // FILE_TRANSFER:recipient:filename:size
    String[] parts = message.split(":", 4);
    if (parts.length >= 4) {
        String recipient = parts[1];
        String filename = parts[2];
        long fileSize = Long.parseLong(parts[3]);
        
        // Implement file transfer logic
        transferFile(recipient, filename, fileSize);
    }
}
```

### 3. Room/Channel System

```java
// Thêm vào ChatServer
private Map<String, Set<String>> chatRooms = new ConcurrentHashMap<>();

public void joinRoom(String username, String roomName) {
    chatRooms.computeIfAbsent(roomName, k -> new HashSet<>())
             .add(username);
    
    broadcastToRoom(roomName, "USER_JOINED_ROOM:" + username + 
                   " joined room " + roomName, null);
}

public void broadcastToRoom(String roomName, String message, String sender) {
    Set<String> roomUsers = chatRooms.get(roomName);
    if (roomUsers != null) {
        for (String username : roomUsers) {
            ClientHandler handler = connectedClients.get(username);
            if (handler != null) {
                handler.sendMessage("ROOM_MESSAGE:" + roomName + ":" + 
                                   sender + ":" + message);
            }
        }
    }
}
```

## Testing và Deployment

### 1. Unit Tests

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class ChatServerTest {
    
    @Test
    public void testServerStartStop() {
        ChatServer server = new ChatServer();
        
        // Test server start
        new Thread(server::start).start();
        
        // Wait a bit for server to start
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // Test connection
        try (Socket testSocket = new Socket("localhost", 12345)) {
            assertTrue(testSocket.isConnected());
        } catch (IOException e) {
            fail("Could not connect to server");
        }
        
        // Test server stop
        server.stop();
    }
    
    @Test
    public void testMessageBroadcast() {
        // Implement broadcast testing
    }
}
```

### 2. Dockerfile cho deployment

```dockerfile
FROM openjdk:11-jre-slim

WORKDIR /app

COPY target/chat-server.jar .

EXPOSE 12345

CMD ["java", "-jar", "chat-server.jar"]
```

## Performance và Optimization

### 1. Connection Pool Tuning

```java
// Trong ChatServer constructor
public ChatServer() {
    // Optimize thread pool based on expected load
    int corePoolSize = Runtime.getRuntime().availableProcessors();
    int maxPoolSize = Math.max(MAX_CLIENTS, corePoolSize * 2);
    
    this.clientPool = new ThreadPoolExecutor(
        corePoolSize,
        maxPoolSize,
        60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(MAX_CLIENTS),
        new ThreadFactory() {
            private int counter = 0;
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "ChatClient-" + counter++);
                t.setDaemon(true);
                return t;
            }
        },
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
}
```

### 2. Memory Management

```java
// Định kỳ dọn dẹp chat history
private void cleanupOldMessages() {
    ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    scheduler.scheduleAtFixedRate(() -> {
        synchronized(chatHistory) {
            if (chatHistory.size() > MAX_HISTORY_SIZE) {
                int removeCount = chatHistory.size() - MAX_HISTORY_SIZE;
                for (int i = 0; i < removeCount; i++) {
                    chatHistory.remove(0);
                }
            }
        }
    }, 1, 1, TimeUnit.HOURS);
}
```

## Kết luận

Trong bài viết này, chúng ta đã xây dựng một ứng dụng chat đa người dùng hoàn chỉnh với Java Socket. Ứng dụng bao gồm:

### Tính năng đã triển khai:
- ✅ **Multi-threaded server** xử lý nhiều client đồng thời
- ✅ **Real-time messaging** với broadcast và private message
- ✅ **User management** với danh sách người dùng online
- ✅ **Chat history** lưu trữ tin nhắn gần đây
- ✅ **GUI client** với Swing interface
- ✅ **Error handling** và logging system
- ✅ **Graceful shutdown** và resource cleanup

### Điểm mạnh của giải pháp:
- **Scalable**: Sử dụng thread pool để tối ưu hiệu suất
- **Robust**: Xử lý lỗi và disconnection tốt
- **User-friendly**: GUI trực quan và dễ sử dụng
- **Extensible**: Dễ dàng thêm tính năng mới

### Hướng phát triển tiếp theo:
- 🔄 **Database integration** cho persistent storage
- 🔐 **Authentication và authorization**
- 📁 **File sharing** capabilities
- 🏠 **Room/Channel system**
- 🎨 **Theme và customization**
- 📱 **Mobile client** development

Đây là một project tuyệt vời để học về network programming, multi-threading, và GUI development trong Java!

---

*Bài viết này là một phần trong series "Lập trình mạng với Java". Source code đầy đủ có thể tìm thấy trên [GitHub](https://github.com/khanhsss0209/java-chat-app).*