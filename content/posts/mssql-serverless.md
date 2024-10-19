---
title: "Sleepy Serverless SQL Database"
date: 2024-10-19T15:51:43-05:00
tags: ["Azure", "SQL Server", "Databases", "Serverless", "Python", "Retry Logic", "Tenacity", "pyodbc", "Cold Start", "Connection Retry"]
toc: true
---
## Introduction
Serverless Azure SQL Database is a compute tier that automatically scales compute resources based on workload demand, offering per-second billing for actual usage. It intelligently manages costs by automatically pausing the database during inactive periods, charging only for storage, and then "seamlessly" (note the quotes here) resuming when activity returns.

Serverless Azure SQL Database is particularly suitable for applications with intermittent, unpredictable usage patterns or for development and testing environments. It allows developers to focus on building applications without worrying about database management or capacity planning.

In case you want to know more details about the service, you can check the [official documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview?view=azuresql&tabs=general-purpose#overview).

Did you believe that? I did too, but when you start using it, you realize that's not as simple as it sounds. It is a great service... But for the right use cases! Wakey-wakey, sleepy DB!

## Waking Up the Sleepy DB
The first time you use Serverless Azure SQL Database, you will be amazed by how easy it is to set up and start using it. You will be able to create a database in minutes, and you will be able to connect to it and start querying it right away. You will be able to see how the database automatically pauses after a period of inactivity, and you will be able to see how it automatically resumes when you start querying it again... If you code your application properly!

As most of you have heard, the biggest challenge with "Serverless" offerings is the cold start times. Serverless Azure SQL Database is no exception. The first time you query the database after it has been paused, you will experience a delay while the database is being resumed. This delay can be significant, and it can cause your application to time out if you are not careful. So it is strongly recommended to follow the [connection retry logic recommendations](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry). So keep in mind that the auto-resume can take from 1 to 10 minutes, depending on the scenario.

## Simple Retry Logic in Python
Here is a simple example of how you can implement a retry logic in Python using [pyodbc](https://github.com/mkleehammer/pyodbc/wiki) and the [Tenacity](https://tenacity.readthedocs.io/en/latest/) library. This example shows how to connect to a database and execute a query, retrying the connection up to 10 times with an exponential backoff strategy.

```python
import time
import pyodbc
from tenacity import retry, stop_after_attempt, wait_exponential

class DatabaseConnection:
    def __init__(self, connection_string):
        self.connection_string = connection_string

    @retry(stop=stop_after_attempt(10), wait=wait_exponential(multiplier=1, min=4, max=10))
    def connect(self):
        try:
            connection = pyodbc.connect(self.connection_string)
            print("Connected successfully")
            return connection
        except pyodbc.Error as e:
            print(f"Error connecting to database: {e}")
            raise

    def execute_query(self, query):
        try:
            connection = self.connect()
            cursor = connection.cursor()
            cursor.execute(query)
            result = cursor.fetchall()
            cursor.close()
            connection.close()
            return result
        except Exception as e:
            print(f"Error executing query: {e}")
            raise

# Usage example
if __name__ == "__main__":
    connection_string = "DRIVER={ODBC Driver 17 for SQL Server};SERVER=your_server;DATABASE=your_database;UID=your_username;PWD=your_password"
    db = DatabaseConnection(connection_string)
    
    try:
        result = db.execute_query("SELECT * FROM your_table")
        print(result)
    except Exception as e:
        print(f"Failed to execute query after multiple attempts: {e}")
```

## Conclusion
Serverless Azure SQL Database is a great service that can help you save costs and simplify database management. However, it is important to be aware of its limitations and challenges, such as cold start times and connection retries. By following best practices and implementing retry logic in your applications, you can make the most of this service and avoid common pitfalls. This post is by no means a comprehensive list of all the issues you may face, but it should save you some time when you start using this service, and suddenly start having problems with this "sleepy" DB.
