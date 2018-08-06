title: Python SAP HANA
---

# Python - SAP HANA (PyHDB)
---

## Installation

https://github.com/SAP/PyHDB

```
$ pip install pyhdb
```

## Connect

```python
import pyhdb

connection = pyhdb.connect(
    host='myhana.example.com',
    port=39015,
    user='SYSTEM',
    password='12345',
)

cursor = connection.cursor()
cursor.execute('SET SCHEMA "%s"' % ('myschema'))
cursor.close()
```
- SET SCHEMA ? (파라메터)는 안되는 것 같다.
- 쌍따옴표를 붙여야 대소문자 구분이 된다.

## Query

```python
csr = connection.cursor()
csr.execute('SELECT ID, CODE,NAME FROM MYTABLE WHERE A=? AND B=?', [a, b])
rows = csr.fetchall()

for row in csr.fetchall():
    data[row[0]] = { 'code': row[1], 'name': row[2] }

csr.close()
```
- 파라메터 전달은 [] 및 () 가능
- pyhdb 는 필드명으로 컬럼을 찾는 기능이 지원되지 않는 것 같다. 인덱스 잘 세어서...

```python
two_rows = csr.fetchmany(2)
```
- fetchall 과 비슷, 행 갯수를 지정할 수 있다.
- 또 fetch* 를 호출하면 다음 행을 얻을 수 있다.

```python
row1 = csr.fetchone()   # 1 row SELECT 할때
maybe_none = csr.fetchone() # 다음 행이 없다면 None 을 리턴한다.
```

```python
print(row[0].strftime("%A %d. %B %Y"))
```
- 컬럼 데이터 타입이 datetime 이면 자동으로 python datetime 오브젝트로 변경되어, datetime 메소드를 사용할 수 있다.

## Modify
```python
cursor.execute('INSERT INTO PYHDB_TEST(NAMES) VALUES(?)', ['hello'])
print(cursor.rowcount) # 1
connection.commit()
```
- 기본적으로 auto commit 모드가 아니라서 commit() 을 호출해줘야 반영된다.
- HANA 는 cursor.lastrowid 가 지원되지 않는다. 😱

```python
cursor.executemany('INSERT INTO PYHDB_TEST(NAMES) VALUES(?)', [ ['1'],['2'],['3'],['4'],['5'] ])
print(cursor.rowcount) # 1?
connection.commit()
```
- 여러개 insert 는 가능하지만 cursor.rowcount 는 1이 리턴되더라.. 😱

## LOB

TODO

## Stored Procedure
```python
cursor.execute('CALL MY_PROC1(?,?)', [a, b])
```
- 기본적인 호출은 그냥 위와 같이 하면 되지만..
- pyhdb로 stored procedure 의 output parameter 를 받는 방법을 못찾겠다. (검색해서 나오는 것은 에러남)
- SP가 필요하다면 hdbcli 를 쓰는 것이 좋겠다.
