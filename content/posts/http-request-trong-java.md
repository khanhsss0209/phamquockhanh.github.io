---
title: "HTTP Request trong Java"
date: 2025-10-23
draft: false
tags: ["Java", "HTTP", "Networking"]
---

Java cung cấp sẵn lớp `HttpURLConnection` để gửi request đến server web.  
Ví dụ:

```java
import java.net.*;
import java.io.*;

public class GetExample {
    public static void main(String[] args) throws Exception {
        URL url = new URL("https://jsonplaceholder.typicode.com/posts/1");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestMethod("GET");

        BufferedReader in = new BufferedReader(new InputStreamReader(con.getInputStream()));
        String input;
        while ((input = in.readLine()) != null) {
            System.out.println(input);
        }
        in.close();
    }
}
Đoạn code trên minh họa cách Java gửi yêu cầu HTTP GET và nhận phản hồi JSON.