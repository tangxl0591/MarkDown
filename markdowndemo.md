```plantuml
participant 新大陆nlservice服务
participant 系统Camera服务
participant 系统Scanner服务
Note over 系统Camera服务: 发送Camera使用请求
系统Camera服务->新大陆nlservice服务: 发送Camera使用请求
Note over 新大陆nlservice服务: 判断当前是否有其他Camera或Scanner操作，设置标志
新大陆nlservice服务-->>系统Camera服务: 返回Success
Note over 系统Camera服务: 打开Camera操作
系统Camera服务->新大陆nlservice服务: 关闭Camera使用请求
Note over 新大陆nlservice服务: 清除Camera使用标志
新大陆nlservice服务-->>系统Scanner服务: 通知Camer使用关闭
```

```wavedrom
{signal: [
  {name: 'clk', wave: 'p.....|...'},
  {name: 'dat', wave: 'x.345x|=.x', data: ['head', 'body', 'tail', 'data']},
  {name: 'req', wave: '0.1..0|1.0'},
  {},
  {name: 'ack', wave: '1.....|01.'}
]}
```
```viz
engine:dot
digraph g {
	node [shape=plaintext];
	A1 -> B1;
	A2 -> B2;
	A3 -> B3;

	A1 -> A2 [label=f];
	A2 -> A3 [label=g];
	B2 -> B3 [label="g'"];
	B1 -> B3 [label="(g o f)'" tailport=s headport=s];

	{ rank=same; A1 A2 A3 }
	{ rank=same; B1 B2 B3 }
}
```
