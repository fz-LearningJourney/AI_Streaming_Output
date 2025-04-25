# **AI流式输出实现**

## 后端实现要点

### 1. Spring上下文和Bean初始化

```java
	private static ApplicationContext applicationContext;
    private OpenAiChatModel openAiChatModel;
    private AIChatService aiChatService;
    @Autowired
    public void setApplicationContext(ApplicationContext context) {
        applicationContext = context;
    }

    private void initBeans() {
        if (openAiChatModel == null) {
            openAiChatModel = applicationContext.getBean(OpenAiChatModel.class);
        }
        if (aiChatService == null) {
            aiChatService = applicationContext.getBean(AIChatService.class);
        }
    }
```

- 使用静态`ApplicationContext`获取Spring管理的Bean
- `initBeans()`方法延迟初始化所需的服务类
- 通过`@Autowired`注入应用上下文

### 2. WebSocket流式响应处理

```java
if(toUserId.equals("0")) {
    System.out.println("AI对话");
    //将时间戳作为AI的发送ID
    SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMddHHmmss");
    String dateStr = sdf.format(new Date());

    StringBuilder answerBuilder = new StringBuilder();//用于拼接回答的消息，以便保存到数据库
    Flux<ChatResponse> flux = openAiChatModel.stream(new Prompt(content));

    flux.toStream().forEach(chatResponse -> {
        String chunk = chatResponse.getResult().getOutput().getText();
        //去除最后一个null的显示
        if(chunk!=null) {
            try {
                session.getBasicRemote().sendText("用户 " + dateStr + " 内容: " + chunk);
                answerBuilder.append(chunk);//拼接
                Thread.sleep(50); // 控制流式输出速度
            } catch (IOException | InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    
    session.getBasicRemote().sendText("用户 " + toUserId + " 内容: AI回答完毕");
    String answer = answerBuilder.toString();
    //保存到数据库
    int status = aiChatService.AddAIChat(new AIChat(Integer.parseInt(fromUserId), content, answer, new Date()));
}
```

- 使用`Flux`实现流式响应
- 为每个AI响应生成唯一ID(使用时间戳)
- 逐块发送响应内容到前端
- 最终拼接完整响应并保存到数据库

## 前端实现要点

### 消息处理逻辑

```js
const handleWebSocketMessage = async (data) => {
  const IsFinish = ref(false);
  console.log("收到WebSocket消息:", data);
  
  // 解析消息格式 "用户 xxx 内容: xxx"
  const match = data.match(/用户 (\d+) 内容: (.+)/);
  if (match) {
    const [, fromUserId, content] = match;
    
    if(content !== "AI回答完毕"){
      // 处理流式内容
      if(!options.message.some(msg => msg.user === fromUserId && !msg.isComplete)) {
        // 创建新消息
        options.message.push({
          user: fromUserId,
          content: content,
          isComplete: false
        });
      } else {
        // 追加到现有消息
        const lastAIMessage = options.message.filter(msg => msg.user === fromUserId).pop();
        if(lastAIMessage) {
          lastAIMessage.content += content;
        }
      }
    } else {
      // 标记完成
      IsFinish.value = true;
      const lastAIMessage = options.message.filter(msg => msg.user === fromUserId).pop();
      if(lastAIMessage) {
        lastAIMessage.isComplete = true;
      }
    }
  }
}
```

### 发送消息

```js
// 发送问题到AI(目标ID设为0)
wsStore.sendMsg(0, options.question);
```

## 关键点总结

1. **流式传输机制**：
   - 后端使用`Flux`实现分块输出
   - 前端逐步拼接接收到的内容
2. **消息格式**：
   - 统一使用"用户 [ID] 内容: [消息内容]"格式
   - 特殊消息"AI回答完毕"标记流结束
3. **状态管理**：
   - 前端使用`isComplete`标记消息是否完整
   - 后端使用时间戳作为临时用户ID
4. **性能考虑**：
   - 后端添加50ms延迟使显示更自然
   - 前端优化消息更新逻辑
5. **数据持久化**：
   - 流式传输完成后保存完整对话到数据库

这种实现方式提供了类似ChatGPT的流畅对话体验，同时保证了数据的完整性和可追溯性。
