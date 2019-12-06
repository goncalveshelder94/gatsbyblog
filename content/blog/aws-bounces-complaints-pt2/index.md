---
title: "Handling Complaints Part 2"
date: 2019-11-01T15:07:02Z
img: "handling-complaints.jpg"
tags: [data]
author: "Helder Goncalves"
---

# Description

We sent a new batch of newsletters yesterday - Received 130 emails back, 95% bounces, 5% complaints. I have to grab the email files from S3 and combine them into a single file again.

This time I get ahold of bounce emails as well as complaints.

I create a Python script to mass unsubscribe (delete) users from an sqllite3 db. I wanted to go with C-Sharp for this task instead but since the newsletter framework is based on Django Python, I thought I’d keep some kind of integrity by sticking with Python. Even if only in my mind.

# Application

Check the django-newsletter application official documentation for the data models since I needed to find where the user subscriptions are stored. 

After finding the table that holds the user subscription data (newsletter_subscription) I start working on my python script that has to connect to the db.sqllite3 (Think of an Excel spreadsheet). I come across: sqlitetutorial.net resulting in the following skelleton:

```python
import sqlite3
from sqlite3 import Error
    
    
def create_connection(db_file):
""" create a database connection to the SQLite database
    specified by the db_file
:param db_file: database file
:return: Connection object or None
"""
conn = None
try:
    conn = sqlite3.connect(db_file)
except Error as e:
    print(e)
    
return conn
    
    
def select_all_tasks(conn):
"""
Query all rows in the tasks table
:param conn: the Connection object
:return:
"""
cur = conn.cursor()
cur.execute("SELECT * FROM tasks")
    
rows = cur.fetchall()
    
for row in rows:
    print(row)
    
    
def select_task_by_priority(conn, priority):
"""
Query tasks by priority
:param conn: the Connection object
:param priority:
:return:
"""
cur = conn.cursor()
cur.execute("SELECT * FROM tasks WHERE priority=?", (priority,))
    
rows = cur.fetchall()
    
for row in rows:
    print(row)
    
    
def main():
database = r"C:\sqlite\db\pythonsqlite.db"
    
# create a database connection
conn = create_connection(database)
with conn:
    print("1. Query task by priority:")
    select_task_by_priority(conn, 1)
    
    print("2. Query all tasks")
    select_all_tasks(conn)
    
    
if __name__ == '__main__':
main()
```
Relevant lines:

```python
[46] cur.execute("SELECT * FROM tasks")
[74] conn = create_connection(database)
```

```python
with conn:
    print("1. Query task by priority:")
    select_task_by_priority(conn, 1)

    print("2. Query all tasks")
    select_all_tasks(conn)
```

SELECT is an SQL operation to print data in a table (again think of an Excel Spreadsheet where you look at rows that have data across several columns). DELETE allows you to delete something from somewhere where something is equal to other thing.

We don’t care about selecting tasks by priority so we get rid of that whole definition itself.

Refactored code looks like:

```python
import sqlite3
import csv
from sqlite3 import Error
    
    
def create_connection(db_file):
""" create a database connection to the SQLite database
    specified by the db_file
:param db_file: database file
:return: Connection object or None
"""
conn = None
try:
    conn = sqlite3.connect(db_file)
except Error as e:
    print(e)
    
return conn
    
    
def select_all_tasks(conn):
"""
Query all rows in the tasks table
:param conn: the Connection object
:return:
"""
cur = conn.cursor()

with open('./emails.csv','rt') as source:
    dr = csv.reader(source)
    for row in dr:
        cur.execute("SELECT * FROM newsletter_subscription WHERE email = ?;", (row))
        print("Being deleted:")
        print(cur.fetchall())
        cur.execute("DELETE FROM newsletter_subscription WHERE email = ?;", (row))

def main():
database = r"./db.sqlite3"
    
# create a database connection
conn = create_connection(database)
with conn:
    print("2. Execute all tasks")
    select_all_tasks(conn)
    
    
if __name__ == '__main__':
main()
```

Our important code is on the select_all_tasks definition above

I open the emails.csv file produced by the program I made in csharp, see previous post

```python
with open('./emails.csv', 'rt') as source:
```

use python csv reader to read the file and iterate through it row by row

```python
dr = csv.reader(source) 
for row in dr:
```

query the database table newsletter_subscription where the field email is equal to row(s) (every line/row of the .csv file)

```python
cur.execute("SELECT * FROM newsletter_subscription WHERE email = ?;", (row))
print("Being deleted:")
print(cur.fetchall())
```

Code above is handy to inform the user running the script which accounts were found on the system database with the .csv file as input and that will be deleted

Next comes the DELETE statement to rid the database of the emails present on the .csv file

```python
cur.execute("DELETE FROM newsletter_subscription WHERE email = ?;", (row))
```

Final & Main bit

We specify the database file
We create the connection conn

With conn being true we select_all_tasks(conn) which executes the bits above

```python
def main():
database = r"./db.sqlite3"
    
# create a database connection
conn = create_connection(database)
with conn:
print("2. Execute all tasks")
select_all_tasks(conn)
    
    
if __name__ == '__main__':
main()
```

Changes to the csharp MailExtractor

Had to add a new regex pattern to match the bounce occurences
an else-if condition to match the new regex
apply a split method to the found line

apply a trim method to the object at index 1 (see below:)

```csharp
Regex emailPattern = new Regex(@"Final-Recipient:[ ]rfc822;.*$");
```

Find anything that says “`Final-Recipientrfc822; and everything behind until the end of the line (string)

Which found all strings containing “Final-Recipient: rfc822; janedoe@yahoo.co.uk”.

Once that’s found:

split the string by the ; semi-colon creating 2 objects:
    - “Final-Recipient: rfc822;”
    - “janedoe@yahoo.co.uk”

some emails are joined to the ; and others have space between [ ] so I have to trim every email (object 1) to remove preceding spaces

```csharp
emailAddr = newLine[1].Trim();
```

“[ ]janedoe@yahoo.co.uk” becomes “janedoe@yahoo.co.uk”

Refactored csharp code

```csharp
using System;
using System.IO;
using System.Text.RegularExpressions;
namespace MailExtractor
{
    class Program
    {
        static void Main(string[] args)
        {
            Extract_lines("newfile.file", "emails.csv");
        }
    public static void Extract_lines(string filein, string fileout)
    {
        Regex searchPattern = new Regex(@"Original-Rcpt-To:[ ](?!\.)(""([^""\r\\]|\\[""\r\\])*""|"
            + @"([-a-z0-9!#$%&'*+/=?^_`{|}~]|(?<!\.)\.)*)(?<!\.)"
            + @"@[a-z0-9][\w\.-]*[a-z0-9]\.[a-z][a-z\.]*[a-z]$");
        Regex emailPattern = new Regex(@"Final-Recipient:[ ]rfc822;.*$");
            using (StreamReader reader = new StreamReader(filein))
        {
            using (StreamWriter writer = new StreamWriter(fileout))
            {
                string line;
                string emailAddr;
                string[] newLine;
                while ((line = reader.ReadLine()) != null)
                {
                    if (searchPattern.IsMatch(line))
                    {
                            newLine = line.Split(" ");
                            writer.WriteLine(newLine[1]);
                    }
                    else if (emailPattern.IsMatch(line))
                    {
                            newLine = line.Split(";");
                            emailAddr = newLine[1].Trim();
                            writer.WriteLine(emailAddr);
                    }
                }
            }
        }
    }
    }
}
```

