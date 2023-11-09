![[logging-framework.svg]]



```java
public interface LogHandler {

	int DEBUG_LEVEL = 1;
	int INFO_LEVEL = 2;
	int ERROR_LEVEL = 3;
	
	void setNext(LogHandler next);
	void log(int level, String message);
}

public abstract class BaseLogHandler implements LogHandler {

	private LogHandler next;
	protected final int baseLevel;

	public BaseLogHandler(int baseLevel) {
		this.baseLevel = baseLevel;
	}

	@Override
	public void setNext(LogHandler next) {
		this.next = next;
	}

	@Override
	public void log(int level, String message) {
		if (next != null) {
			next.log(level, message);
		}
	}
}

public class DebugLogHandler extends BaseLogHandler {

	public DebugLogHandler(int baseLevel) {
		super(baseLevel);
	}
	
	@Override
	public void log(int level, String message) {
		if (level == DEBUG_LEVEL && this.baseLevel <= DEBUG_LEVEL) {
			String logMessage = String
					.format("%s [DEBUG]: %s", LocalDateTime.now(), message);
			System.out.println(logMessage);
			return;
		}

		super.log(level, message);
	}
}


public class InfoLogHandler extends BaseLogHandler {

	public InfoLogHandler(int baseLevel) {
		super(baseLevel);
	}
	
	@Override
	public void log(int level, String message) {
		if (level == INFO_LEVEL && this.baseLevel <= INFO_LEVEL) {
			String logMessage = String
					.format("%s [INFO]: %s", LocalDateTime.now(), message);
			System.out.println(logMessage);
			return;
		}

		super.log(level, message);
	}
}

public class ErrorLogHandler extends BaseLogHandler {

	public ErrorLogHandler(int baseLevel) {
		super(baseLevel);
	}
	
	@Override
	public void log(int level, String message) {
		if (level == ERROR_LEVEL && this.baseLevel <= ERROR_LEVEL) {
			String logMessage = String
					.format("%s [ERROR]: %s", LocalDateTime.now(), message);
			System.out.println(logMessage);
			return;
		}

		super.log(level, message);
	}
}


public class Logger {

	private final LogHandler baseLogHandler;

	public Logger(LogHandler logHandler) {
		this.baseLogHandler = logHandler;
	}

	public void info(String message) {
		log(LogHandler.INFO_LEVEL, message);
	}
	
	public void error(String message) {
		log(LogHandler.ERROR_LEVEL, message);
	}
	
	public void debug(String message) {
		log(LogHandler.DEBUG_LEVEL, message);
	}
	
	private void log(int level, String message) {
		baseLogHandler.log(level, message);
	}
}


public class LogManager {

	public static Logger getLogger(int level) {
		LogHandler debugLogger = new DebugLogHandler(level);
		LogHandler infoLogger = new InfoLogHandler(level);
		LogHandler errorLogger = new ErrorLogHandler(level);
		
		debugLogger.setNext(infoLogger);
		infoLogger.setNext(errorLogger);
		
		Logger log = new Logger(debugLogger);
		return log;
	}
}


public class LoggerClient {
	
	public static void main(String[] args) {
		Logger log = LogManager.getLogger(LogHandler.DEBUG_LEVEL);
		log.info("chain of responsibility");
		log.error("exception occurred");
		log.debug("value of client = 2");
	}

}

```