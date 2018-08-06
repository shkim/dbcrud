title: Python MySQL
---

# Python MySQL
---

## Installation
```
$ pip install mysql-connector-python
```

## Connect
```python
import mysql.connector as mysql

cnx = mysql.connect(user='scott',
                    password='password',
                    host='127.0.0.1',
                    port='13306',
                    database='employees')
cnx.close()
```

```python
import mysql.connector as mysql
from mysql.connector import errorcode

try:
    cnx = mysql.connect(user='scott', database='testt')
except mysql.connector.Error as err:
    if err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
        print("Something is wrong with your user name or password")
    elif err.errno == errorcode.ER_BAD_DB_ERROR:
        print("Database does not exist")
    else:
        print(err)
else:
    cnx.close()
```

## Query
```python
cursor = cnx.cursor()
cursor.execute("SELECT * FROM MYTABLE", (hire_start, hire_end))

for (first_name, last_name, hire_date) in cursor:
    print("{}, {} was hired on {:%d %b %Y}".format(last_name, first_name, hire_date))

cursor.close()
```


TODO