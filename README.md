# hive_compared_bq

hive_compared_bq is a Python program that compares 2 (SQL like) tables, and graphically shows the rows/columns that are different if any are found.

Currently, there is no solution available on the internet that allows to compare the full content of some tables in a scalable way.

hive_compared_bq tackles this problem by:
* Not moving the data, avoiding long data transfer typically needed for a "Join compared" approach. Instead, only the aggregated data is moved.
* Working on the full datasets and all the columns, so that we are 100% sure that the 2 tables are the same
* Leveraging the tools (currently: Hive, BigQuery) to be as scalable as possible
* Showing the developer in a graphical way the differences found, so that the developer can easily understands its mistakes and fix them

# Table of contents
  * [Features](#features)
  * [Installation](#installation)
    + [Note Python version 2.6](#note-python-version-26)
    + [Installing Python 2.7](#installing-python-27)
    + [For Hive:](#for-hive-)
    + [For Big Query:    (steps extracted from https://cloud.google.com/bigquery/docs/reference/libraries#client-libraries-install-python)](#for-big-query------steps-extracted-from-https---cloudgooglecom-bigquery-docs-reference-libraries-client-libraries-install-python-)
  * [Usage](#usage)
    + [Get Help](#get-help)
    + [Basic execution](#basic-execution)

## Features

* Engine supported: Hive, BigQuery. (in theory, it is easy to extend it to other SQL backends such as Spanner, CloudSQL, Oracle... Help is welcomed :) ! )
* Possibility to only select specific columns or to remove some of them (useful if the schema between the tables is not exactly the same, or if we know that some columns are different and we don't want them to "pollute" the results)
* Possibility to just do a quick check (counting the rows in an advanced way) instead of complete checksum verification
* Detection of skew


# TODO explain a bit algorithm + show images of results + explain how to use it

## Installation

Make sure the following software is available:
* Python = 2.7 or 2.6 (see restrictions for 2.6)
* pyhs2 (just for Hive)
* google.cloud.bigquery (just for BigQuery)

### Note Python version 2.6
BigQuery is not supported by this version of Python (Google Cloud SDK only supports Python 2.7)

It is needed to install some backports for Python's collection. For instance, for a Redhat server it would be:
```bash
yum install python-backport_collections.noarch
```

### Installing Python 2.7
On RHEL6, Python 2.6 was installed by default. And the RPM version in EPEL for Python2.7 had no RPM available, so I compiled it this way:
```bash
su -c "yum install gcc gcc-c++ zlib-devel sqlite-devel openssl-devel"
wget https://www.python.org/ftp/python/2.7.13/Python-2.7.13.tgz
tar xvfz Python-2.7.13.tgz
cd Python-2.7.13
./configure --with-ensurepip=install
make
su -c "make altinstall"
su -c "echo 'export CLOUDSDK_PYTHON=/usr/local/bin/python2.7' >> /etc/bashrc"
export CLOUDSDK_PYTHON=/usr/local/bin/python2.7
```

### For Hive:
Execute this as "root":
```bash
yum install cyrus-sasl-gssapi cyrus-sasl-devel
pip install pyhs2
```

### For Big Query:    (steps extracted from https://cloud.google.com/bigquery/docs/reference/libraries#client-libraries-install-python)
To execute below commands, make sure that you already have a Google Cloud account with a BigQuery project created.

```bash
mkdir -p ~/bin/googleSdk
cd ~/bin/googleSdk
su -c "pip install --upgrade google-cloud-bigquery"

wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-170.0.0-linux-x86_64.tar.gz
tar xvfz google-cloud-sdk-170.0.0-linux-x86_64.tar.gz 

./google-cloud-sdk/bin/gcloud init
./google-cloud-sdk/bin/gcloud auth application-default login
./google-cloud-sdk/install.sh
```

This last command tells you to source 2 files in every session. So include them in your bash profile. For instance in my case it would be:
```bash
echo 'source /home/sluangsay/bin/googlesdk/google-cloud-sdk/path.bash.inc' >> ~/.bash_profile
echo 'source /home/sluangsay/bin/googlesdk/google-cloud-sdk/completion.bash.inc' >> ~/.bash_profile
```
Open a new bash session to activate those changes

## Usage

### Get Help

To see all the quick options of hive_compared_bq.py, just execute it without any arguments:
```bash
python hive_compared_bq.py
```

When calling the help, you get more detailled on the different options and also some examples:
```bash
python hive_compared_bq.py --help
```

### Basic execution

You must indicate the 2 tables to compare as arguments (the first table will be considered as the "source" table, and the second one as the "destination" table).

Each table must have the following format: <type>/<database>.<table>
where:
* "type" is the technology of your database (currently, only 'hive' and 'bq' (BigQuery) are supported)
* "database" is the name of your database (also called "dataset" in BigQuery)
* "table" is of course the name of your table

About the location of those databases:
* In the case of BigQuery, the default Google Cloud project configured in your environment is selected
* In the case of Hive, you must specify the hostname of the HiveServer2, using the 'hs2' option.

Another note for Hive: you need to pass the HDFS direction of the jar of the required UDF (see installation of Hive above), using the 'jar' option.

To clarify all the above, let's consider that we want to compare the following 2 tables:
* A Hive table called "hive_compared_bq_table", inside the database "sluangsay". With those parameters, the argument to give is: "hive/sluangsay.hive_compared_bq_table". Let's suppose also that the hostname of HiveServer2 is 'master-003.bol.net', and that we installed the required Jar in the HDFS path 'hdfs://hdp/user/sluangsay/lib/sha1.jar'. Then, since this table is the first one on our command line, it is the source table and we need to define the option: -s "{'jar': 'hdfs://hdp/user/sluangsay/lib/sha1.jar'
* A BigQuery table also alled "hive_compared_bq_table", inside the dataset "bidwh2".

To compare above 2 tables, you need to execute:
```bash
python hive_compared_bq.py -s "{'jar': 'hdfs://hdp/user/sluangsay/lib/sha1.jar', 'hs2': 'master-003.bol.net'}" hive/sluangsay.hive_compared_bq_table bq/bidwh2.hive_compared_bq_table
```

### Explanation on results

When executing above command, if the 2 tables have exactly the same data, then we would get an output similar to:

```
[INFO]  [2017-09-15 10:59:20,851]  (MainThread) Analyzing the columns ['rowkey', 'calc_timestamp', 'categorization_step', 'dts_modified', 'global_id', 'product_category', 'product_group', 'product_subgroup', 'product_subsubgroup', 'unit'] with a sample of 10000 values
[INFO]  [2017-09-15 10:59:22,285]  (MainThread) Best column to do a GROUP BY is rowkey (apparitions of most frequent value: 1 / the 50 most frequentvalues sum up 50 apparitions)
[INFO]  [2017-09-15 10:59:22,286]  (MainThread) Executing the 'Group By' Count queries for sluangsay.hive_compared_bq_table (hive) and bidwh2.hive_compared_bq_table (bigQuery) to do first comparison
No differences where found when doing a Count on the tables sluangsay.hive_compared_bq_table and bidwh2.hive_compared_bq_table and grouping by on the column rowkey
[INFO]  [2017-09-15 10:59:48,727]  (MainThread) Executing the 'shas' queries for hive_sluangsay.hive_compared_bq_table and bigQuery_bidwh2.hive_compared_bq_table to do final comparison
Sha queries were done and no differences where found: the tables hive_sluangsay.hive_compared_bq_table and bigQuery_bidwh2.hive_compared_bq_table are equal!
```

The first 2 lines describe the first step of the algorithm (see the explanation of the algorithm later), and we see that 'rowkey' is the column that is here used to perform the "Group By" operations.
Then, the next 2 lines show the second step: we count the number of rows for each value of 'rowkey'. In that case, those values match for the 2 tables.
Finally, we do the extensive computation of the SHAs for all the columns and rows of the 2 tables. At the end, the script tells us that our 2 tables are identical.
We can also check the returns value of the script: it is 0, which means no differences.

# TODO: explain results (count, sha pb)

# TODO: describe advanced execution

# TODO: explain the installation of Jar + give source code



