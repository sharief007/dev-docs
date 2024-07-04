---
title: Logger
type: docs
sidebar:
  open: true
toc: true
params:
  editURL: 
---

Designing a logging framework using object-oriented principles requires careful consideration of flexibility, extensibility, and adherence to the SOLID principles. Here's a design that covers various requirements:

### Key Requirements:
1. **Flexibility**: Support different log levels (e.g., INFO, DEBUG, ERROR).
2. **Extensibility**: Easily extendable to support new logging destinations (e.g., file, console, remote server).
3. **Configuration**: Allow dynamic configuration changes.
4. **Thread-Safety**: Handle concurrent logging.
5. **Performance**: Minimize performance overhead.

### Design Overview:

```python
import os
import sqlite3
import sys
import threading
from abc import ABC, abstractmethod
from enum import Enum


class LogLevel(Enum):
    DEBUG = 1
    INFO = 2
    WARNING = 3
    ERROR = 4
    CRITICAL = 5


# Base logger
class Logger(ABC):
    def __init__(self, level: LogLevel = LogLevel.INFO):
        self.level = level
        self.lock = threading.Lock()

    def log(self, level: LogLevel, message: str):
        with self.lock:
            if level.value >= self.level.value:
                self._log_handler(message)

    def set_level(self, level: LogLevel):
        with self.lock:
            self.level = level

    @abstractmethod
    def _log_handler(self, message: str):
        pass

    @abstractmethod
    def close(self):
        pass


# Concrete logger implementation
class ConsoleLogger(Logger):
    def __init__(self, level: LogLevel = LogLevel.INFO):
        super(ConsoleLogger, self).__init__(level)
        self._destination = sys.stdout

    def _log_handler(self, message: str):
        try:
            print(message, file=self._destination)
        except Exception as e:
            print(f"Failed to log message to console: {e}")

    def close(self):
        pass  # No resources to close for console logger


# Concrete logger implementation
class FileLogger(Logger):
    def __init__(self, level: LogLevel = LogLevel.INFO, filepath: str = "output.log"):
        super(FileLogger, self).__init__(level)
        self._filepath = filepath
        self._file = open(self._filepath, mode="a", encoding="utf-8")

    def _log_handler(self, message: str):
        try:
            print(message, file=self._file)
        except Exception as e:
            print(f"Failed to log message to file: {e}")

    def close(self):
        try:
            self._file.close()
        except Exception as e:
            print(f"Failed to close file: {e}")


# Concrete logger implementation
class DBLogger(Logger):
    def __init__(self, level: LogLevel = LogLevel.INFO, database: os.PathLike = ':memory:'):
        super(DBLogger, self).__init__(level)
        self._connection = sqlite3.connect(database)
        self._destination = self._connection.cursor()

    def _log_handler(self, message: str):
        try:
            self._destination.execute('INSERT INTO logs (message) VALUES (?)', (message,))
            self._connection.commit()
        except Exception as e:
            print(f"Failed to log message to database: {e}")

    def close(self):
        try:
            self._destination.close()
            self._connection.close()
        except Exception as e:
            print(f"Failed to close database connection: {e}")


# Factory class
class LoggerFactory:
    @staticmethod
    def create_logger(logger_type: str, level: LogLevel = LogLevel.INFO, **kwargs) -> Logger:
        if logger_type == "db":
            return DBLogger(level, **kwargs)
        elif logger_type == "file":
            return FileLogger(level, **kwargs)
        else:
            return ConsoleLogger(level)


# Singleton class
class LogManager:
    _instance = None
    _lock = threading.Lock()
    _loggers = []

    def __new__(cls, *args, **kwargs):
        if not LogManager._instance:
            with LogManager._lock:
                if not LogManager._instance:
                    LogManager._instance = super().__new__(cls, *args, **kwargs)
        return LogManager._instance

    def register_logger(self, logger: Logger):
        with self._lock:
            self._loggers.append(logger)

    def unregister_logger(self, logger: Logger):
        with self._lock:
            self._loggers.remove(logger)
            logger.close()

    def log(self, level: LogLevel, message: str):
        with self._lock:
            for logger in self._loggers:
                logger.log(level, message)

    def set_level(self, level: LogLevel):
        with self._lock:
            for logger in self._loggers:
                logger.set_level(level)

    def close(self):
        with self._lock:
            for logger in self._loggers:
                logger.close()
            self._loggers = []


# Usage example
if __name__ == "__main__":
    log_manager = LogManager()

    console_logger = LoggerFactory.create_logger("console", LogLevel.DEBUG)
    file_logger = LoggerFactory.create_logger("file", LogLevel.INFO, filepath="app.log")
    db_logger = LoggerFactory.create_logger("db", LogLevel.WARNING, database="logs.db")

    log_manager.register_logger(console_logger)
    log_manager.register_logger(file_logger)
    log_manager.register_logger(db_logger)

    log_manager.log(LogLevel.DEBUG, "This is a debug message.")
    log_manager.log(LogLevel.INFO, "This is an info message.")
    log_manager.log(LogLevel.ERROR, "This is an error message.")

    log_manager.set_level(LogLevel.ERROR)
    log_manager.log(LogLevel.INFO, "This message should not appear in any log.")
    log_manager.log(LogLevel.ERROR, "This is another error message.")

    # Unregister loggers and close resources
    log_manager.close()
```

### Key Points:
- **Thread-Safety**: The use of `threading.Lock` ensures that logging operations are thread-safe.
- **Factory Pattern**: The `LoggerFactory` class simplifies the creation of different types of loggers.
- **Strategy Pattern**: Encapsulates logging algorithms (different loggers) and allows runtime substitution.
- **Singleton Pattern**: `LoggerManager` as a Singleton to ensure only one instance to manage all loggers throughout the application.
- **Template Pattern**: `LoggerManager` notifies all logger up receiving a new log message.
