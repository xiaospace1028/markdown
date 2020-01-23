# 模仿阿里云arms

## 注意：以下行为没有测量性能 ，只是检测可行性；

企业全面拥抱阿里云，里面有一个arms监控。可以不加入sdk的情况下查看**方法栈**和**线程栈**。有意思，研究一下怎么实现的，除了k8s配置化启动方式，还有下载jar包本地启动模式；

我们把jar下下来反编译后得到agent，根据java agent 进行代码监控。

java agent 有大致两个方法（其实是四个）premain和agentmain 阿里云用的是premain 也就是启动项里面加参数的方式

 -javaagent:xxxxx

[阿里云arms介绍](https://helpcdn.aliyun.com/document_detail/143534.html?spm=a2c4g.11186623.6.552.77ab5b2e2Zfu0E)

我们可以尝试用其他的方式来做agentmain(Attach机制) ，因为有些项目不会重启，或者说想给已有的线上项目添加的时候怎么做？

### 问题一：当addTransformer 方法中没有找到自己的类怎么处理？

一般的做法就是Instrumentation.addTransformer 方法因为这里面可以修改class文件 实现ClassFileTransformer 就可以了，但是在已经启动的项目当中很多的class都已经是启动过后已经加载好了，这个时候需要将原来的类retransformClasses一下，可以让类被重新转化，从而进入addTransformer方法；

```java
inst.addTransformer((ClassLoader loader, String className, Class<?> classBeingRedefined,
				ProtectionDomain protectionDomain, byte[] classfileBuffer) -> {
    //这里时第二次过滤需要使用Adpter的类
			if (classBeingRedefined.getName().startsWith("com.xiaospace.")) {
				ClassReader cr = new ClassReader(classfileBuffer);
				ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
				cr.accept(new TestClassAdpter(cw), ClassReader.EXPAND_FRAMES);
				return cw.toByteArray();
			}
			return null;
		}, true);

		Class[] allLoadedClasses = inst.getAllLoadedClasses();
		for (int i = 0; i < allLoadedClasses.length; i++) {
			Class classes = allLoadedClasses[i];
            //这里是重新转化的类过滤
			if (classes.getName().startsWith("com.xiaospace.")) {
				inst.retransformClasses(classes);
			}
		}
```

### 问题二：怎么更改	字节码？

先需要理解字节码是什么，以及怎么修改？

需要阅读完以下文档前半部分，4和5差距我不知道什么但是阅读完你能够对这个框架以及字节码有所了解，看完前五章（主要前三章），asm api操作的，然后下载插件，笔者是eclipse所以就用bytecode查看就行，安装的话自己百度就行；

[ASM4使用指南.pdf](../pdf/ASM4使用指南.pdf)

maven--asm

```xml
<!-- https://mvnrepository.com/artifact/org.ow2.asm/asm-all -->
<dependency>
	<groupId>org.ow2.asm</groupId>
	<artifactId>asm-all</artifactId>
	<version>5.2</version>
</dependency>
```

打包pom

```xml
<build>
		<defaultGoal>compile</defaultGoal>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<version>3.0.2</version>
			</plugin>
			<plugin>
				<artifactId>maven-assembly-plugin</artifactId>
				<configuration>
					<descriptorRefs>
						<descriptorRef>jar-with-dependencies</descriptorRef>
					</descriptorRefs>
					<archive>
						<manifestEntries>
							<Agent-Class>com.mock.agent.TestAgent</Agent-Class>
							<Can-Redefine-Classes>true</Can-Redefine-Classes>
							<Can-Retransform-Classes>true</Can-Retransform-Classes>
						</manifestEntries>
					</archive>
				</configuration>
				<executions>
					<execution>
						<id>make-assembly</id>
						<phase>package</phase>
						<goals>
							<goal>single</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
```

# java相关代码 

## TestClassAdpter.java

```java
package com.mock.agent.asm;

import org.objectweb.asm.AnnotationVisitor;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AnalyzerAdapter;
import org.objectweb.asm.commons.LocalVariablesSorter;

import com.mock.agent.util.OutTime;

public class TestClassAdpter extends ClassVisitor implements Opcodes {
	private String className;
	private String methodName;
	private String methodDesc;

	private boolean isInterface;

	private boolean isAnno = false;

	public TestClassAdpter(ClassVisitor cv) {
		super(ASM5, cv);
	}

	@Override
	public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
		cv.visit(version, access, name, signature, superName, interfaces);
		className = name;
		isInterface = (access & ACC_INTERFACE) != 0;
	}

	@Override
	public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
		AnnotationVisitor visitAnnotation = cv.visitAnnotation(desc, visible);
		isAnno = true;
		return visitAnnotation;
	}

	@Override
	public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
		MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);
		// 分别的意思是 不是接口类，非"<init>"方法非lamba表达式，存在注解类的类，这里是可以过滤类上面的参数的，方便哪些类的方法需要统计哪些不需要
		if (!isInterface && mv != null && !name.equals("<init>") && !name.startsWith("lambda") && isAnno) {
			methodName = name;
			TestMethodAdapter atma = new TestMethodAdapter(mv);
			atma.setAa(new AnalyzerAdapter(className, access, name, desc, atma));
			atma.setLvs(new LocalVariablesSorter(access, desc, atma.getAa()));
			return atma.getLvs();
		}
		return mv;
	}

	class TestMethodAdapter extends MethodVisitor {
		private AnalyzerAdapter aa;
		private LocalVariablesSorter lvs;
		private int durationId = -1;
		private int maxStack;

		public AnalyzerAdapter getAa() {
			return aa;
		}

		public void setLvs(LocalVariablesSorter lvs) {
			this.lvs = lvs;
		}

		public void setAa(AnalyzerAdapter aa) {
			this.aa = aa;
		}

		public LocalVariablesSorter getLvs() {
			return lvs;
		}

		public TestMethodAdapter(MethodVisitor mv) {
			super(ASM5, mv);
		}

		@Override
		public void visitParameter(String name, int access) {
			super.visitParameter(name, access);
		}

		@Override
		public void visitCode() {
			mv.visitCode();
			durationId = lvs.newLocal(Type.getType(OutTime.class));
			mv.visitTypeInsn(NEW, "com/mock/agent/util/OutTime");
			mv.visitInsn(DUP);
			mv.visitMethodInsn(INVOKESPECIAL, "com/mock/agent/util/OutTime", "<init>", "()V", false);
			mv.visitVarInsn(ASTORE, durationId);
			maxStack = 4;
		}

		@Override
		public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
			if (isAnno) {
				methodDesc = desc;
			}
			super.visitMethodInsn(opcode, owner, name, desc, itf);
		}

		@Override
		public void visitInsn(int opcode) {
			if ((opcode >= IRETURN && opcode <= RETURN) || opcode == ATHROW) {

				mv.visitVarInsn(ALOAD, durationId);
				mv.visitLdcInsn(className);
				mv.visitLdcInsn(methodName);
				mv.visitLdcInsn(methodDesc);
				mv.visitMethodInsn(INVOKEVIRTUAL, "com/mock/agent/util/OutTime", "endMethod",
						"(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V", false);
				maxStack = Math.max(aa.stack.size() + 4, maxStack);
			}
			mv.visitInsn(opcode);
		}

		@Override
		public void visitMaxs(int maxStack, int maxLocals) {
			super.visitMaxs(Math.max(maxStack, this.maxStack), maxLocals);
		}
	}
}
```

## OutTime.java 

```java
package com.mock.agent.util;
public class OutTime {
	private long start;
	public OutTime() {
		start = System.currentTimeMillis();
	}
	public void endMethod(String className, String methodName,String methodDesc) {
		long end = System.currentTimeMillis();
		System.out.println(className+"."+methodName+"【"+methodDesc+"】--time--【"+ (end - start) + "】毫秒");
	}
}
```

启动项目后

```java
public static void main(String[] args)
			throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
		List<VirtualMachineDescriptor> list = VirtualMachine.list();
		for (VirtualMachineDescriptor vmd : list) {
			System.out.println();
			System.out.print(vmd.displayName());
			if (vmd.displayName().endsWith("WebTestApplication")
					|| vmd.displayName().endsWith("webtest-0.0.1-SNAPSHOT.jar")
				|| vmd.displayName().startsWith("com.xiaospace.")) {
				VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
				virtualMachine.loadAgent(
						"D:\\workspace-eclipse\\myself\\agent-demo\\target\\agent-demo-0.0.1-SNAPSHOT-jar-with-dependencies.jar",
						vmd.displayName().replace(".", "_"));
				System.out.print("--成功");
				virtualMachine.detach();
			}else {
				if(!StringUtils.isEmpty(vmd.displayName())) {
					System.out.print("--失败");
				}
			}
		}
	}
```

log日志

```log
com.xiaospace.webtest.asm.service.AsmService.send(Ljava/lang/String;)Ljava/lang/String;--time--【1】毫秒
com.xiaospace.webtest.asm.controller.AsmController.send(Ljava/lang/Object;)V--time--【2】毫秒
```

在很多地方都可以做埋点等等操作，

* 耗时操作：写一个startMethod以及endMethod ，不太推荐耗时操作

* jar包冲突：agent导入jar包的时候最好不要冲突原来的项目类似于日志，json化工具，http请求客户端，要确保原来项目健壮性，最好有一个检测包的机制，只打印自己写的代码的时间长度，以及扫描原项目导入的包和agent项目的包冲突情况，类似于阿里云asm插件包jar里面的功能；**一定需要保证被监控的项目的稳定性。**

* TraceId传输：TraceId这个第一个是可以在进入项目tomcat的servlet可以判断是否存在header里面是不是有traceId，有的话存入用threadlocal，以及修改http请求某几个jar包的header里面处理，这样子分布式服务里面就会存在traceId
* 日志处理：阿里云的arms是将一些日志先存在本地，然后推送到oss上面，占用了一点点io资源，至于内存和cpu消耗是少的，走的也应该是内网io所以整体上没有，日志也应该有其他服务处理，变成可查询的数据；

上面只是一个demo和一些想法需要系统的出来一个项目需要好多东西，