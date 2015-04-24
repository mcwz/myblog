---
layout: post
title: 使用netty基于websocket做一个聊天室
description: 本文介绍netty如何建立websocket链接，如果给别人发送信息，以及如何发送广播。
keywords: memcached,mongodb,redis,区别,项目实践
---

实际上以前做过一次netty实现websocket的实验，当时是按着《netty权威指南》书中的介绍敲代码做的测试，虽然运行没有问题，但是想进一步实现客户端互发消息没成功，恰好当时忽然忙起来了，这事儿就过去了。

最近稍微闲了些，决定继续试试。以前卡在的地方就是每次客户端连接生成的相关对象都是针对自己的，如果想互相发信息，必须要先识别到对方的java对象，思索良久，总是卡着过不去。后来灵机一动，才反应过来，可以设一个应用级的变量池子啊，把对象都放进去，然后需要给谁发就挑出哪个对象即可。

通了之后，立刻行动，先做个广播消息吧，也就是说不用识别是哪个对象了，收到个消息，然后循环调用每个对象发送这个消息，即完成广播功能，看起来就是一个聊天室的雏形了。

以下列举的代码大部分是《netty权威指南》提到的，在此表示感谢，我修改了一下，增加一些能够群发的逻辑，同时这只是个演示，代码中涉及到了多线程，但是没处理线程同步问题。

## 列出代码

WebSocketServer.java 主要完成程序启动

<pre class="prettyPrint">
public class WebSocketServer {

	public void run(int port) throws Exception {
		EventLoopGroup bossGroup = new NioEventLoopGroup();
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		try {
			ServerBootstrap b = new ServerBootstrap();
			b.group(bossGroup, workerGroup)
					.channel(NioServerSocketChannel.class)
					.childHandler(new ChannelInitializer<SocketChannel>() {
						@Override
						protected void initChannel(SocketChannel ch)
								throws Exception {
							ChannelPipeline pipeline = ch.pipeline();
							pipeline.addLast("http-codec",
									new HttpServerCodec());
							pipeline.addLast("aggregator",
									new HttpObjectAggregator(65536));
							ch.pipeline().addLast("http-chunked",
									new ChunkedWriteHandler());
							pipeline.addLast("handler",
									new WebSocketServerHandler());
						}
					});
			Channel ch = b.bind(port).sync().channel();
			System.out.println("Web Socket Server started
			at port :" + port	+ ".");
			System.out.println("Open your web browser and 
			navigate to http://localhost:"+ port + "/");
			
			CheckRunningPoll crp=new CheckRunningPoll();
			Thread checkpoll = new Thread(crp);
			checkpoll.start();
			
			ch.closeFuture().sync();
			
			

		} finally {
			bossGroup.shutdownGracefully();
			workerGroup.shutdownGracefully();
		}
	}

	public static void main(String[] args) {
		int port = 8080;

		if (args.length > 0) {
			try {
				port = Integer.parseInt(args[0]);
			} catch (NumberFormatException e) {
				e.printStackTrace();
			}
		}
		try {
			new WebSocketServer().run(port);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
</pre>

ClientPoll.java 客户端池子

<pre class="prettyPrint">
public class ClientPoll {
	public static Map<String,ChannelHandlerContext> clientMap=new HashMap<String, ChannelHandlerContext>();
}
</pre>

WebSocketServerHandler.java 完成websocket的定义过程

<pre class="prettyPrint">
public class WebSocketServerHandler extends SimpleChannelInboundHandler<Object> {
	private static final Logger logger=Logger.getLogger(WebSocketServerHandler.class.getName());
	private WebSocketServerHandshaker handshaker;
	private String clientIp;

	@Override
	protected void messageReceived(ChannelHandlerContext ctx, Object msg)
			throws Exception {
		if(msg instanceof FullHttpRequest)
		{
			FullHttpRequest mReq=(FullHttpRequest)msg;
			handleHttpRequest(ctx, mReq);
		}
		else if(msg instanceof WebSocketFrame)
		{
			handleWebSocketFrame(ctx, (WebSocketFrame)msg);
		}
		else if (msg instanceof HttpRequest) {
			HttpRequest mReq = (HttpRequest) msg;
			String clientIP = mReq.headers().get("X-Forwarded-For");
			if (clientIP == null) {
				InetSocketAddress insocket = (InetSocketAddress) ctx.channel()
						.remoteAddress();
				clientIP = insocket.getAddress().getHostAddress();
			}
		}
	}
	
	
	@Override
	public void channelReadComplete(ChannelHandlerContext ctx) throws Exception
	{
		ctx.flush();
	}
	
	@Override
	public void exceptionCaught(ChannelHandlerContext ctx,Throwable cause) throws Exception
	{
		cause.printStackTrace();
		ctx.close();
	}
	
	private void handleHttpRequest(ChannelHandlerContext ctx,FullHttpRequest req) throws Exception
	{
		if(!req.getDecoderResult().isSuccess() || (!"websocket".equals(req.headers().get("Upgrade"))))
		{
			sendHttpResponse(ctx, req, new DefaultFullHttpResponse(HttpVersion.HTTP_1_1,HttpResponseStatus.BAD_REQUEST));
			return ;
		}
		WebSocketServerHandshakerFactory wsFactory=new WebSocketServerHandshakerFactory("ws://192.168.12.51:8080/websocket", null, false);
		handshaker=wsFactory.newHandshaker(req);
		if(handshaker==null)
		{
			WebSocketServerHandshakerFactory.sendUnsupportedWebSocketVersionResponse(ctx.channel());
		}
		else
		{
			handshaker.handshake(ctx.channel(), req);
			String clientIP = req.headers().get("X-Forwarded-For");
			if (clientIP == null) {
				InetSocketAddress insocket = (InetSocketAddress) ctx.channel()
						.remoteAddress();
				clientIP = insocket.getAddress().getHostAddress();
			}
			this.clientIp=clientIP;
			ClientPoll.clientMap.put(clientIP, ctx);
		}

	}
	
	private void handleWebSocketFrame(ChannelHandlerContext ctx,WebSocketFrame frame)
	{
		if(frame instanceof CloseWebSocketFrame)
		{
			handshaker.close(ctx.channel(), (CloseWebSocketFrame) frame.retain());
			return ;
		}
		
		if(frame instanceof PingWebSocketFrame)
		{
			ctx.channel().write(new PongWebSocketFrame(frame.content().retain())); 
			return ;
		}
		
		if(!(frame instanceof TextWebSocketFrame))
		{
			throw new UnsupportedOperationException(String.format("%s frame types not supported ", frame.getClass().getName()));
		}
		
		String request=((TextWebSocketFrame) frame).text();
		if(logger.isLoggable(Level.FINE))
		{
			logger.fine(String.format("%s received %s", ctx.channel()));
		}
		Broadcast.broadcast("IP是"+this.clientIp+"的朋友给大家广播了一条消息："+request);
		//ctx.channel().write(new TextWebSocketFrame(request+",这里是服务器应答："+new java.util.Date().toString()));
	}
	
	private static void sendHttpResponse(ChannelHandlerContext ctx,FullHttpRequest req,FullHttpResponse res)
	{
		if(res.getStatus().code()!=200)
		{
			ByteBuf buf=Unpooled.copiedBuffer(res.getStatus().toString(),CharsetUtil.UTF_8);
			res.content().writeBytes(buf);
			buf.release();
			setContentLength(res,res.content().readableBytes());
		}
		ChannelFuture f=ctx.channel().writeAndFlush(res);
		if(!isKeepAlive(req) || res.getStatus().code()!=200)
		{
			f.addListener(ChannelFutureListener.CLOSE);
		}
	}
}
</pre>

CheckRunningPoll.java我自己写的一个线程，检测是否有新消息要群发

<pre class="prettyPrint">
public class CheckRunningPoll implements Runnable {

	@Override
	public void run() {
		try {
			while (true) {
				for (Entry<String, ChannelHandlerContext> entry : ClientPoll.clientMap.entrySet()) {
					String clientIP=entry.getKey();
					ChannelHandlerContext ctx=entry.getValue();
					ctx.channel().writeAndFlush(new TextWebSocketFrame("您的服务器IP是："+clientIP+",服务器发给您的消息是："+ new java.util.Date().toString()));
				}
				

				Thread.sleep(2000);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

}
</pre>

Broadcast.java广播消息

<pre class="prettyPrint">
public class Broadcast {
	public static void broadcast(String message)
	{
		for (Entry<String, ChannelHandlerContext> entry : ClientPoll.clientMap.entrySet()) {
			ChannelHandlerContext ctx=entry.getValue();
			ctx.channel().writeAndFlush(new TextWebSocketFrame(message));
		}
		
		
		
	}
}
</pre>

HTML代码，可以直接用支持Websocket的浏览器打开即可

<pre class="prettyPrint">
<p>
	&lt;!DOCTYPE html&gt;<br />
&lt;html&gt;<br />
&lt;head&gt;<br />
&lt;meta charset="UTF-8"&gt;<br />
Netty WebSocket 时间服务器<br />
&lt;/head&gt;<br />
&lt;br&gt;<br />
&lt;body&gt;<br />
&lt;br&gt;<br />
&lt;script type="text/javascript"&gt;<br />
var socket;<br />
var autosend;<br />
if (!window.WebSocket)&nbsp;<br />
{<br />
window.WebSocket = window.MozWebSocket;<br />
}<br />
if (window.WebSocket) {<br />
socket = new WebSocket("ws://websocket.gxd.test.com:8080/websocket");<br />
socket.onmessage = function(event) {<br />
var ta = document.getElementById('responseText');<br />
ta.value = ta.value+event.data+"\r\n";<br />
};<br />
socket.onopen = function(event) {<br />
var ta = document.getElementById('responseText');<br />
ta.value = "打开WebSocket服务正常，浏览器支持WebSocket!\r\n";<br />
autosend=setInterval(function(){<br />
now=new Date();<br />
send("我的时间是："+now.getFullYear() +"年"+(now.getMonth()+1)+"月"+now.getDate() +"日"+now.getHours()+"时"+now.getMinutes() +"分"+now.getSeconds()+"秒");<br />
},8000);<br />
};<br />
socket.onclose = function(event) {<br />
var ta = document.getElementById('responseText');<br />
ta.value = "";<br />
ta.value = "WebSocket 关闭!";&nbsp;<br />
};<br />
}<br />
else<br />
{<br />
alert("抱歉，您的浏览器不支持WebSocket协议!");<br />
}<br />
<br />
function send(message) {<br />
if (!window.WebSocket) { return; }<br />
if (socket.readyState == WebSocket.OPEN) {<br />
socket.send(message);<br />
}<br />
else<br />
{<br />
&nbsp;alert("WebSocket连接没有建立成功!");<br />
}<br />
}<br />
&lt;/script&gt;<br />
&lt;form onsubmit="return false;"&gt;<br />
&lt;input type="text" name="message" value="Netty最佳实践"/&gt;<br />
&lt;br&gt;&lt;br&gt;<br />
&lt;input type="button" value="发送WebSocket请求消息" onclick="send(this.form.message.value)"/&gt;<br />
&lt;hr color="blue"/&gt;<br />
&lt;h3&gt;服务端返回的应答消息&lt;/h3&gt;<br />
&lt;textarea id="responseText" style="width:900px;height:300px;"&gt;&lt;/textarea&gt;<br />
&lt;/form&gt;<br />
&lt;/body&gt;<br />
&lt;/html&gt;
</p>
<p>
	<br />
</p>
</pre>

## 大概解释一下

我就不详细介绍每行代码了，有不了解原理的建议读一下《netty权威指南》这本书，或者发评论咨询。

在这里我仅仅对我的改进思路解释一下：首先在websocket握手成功之后，把代表当前的客户端的对象存入池子，然后写一个线程每个两分钟看一下池子，有对象就循环发送服务器消息（服务器时间），在handleWebSocketFrame方法中调取Broadcast把刚刚收到的消息广播出去。

P.S. 

1. 文中的代码不是最优方案，仅作演示，有好的方案欢迎交流。
2. 文中代码取自《netty权威指南》223-230页。
