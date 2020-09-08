Space Operation
=================

http://master_server is the master service, $db_name is the name of the created database, $space_name is the name of the created tablespace.

Create Space
------------

::
   
  curl -XPUT -H "content-type: application/json" -d'
  {
      "name": "space1",
      "partition_num": 1,
      "replica_num": 1,
      "engine": {
          "name": "gamma",
          "index_size": 70000,
          "max_size": 10000000,
          "nprobe": 10,
          "metric_type": "InnerProduct",
          "ncentroids": 256,
          "nsubvector": 32
      },
      "properties": {
          "field1": {
              "type": "keyword"
          },
          "field2": {
              "type": "integer"
          },
          "field3": {
              "type": "float",
                  "index": "true"
          },
          "field4": {
              "type": "string",
              "array": true,
              "index": "true"
          },
          "field5": {
              "type": "integer",
              "array": false,
              "index": "true"
          },
          "field6": {
              "type": "vector",
              "dimension": 128
          },
          "field7": {
              "type": "vector",
              "dimension": 256,
              "store_type": "MemoryOnly",
              "store_param": {
              }
          }
      }
  }
  ' http://master_server/space/$db_name/_create


Parameter description:

+-------------+------------------+---------------+----------+------------------+
|field name   |field description | field type    |must      |remarks           | 
+=============+==================+===============+==========+==================+
|name         |space name        |string         |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|partition_num|partition number  |int            |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|replica_num  |replica number    |int            |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|engine       |engine config     |json           |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|properties   |schema config     |json           |true      |define space field|
+-------------+------------------+---------------+----------+------------------+

1、Space name not be empty, do not start with numbers or underscores, try not to use special characters, etc.

2、partition_num: Specify the number of tablespace data fragments. Different fragments can be distributed on different machines to avoid the resource limitation of a single machine.

3、replica_num: The number of copies is recommended to be set to 3, which means that each piece of data has two backups to ensure high availability of data. 

engine config:

+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|field name   |field description                     |field type |must      |remarks                                | 
+=============+======================================+===========+==========+=======================================+
|name         |engine name                           |string     |true      |currently fixed to gamma               |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|index_size   |slice index threshold                 |int        |false     |                                       |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|max_size     |maximum number of records in segments |int        |false     |                                       |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|nprobe       |number of cable bins                  |int        |false     |default 10                             |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|metric_type  |calculation method                    |string     |false     |InnerProduct and L2,  default L2       |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|ncentroids   |                                      |int        |false     |default 256                            |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+
|nsubvector   |PQ disassembler vector size           |int        |false     |default 64, must be a multiple of 4    |
+-------------+--------------------------------------+-----------+----------+---------------------------------------+


1. index_size: Specify the number of records in each partition to start index creation. If not specified, no index will be created. 

2. max_size: Specify the maximum number of records stored in each partition to prevent excessive server memory consumption. 

3. nprobe: The number of buckets to search in the index cannot be greater than the value of ncentroids.

4. metric_type: Specify the calculation method, inner product or Euclidean distance. 

5. ncentroids: Specifies the number of buckets for indexing.

6. nsubvector: PQ disassembler vector size.

properties config:

1. There are four types (that is, the value of type) supported by the field defined by the table space structure: keyword, integer, float, vector (keyword is equivalent to string).

2. The keyword type fields support index and array attributes. Index defines whether to create an index, and array specifies whether to allow multiple values.

3. Integer, float type fields support the index attribute, and the fields with index set to true support the use of numeric range filtering queries.

4. Vector type fields are feature fields. Multiple feature fields are supported in a table space. The attributes supported by vector type fields are as follows:

+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|field name   |field description          |field type |must    |remarks                                                     | 
+=============+===========================+===========+========+============================================================+
|dimension    |feature dimension          |int        |true    |Value is an integral multiple of the above nsubvector value |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|store_type   |feature storage type       |string     |false   |support Mmap and RocksDB, default Mmap                      |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|store_param  |storage parameter settings |json       |false   |set the memory size of data                                 |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|model_id     |feature plug-in model      |string     |false   |Specify when using the feature plug-in service              |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+

5. dimension: define that type is the field of vector, and specify the dimension size of the feature.

6. store_type: raw vector storage type, there are the following three options

"MemoryOnly": Vectors are stored in the memory, and the amount of stored vectors is limited by the memory. It is suitable for scenarios where the amount of vectors on a single machine is not large (10 millions) and high performance requirements

"RocksDB": Vectors are stored in RockDB (disk), and the amount of stored vectors is limited by the size of the disk. It is suitable for scenarios where the amount of vectors on a single machine is huge (above 100 millions) and performance requirements are not high.

"Mmap": Vectors are stored in the disk file, and the amount of stored vectors is limited by the size of the disk. It is suitable for scenarios where the amount of vectors on a single machine is huge (above 100 millions) and performance requirements are not high.

7. store_param: storage parameters of different store_type, it contains the following two sub-parameters

cache_size: interge type, the unit is M bytes, the default is 1024. When store_type="RocksDB", it indicates the read buffer size of RocksDB. The larger the value, the better the performance of reading vector. Generally set 1024, 2048, 4096 and 6144; when store_type="Mmap", it indicates the size of the write buffer , Don’t need to be too big, generally 512, 1024 or 2048 will do; store_type="MemoryOnly", it is useless.

compress: bool type, default false. True means to compress the original vector, generally the original vector will be compressed to 50% of the original, which can save memory and disk; false means no compression.


View Space
----------
::
  
  curl -XGET http://master_server/space/$db_name/$space_name


Delete Space
------------
::
 
  curl -XDELETE http://master_server/space/$db_name/$space_name

