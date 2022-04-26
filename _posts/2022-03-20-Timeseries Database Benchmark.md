# Timeseries database performance tests

This is going to be just a short post about a simple benchmark for storing timestamped data in different database systems.

NOTE: The benchmark was done in only one afternoon and should not be considered for making important decisions or final conclusions.

The benchmark consisted on writing multiple records of timestamped data into a database and measuring how much time the operation took. The specific conditions under which each test was carried can be inferred from the scripts below. The results can be seen in the following image:

![Timeseries](/homepage/assets/img/2022-03-20-timeseries.png)

As can be seen on the chart, MongoDB outperformed all other database systems, even InfluxDB, whose main use case is timeseries data storage (albeit Influx does much more processing when inserting a new record than what we did with MongoDB, for example, checking that timestamps for a record are unique). The tests labeled with a "2" refer mostly to tests that were done with a record buffer to reduce the number of operations (see the code below). In all cases, an index in the timestamp was created (Influx indexes the `time` field automatically).

In a few words, four database systems were tested:
- InfluxDB 1.8
- MariaDB
- Postgres
- SQLite
- MongoDB

The system on which the tests were made was a Linux (Lubuntu) virtual machine with 4 processors (Intel i5 8th gen) and 4 gigabytes of RAM. The database were run on docker containers, as per the following commands:

```shell
sudo docker run -d -p 8086:8086 -v influxdb:/var/lib/influxdb influxdb:1.8
sudo docker run -d -p 3306:3306 -v mariadb:/var/lib/mysql -e MARIADB_ALLOW_EMPTY_ROOT_PASSWORD=True mariadb:latest
sudo docker run -d -p 5432:5432 -v postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres
sudo docker run -d -p 27017:27017 -v mongodb:/etc/mongo mongo
```

Python was used for running the tests. The corresponding scripts for each test are copied below:

- Main test class (uses one of the classes below):
    
```python
import random
import string 
import time
import datetime


Letters = ['a', 'b', 'c', 'd', 'e']


class Test:

	def __init__(self, test_client):
		self.series = ["Serie" + l.upper() for l in Letters]
		self.test_record_count = 100
		self.test_client = test_client()
		self.rand_seed = 0

	# Aux Functions
	def rand(self, type):
		val = None
		if type == bool:
			val = random.random() > 0.5
		if type == int:
			val = random.randint(-999999, 999999)
		if type == float:
			val = random.random() * random.randint(0, 99999)
		if type == str:
			val = ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(50))

		self.rand_seed += 1
		if self.rand_seed % 7 == 0: return val
		return None

	def generate_record(self):
		record = {}
		for l in Letters:
			record[l + str(1)] = self.rand(bool)
			record[l + str(2)] = self.rand(int)
			record[l + str(3)] = self.rand(float)
			record[l + str(4)] = self.rand(str)
		return {x: record[x] for x in record if record[x] is not None}

	# Write test
	def run_write_test(self):
		elapsed_times = []

		# Run a test for each series
		for serie in self.series:
			self.rand_seed = 0
			print(f"Writing records in {serie}")

			t0 = time.time()
			for i in range(0, self.test_record_count):
				record = self.generate_record()
				record["time"] = datetime.datetime.now() - datetime.timedelta(days=i)
				self.test_client.write_one(serie, record)

			elapsed_time = time.time() - t0
			print(f"Done writing to {serie}. Elapsed time: {elapsed_time} seconds")
			elapsed_times.append(str(elapsed_time))

		f = open(f"results/W_{self.test_client.name}_{self.test_record_count}", "w+")
		f.write(f"W,{self.test_client.name},{self.test_record_count},{','.join(elapsed_times)}")
		f.close()
```

- InfluxDB

```python
from influxdb import InfluxDBClient


class InfluxDB1:

	def __init__(self):
		self.client = InfluxDBClient("localhost", 8086, database="test")
		self.name = "influx"

	def write_one(self, serie, record):
		record["time"]  = record["time"].strftime("%Y-%m-%dT%H:%M:%SZ")
		self.client.write_points([{"measurement": serie, "tags": {}, "time": record["time"], "fields": record}])

	def read(self, serie, since, until):
		since = since.strftime("%Y-%m-%dT%H:%M:%SZ")
		until = until.strftime("%Y-%m-%dT%H:%M:%SZ")
		R = self.client.query(f"SELECT a1, a2, a3, a4 FROM {serie} WHERE time >= '{since}' AND time <= '{until}'")
		return list(R)[0]
		
class InfluxDB2:

	def __init__(self):
		self.client = InfluxDBClient("localhost", 8086, database="test")
		self.name = "influx2"
		self.buffer = []

	def write_one(self, serie, record):
		record["time"]  = record["time"].strftime("%Y-%m-%dT%H:%M:%SZ")
		self.buffer.append({"measurement": serie, "tags": {}, "time": record["time"], "fields": record})
		if len(self.buffer) == 100:
			self.client.write_points(self.buffer)
			self.buffer = []

	def read(self, serie, since, until):
		since = since.strftime("%Y-%m-%dT%H:%M:%SZ")
		until = until.strftime("%Y-%m-%dT%H:%M:%SZ")
		R = self.client.query(f"SELECT a1, a2, a3, a4 FROM {serie} WHERE time >= '{since}' AND time <= '{until}'")
		return list(R)[0]
```

- MariaDB, Postgres and SQLite

```python
import sqlite3
import mysql.connector
import psycopg2



def aux_bool_value_(dbs, v):
	if dbs == "postgres": return 'TRUE' if v else 'FALSE'
	else: return '1' if v else '0'

# ======================================================================
# SQL 1
# ======================================================================
class Sql1:

	def __init__(self):
		pass

	def write_one(self, serie, record):

		# Get time
		record["time"]  = record["time"].strftime("%Y-%m-%d %H:%M:%S")
		t = record["time"]

		# Create Table
		cur = self.client.cursor()
		cur.execute(f'CREATE TABLE IF NOT EXISTS {serie} (t TIMESTAMP NOT NULL, field VARCHAR(255) NOT NULL, int_value INTEGER DEFAULT NULL, float_value FLOAT(32) DEFAULT NULL, bool_value BOOL DEFAULT NULL, string_value VARCHAR(255) DEFAULT NULL)')

		# Create index for time
		cur.execute(f'CREATE INDEX IF NOT EXISTS tidx ON {serie} (t)')

		# Insert values for each record
		q = f"INSERT INTO {serie} (t, field, bool_value, int_value, float_value, string_value) VALUES "
		for field in record:
			if field == "time": continue
			value = record[field]
			if value is None : continue

			if isinstance(value, bool):
				q += f"('{t}', '{field}', {aux_bool_value_(self.dbs, value)}, NULL, NULL, NULL), "
			elif isinstance(value, int):
				q += f"('{t}', '{field}', NULL, {value}, NULL, NULL), "
			elif isinstance(value, float):
				q += f"('{t}', '{field}', NULL, NULL, {value}, NULL), "
			elif isinstance(value, str):
				q += f"('{t}', '{field}', NULL, NULL, NULL, '{value}'), "

		cur.execute(q[:-2])
		self.client.commit()


	def read(self, serie, since, until):

		# Get time
		since = since.strftime("%Y-%m-%d %H:%M:%S")
		until = until.strftime("%Y-%m-%d %H:%M:%S")

		# Perform query
		cur = self.client.cursor()
		cur.execute(f"SELECT * FROM {serie} WHERE t>='{since}' AND t<='{until}' AND (field = 'a1' OR field = 'a2' OR field = 'a3' OR field = 'a4')")
		R = cur.fetchall()
		
		# Group results by time
		data = {}
		for r in R:
			t = r[0]
			field = r[1]
			value = None
			if r[2] is not None: value = r[2]
			elif r[3] is not None: value = r[3]
			elif r[4] is not None: value = r[4]
			elif r[5] is not None: value = r[5]

			if t not in data: data[t] = {"t": t}
			data[t][field] = value

		return list(data.values())


# ======================================================================
# SQL2
# ======================================================================

class Sql2:

	def __init__(self):
		pass

	def write_one(self, serie, record):
		record["time"]  = record["time"].strftime("%Y-%m-%d %H:%M:%S")
		t = record["time"]

		cur = self.client.cursor()
		cur.execute(f"CREATE TABLE IF NOT EXISTS data (t TIMESTAMP NOT NULL, serie VARCHAR(255) NOT NULL, field VARCHAR(255) NOT NULL, int_value INTEGER DEFAULT NULL, float_value FLOAT(32) DEFAULT NULL, bool_value BOOL DEFAULT NULL, string_value VARCHAR(255) DEFAULT NULL)")
		cur.execute(f"CREATE INDEX IF NOT EXISTS tidx ON data (t)")

		q = f"INSERT INTO data (t, serie, field, bool_value, int_value, float_value, string_value) VALUES "
		for field in record:
			if field == "time": continue
			value = record[field]
			if value is None : continue

			if isinstance(value, bool):
				q += f"('{t}', '{serie}', '{field}', {aux_bool_value_(self.dbs, value)}, NULL, NULL, NULL), "
			elif isinstance(value, int):
				q += f"('{t}', '{serie}', '{field}', NULL, {value}, NULL, NULL), "
			elif isinstance(value, float):
				q += f"('{t}', '{serie}', '{field}', NULL, NULL, {value}, NULL), "
			elif isinstance(value, str):
				q += f"('{t}', '{serie}', '{field}', NULL, NULL, NULL, '{value}'), "

		cur.execute(q[:-2])
		self.client.commit()


	def read(self, serie, since, until):
		since = since.strftime("%Y-%m-%d %H:%M:%S")
		until = until.strftime("%Y-%m-%d %H:%M:%S")

		cur = self.client.cursor()
		cur.execute(f"SELECT * FROM data WHERE serie = '{serie}' AND t>='{since}' AND t<='{until}' AND (field = 'a1' OR field = 'a2' OR field = 'a3' OR field = 'a4')")
		R = cur.fetchall()
		
		data = {}
		for r in R:
			t = r[0]
			field = r[2]
			value = None
			if r[3] is not None: value = r[3]
			elif r[4] is not None: value = r[4]
			elif r[5] is not None: value = r[5]
			elif r[6] is not None: value = r[6]

			if t not in data: data[t] = {"t": t}
			data[t][field] = value

		return list(data.values())


# ======================================================================
# Sqlite
# ======================================================================

class Sqlite1(Sql1):
	def __init__(self):
		self.client = sqlite3.connect("testa1.db")
		self.dbs = "sqlite"
		self.name = "sqlite1"
		super().__init__()

class Sqlite2(Sql2):
	def __init__(self):
		self.client = sqlite3.connect("testa2.db")
		self.dbs = "sqlite"
		self.name = "sqlite2"
		super().__init__()


# ======================================================================
# MariaDB
# ======================================================================

class MariaDB1(Sql1):
	def __init__(self):
		self.client = mysql.connector.connect(host="localhost", port=3306, database="testa1", username="root", password="")
		self.dbs = "mariadb"
		self.name = "mariadb1"
		super().__init__()

class MariaDB2(Sql2):
	def __init__(self):
		self.client = mysql.connector.connect(host="localhost", port=3306, database="testa2", username="root", password="")
		self.dbs = "mariadb"
		self.name = "mariadb2"
		super().__init__()


# ======================================================================
# PostgreSQL
# ======================================================================

class Postgres1(Sql1):
	def __init__(self):
		self.client = psycopg2.connect(host="localhost", port=5432, database="testa1", user="postgres", password="password")
		self.dbs = "postgres"
		self.name = "postgres1"
		super().__init__()

class Postgres2(Sql2):
	def __init__(self):
		self.client = psycopg2.connect(host="localhost", port=5432, database="testa2", user="postgres", password="password")
		self.dbs = "postgres"
		self.name = "postgres2"
		super().__init__()
```


- MongoDB

```python
import pymongo
import urllib.parse


class MongoDB:

	def __init__(self):
		self.client = pymongo.MongoClient("mongodb://localhost:27017")
		self.database = "testA2"
		self.name = "mongo"

		self.client.drop_database(self.database)

	def write_one(self, serie, record):
		collection = self.client[self.database][serie]  # [database][collection]
		collection.create_index('time')
		collection.insert_one(record)

	def read(self, serie, since, until):
		collection = self.client[self.database][serie]  # [database][collection]
		found = collection.find({"time": {"$gte": since, "$lte": until}}, {"time": True, "a1": True, "a2": True, "a3": True, "a4": True, "_id": False})
		return [x for x in found if len(x) > 1]


class MongoDB2:

	def __init__(self):
		self.client = pymongo.MongoClient("mongodb://localhost:27017")
		self.database = "testA2"
		self.name = "mongo2"
		self.buffer = []
		self.client.drop_database(self.database)

	def write_one(self, serie, record):
		self.buffer.append(record)
		if len(self.buffer) == 100:
			collection = self.client[self.database][serie]  # [database][collection]
			collection.create_index('time')
			collection.insert_many(self.buffer)
			self.buffer = []

	def read(self, serie, since, until):
		collection = self.client[self.database][serie]  # [database][collection]
		found = collection.find({"time": {"$gte": since, "$lte": until}}, {"time": True, "a1": True, "a2": True, "a3": True, "a4": True, "_id": False})
		return [x for x in found if len(x) > 1]
```

