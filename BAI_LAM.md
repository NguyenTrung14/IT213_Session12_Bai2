# Bài 2: Dò lỗi và tối ưu cấu hình Stdio cho MCP Server

## 1. Phân tích nguyên nhân kỹ thuật

MCP chạy qua Stdio sử dụng hai luồng của tiến trình với hai mục đích khác nhau:

- `System.in`: MCP Server nhận thông điệp JSON-RPC từ MCP Client.
- `System.out`: MCP Server trả thông điệp JSON-RPC cho MCP Client.

Vì vậy, `System.out` không còn là một khu vực hiển thị văn bản thông thường. Nó trở thành **kênh dữ liệu của giao thức**. Mỗi dữ liệu được ghi vào luồng này phải tuân thủ đúng định dạng và cách đóng khung thông điệp mà MCP Client mong đợi.

Khi khởi động theo cấu hình mặc định, Spring Boot có thể in banner ASCII và Logback có thể ghi log ra `System.out`. Chẳng hạn, thay vì nhận được một thông điệp hợp lệ như:

```json
{"jsonrpc":"2.0","id":1,"result":{"capabilities":{}}}
```

MCP Client có thể đọc được banner trước JSON:

```text
  .   ____          _            __ _ _
 :: Spring Boot ::
{"jsonrpc":"2.0","id":1,"result":{"capabilities":{}}}
```

Ký tự đầu tiên lúc này không phải là phần mở đầu hợp lệ của thông điệp JSON-RPC. Bộ phân tích JSON của Client gặp nội dung ASCII hoặc một dòng log như `2026-... INFO ...`, báo lỗi `Unexpected token` và không thể xác định ranh giới thông điệp. Chỉ một lần ghi ngoài giao thức vào `stdout` cũng có thể làm mất đồng bộ toàn bộ phiên giao tiếp; Client thường coi Server là lỗi và đóng kết nối.

Do đó, quy tắc quan trọng của MCP qua Stdio là:

> `stdout` chỉ dành cho dữ liệu JSON-RPC; không dùng `System.out.println`, banner, log hoặc thông tin chẩn đoán trên luồng này.

## 2. Bản vá lớp khởi động Spring Boot

Tệp `src/main/java/com/rikkei/mcp/LogisticsMcpServerApplication.java`:

```java
package com.rikkei.mcp;

import org.springframework.boot.Banner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class LogisticsMcpServerApplication {

    public static void main(String[] args) {
        SpringApplication application =
                new SpringApplication(LogisticsMcpServerApplication.class);
        application.setBannerMode(Banner.Mode.OFF);
        application.run(args);
    }
}
```

Thay vì gọi trực tiếp phương thức tĩnh `SpringApplication.run(...)`, chương trình tạo một đối tượng `SpringApplication`. Lệnh `setBannerMode(Banner.Mode.OFF)` được thực hiện trước `run(args)`, nhờ đó banner bị vô hiệu hóa ngay từ quá trình khởi động.

## 3. Bản vá cấu hình Logback

Tệp `src/main/resources/logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- STDOUT được dành riêng cho các thông điệp JSON-RPC của MCP. -->
    <appender name="STDERR" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.err</target>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <root level="DEBUG">
        <appender-ref ref="STDERR"/>
    </root>
</configuration>
```

`ConsoleAppender` được trỏ rõ ràng tới `System.err`. Root logger đặt ở mức `DEBUG`, vì thế các sự kiện `DEBUG`, `INFO`, `WARN` và `ERROR` đều được chuyển tới appender này. Không có appender nào trỏ tới `System.out`.

## 4. Tại sao `System.err` không làm hỏng JSON-RPC?

Hệ điều hành cung cấp cho mỗi tiến trình các luồng độc lập:

- Standard Input (`stdin`)
- Standard Output (`stdout`)
- Standard Error (`stderr`)

MCP Client trao đổi dữ liệu giao thức với Server qua `stdin` và `stdout`, nhưng có thể thu thập hoặc hiển thị `stderr` như một kênh chẩn đoán riêng. Vì dữ liệu của hai luồng không bị trộn vào nhau, các dòng log trên `stderr` không được đưa vào bộ phân tích JSON-RPC đang đọc `stdout`.

Nhờ đó, lập trình viên vẫn có thể xem thời điểm, mức log, luồng thực thi, logger và nội dung lỗi trong terminal hoặc nhật ký của Claude Desktop, trong khi `stdout` luôn sạch và chỉ chứa thông điệp MCP hợp lệ.

## 5. Lưu ý khi phát triển

- Không gọi `System.out.print` hoặc `System.out.println` trong bất kỳ mã ứng dụng, thư viện hay đoạn gỡ lỗi nào.
- Dùng SLF4J/Logback cho log để cấu hình `logback-spring.xml` kiểm soát đúng luồng đích.
- Kiểm tra cả thư viện phụ thuộc vì một thư viện tự ghi trực tiếp ra `System.out` vẫn có thể gây ô nhiễm giao thức.
- Có thể giảm root logger xuống `INFO` trong môi trường vận hành nếu lượng log `DEBUG` quá lớn; việc này không thay đổi nguyên tắc tất cả log phải đi qua `stderr`.

## Kết luận

Bản vá tách biệt hoàn toàn hai loại dữ liệu: JSON-RPC đi qua `stdout`, còn thông tin chẩn đoán đi qua `stderr`. Việc tắt banner và cấu hình duy nhất một appender `System.err` loại bỏ hai nguồn gây ô nhiễm Stdio phổ biến nhất của ứng dụng Spring Boot.
