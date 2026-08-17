Bài 2: Đọc hiểu & Dò lỗi — Endpoint Stream “giả SSE”
1. Nguyên nhân lỗi cốt lõi

Đoạn code ban đầu:

@GetMapping("/api/v1/ai/stream")
public Flux<String> getStreamResponse(@RequestParam String message) {
    return chatModel.stream(new Prompt(message))
            .map(response -> response.getResult().getOutput().getText());
}
Lỗi chính

Endpoint trả về Flux<String> nhưng chưa khai báo rõ kiểu dữ liệu SSE.

Cụ thể, @GetMapping đang thiếu:

produces = MediaType.TEXT_EVENT_STREAM_VALUE

Nên Spring không được yêu cầu rõ ràng phải serialize từng phần tử của Flux theo định dạng Server-Sent Events.

Cần sửa thành:

@GetMapping(
    value = "/api/v1/ai/stream",
    produces = MediaType.TEXT_EVENT_STREAM_VALUE
)
Một vấn đề khác

Đoạn:

.map(response -> response.getResult().getOutput().getText())

cũng có thể gây lỗi nếu một chunk từ model không có nội dung text hoặc một object trung gian là null.

Nên kiểm tra dữ liệu trước khi lấy text.

2. SSE hoạt động như thế nào trong Spring WebFlux?

SSE = Server-Sent Events.

Đây là cơ chế cho phép:

Client
   │
   │ HTTP GET /api/v1/ai/stream
   ▼
Spring WebFlux
   │
   │ Flux<String>
   ▼
LLM
   │
   ├── chunk 1 ──────────► Client
   ├── chunk 2 ──────────► Client
   ├── chunk 3 ──────────► Client
   ├── chunk 4 ──────────► Client
   └── chunk 5 ──────────► Client

Thay vì:

LLM xử lý 20 giây
       ↓
Hoàn thành toàn bộ
       ↓
Server trả response
       ↓
Client nhận tất cả

SSE cho phép:

LLM tạo "Xin"
       ↓
Client nhận "Xin"


LLM tạo " chào"
       ↓
Client nhận " chào"


LLM tạo " bạn"
       ↓
Client nhận " bạn"

Do đó người dùng nhìn thấy câu trả lời xuất hiện dần dần.

3. Vai trò của Flux

Trong WebFlux:

Flux<String>

có thể hiểu đơn giản là:

Một dòng dữ liệu có thể phát ra nhiều giá trị theo thời gian.

Ví dụ:

Flux.just("Xin", " chào", " bạn")

sẽ phát ra:

Xin
 chào
 bạn

chứ không nhất thiết phải đợi gom thành một String duy nhất.

Trong trường hợp Spring AI:

chatModel.stream(new Prompt(message))

model có thể trả về nhiều ChatResponse theo từng chunk.

Sau đó:

.map(...)

chuyển mỗi ChatResponse thành phần text tương ứng.

4. Tại sao Flux chưa chắc đã là SSE?

Đây là điểm quan trọng của bài.

Có:

Flux<String>

không đồng nghĩa tự động với việc API đang trả SSE.

SSE cần HTTP response có Content-Type:

text/event-stream

Vì vậy nên khai báo:

produces = MediaType.TEXT_EVENT_STREAM_VALUE

Ví dụ:

@GetMapping(
    value = "/api/v1/ai/stream",
    produces = MediaType.TEXT_EVENT_STREAM_VALUE
)

Khi đó Spring WebFlux biết rằng response là một stream SSE và có thể gửi từng event xuống client khi chúng phát sinh.

5. Controller hoàn chỉnh sau khi sửa
package com.example.rlogistics.controller;


import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;


@RestController
public class AiStreamController {


    private final ChatModel chatModel;


    public AiStreamController(ChatModel chatModel) {
        this.chatModel = chatModel;
    }


    @GetMapping(
            value = "/api/v1/ai/stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<String> getStreamResponse(
            @RequestParam String message
    ) {


        return chatModel.stream(new Prompt(message))
                .map(ChatResponse::getResult)
                .map(output -> output.getOutput().getText())
                .filter(text -> text != null && !text.isBlank());
    }
}
Điểm quan trọng nhất
produces = MediaType.TEXT_EVENT_STREAM_VALUE

Khai báo này cho Spring biết:

Response
   ↓
text/event-stream
   ↓
SSE
   ↓
gửi dữ liệu từng phần
6. Có thể tối ưu thêm bằng ServerSentEvent

Nếu muốn kiểm soát SSE rõ ràng hơn, có thể trả về:

Flux<ServerSentEvent<String>>

Code:

package com.example.rlogistics.controller;


import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.http.MediaType;
import org.springframework.http.codec.ServerSentEvent;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;


@RestController
public class AiStreamController {


    private final ChatModel chatModel;


    public AiStreamController(ChatModel chatModel) {
        this.chatModel = chatModel;
    }


    @GetMapping(
            value = "/api/v1/ai/stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<ServerSentEvent<String>> getStreamResponse(
            @RequestParam String message
    ) {


        return chatModel.stream(new Prompt(message))
                .map(response ->
                        response.getResult()
                                .getOutput()
                                .getText()
                )
                .filter(text -> text != null && !text.isBlank())
                .map(text ->
                        ServerSentEvent.<String>builder()
                                .data(text)
                                .build()
                );
    }
}

Khi đó dữ liệu HTTP có dạng SSE:

data: Xin


data: chào


data: bạn


data: Tôi có thể hỗ trợ quy trình thông quan...

Client có thể đọc từng event.