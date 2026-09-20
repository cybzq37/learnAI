
```
                 JSON
                  │
              Jackson
                  │
        ┌─────────┼──────────┐
        ↓         ↓          ↓
     Trigger   Condition   Action
                  │
                  ↓
                ANTLR
                  │
                  ↓
                 AST
                  │
                  ↓
             Rule Runtime
        ┌─────────┼──────────┐
        ↓         ↓          ↓
    Expression   Window     State
        │         │          │
        └─────────┼──────────┘
                  ↓
                Action
```

```
用户输入 DSL
    ↓
ANTLR Lexer 词法分析
    ↓
ANTLR Parser 语法分析
    ↓
Parse Tree
    ↓
AST Builder
    ↓
自定义 AST
    ↓
Rule Runtime
    ↓
true / false
    ↓
Action
```

定义 DSL（Domain Specific Language，领域专用语言），通过 ANTLR，转换成ASL（Abstract Syntax Tree，抽象语法树），计算：

ANTLR = “帮我定义并解析一种语言”  
CEL = “已经有一种成熟的表达式语言，你直接拿来用”

CEL（Common Expression Language）其实很适合这种场景

```
              IoT Rule DSL
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
     普通表达式             IoT Operator
          │                   │
         CEL              Window
          │               Duration
          │               Aggregate
          │               State
          └─────────┬─────────┘
                    ↓
              Rule Runtime
```

中小型平台
```
Spring Boot
    +
ANTLR
    +
自定义 AST
    +
自定义 Rule Runtime
    +
Redis
    +
Kafka
```

大型平台
```
Kafka
   ↓
Rule Engine
   ├── CEL
   ├── AST
   ├── Window
   ├── State
   └── CEP
   ↓
Flink / Kafka Streams
```