---
layout: post
title: 使用JMX动态设置JVM的HeapDumpPath
---
设置容器内存溢出时生成堆转存：
```
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data2/log/resin/`date`.hprof
```
偶尔发生内存溢出后发现生成的heap dump文件直接就叫\`date\`.hprof。
搜索了一下发现其实还可以通过JMX去动态修改部分JVM设置：
```
import java.lang.management.ManagementFactory;
import java.text.SimpleDateFormat;
import java.util.Date;

import javax.annotation.PostConstruct;

import org.apache.log4j.Logger;
import org.springframework.stereotype.Service;

import com.sun.management.HotSpotDiagnosticMXBean;
import com.sun.management.VMOption;

/**
 * Set heap dump file name.
 * @author lixuanbin
 */
@SuppressWarnings("restriction")
@Service
public class JmxBean {
	protected static final Logger log = Logger.getLogger(JmxBean.class);

	public static final boolean IS_WINDOWS = System.getProperty("os.name").toLowerCase()
			.startsWith("win");

	private String dumpPath;

	private HotSpotDiagnosticMXBean bean;

	@PostConstruct
	public void init() {
		try {
			if (IS_WINDOWS) {
				dumpPath = "E:/";
			} else {
				dumpPath = "/data2/log/resin";
			}
			String pid = ManagementFactory.getRuntimeMXBean().getName();
			pid = pid.substring(0, pid.indexOf('@'));
			String date = new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());
			String fileName = dumpPath + "/heap_" + pid + "_" + date + ".hprof";
			bean = ManagementFactory.newPlatformMXBeanProxy(
					ManagementFactory.getPlatformMBeanServer(),
					"com.sun.management:type=HotSpotDiagnostic", HotSpotDiagnosticMXBean.class);
			bean.setVMOption("HeapDumpOnOutOfMemoryError", "true");
			bean.setVMOption("HeapDumpPath", fileName);
			bean.setVMOption("PrintGC", "true");
			bean.setVMOption("PrintGCDetails", "true");
			bean.setVMOption("PrintGCDateStamps", "true");
		} catch (Exception e) {
			log.error(e.getMessage(), e);
		}
	}

	public HotSpotDiagnosticMXBean getMxBean() {
		return this.bean;
	}
	
	public String printVmOptions() {
		StringBuffer sb = new StringBuffer();
		if (getMxBean() != null) {
			for (VMOption option : getMxBean().getDiagnosticOptions()) {
				sb.append(option.getName() + ":" + option.getValue() + "\r\n");
			}
		}
		return sb.toString();
	}
}
```
