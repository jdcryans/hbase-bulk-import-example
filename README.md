hbase-bulk-import-example
====================

An example of how to bulk import data from CSV files into a HBase table.

HBase gives random read and write access to your big data, but getting your big data into HBase can be a challenge. Using the API to put the data in works, but because it has to traverse HBase's write path (i.e. via the WAL and memstore before it is flushed to a HFile) it is slower than if you simply bypassed the lot and created the HFiles yourself and copied them directly into the HDFS.

Luckily HBase comes with bulk load capabilities, and this example demonstrates how they work. The HBase bulk load process consists of two steps:

1. Data preparation via a MapReduce job, and
2. Completing the data load

The aim of the MapReduce job is to generate HBase data files (HFiles) from your input data using HFileOutputFormat. This output format writes out data in HBase's internal storage format so that they can be efficiently loaded into HBase.

If you understand HBase's architecture you will know that tables are broken down into a number of regions, and in order to work correctly HFileOutputFormat must be configured such that each HFile it generates fits within a single region. To do this Hadoop's TotalOrderPartitioner is used to map output to the key ranges of the regions in the HBase table.

HFileOutputFormat includes a convenience function, configureIncrementalLoad(), which automatically sets up a TotalOrderPartitioner based on the current region boundaries of a table.

There are two methods to import the generated HFiles into a HBase table. The first is a command line tool called completeBulkLoad. The second is a programmatic approach which uses the LoadIncrementalHFiles.doBulkLoad method to load the HFiles generated by the previous MapReduce job into the given HBase table. The later approach is used in this example.

Data preparation
-------------

The data I've used comes from the 2010 NBA Finals (game 1) between the Lakers and Celtics and contains public Facebook and Twitter messages.

> Thanks to simplymeasured.com for providing the data: http://simplymeasured.com/blog/2010/06/lakers-vs-celtics-social-media-breakdown-nba/

The MapReduce job's mapper uses TextInputFormat to read in the Facebook and Twitter messages and outputs the required KV pair classes (ImmutableBytesWritable, KeyValue). These classes are used by the subsequent partitioner and reducer to create the HFiles.

The destination HBase table is pre-split into 6 regions (before game, Q1, Q2, Q3, Q4, and after game). The row key generated by the mapper comprises of the offset in seconds from tip-off, the type of message (Facebook or Twitter), and the senders username: [offset]:[msgtype]:[username] and is used to determine which region (HFile) the record is added to.

There is no need to write your own reducer as the HFileOutputFormat.configureIncrementalLoad() as used in the driver code sets the correct reducer and partitioner up for you.

Setup HBase table
-------------

Open up a HBase shell and run the following to setup the table:

	create 'NBAFinal2010', {NAME => 'srv'}, {SPLITS => ['0000', '0900', '1800', '2700', '3600']} 
	disable 'NBAFinal2010' 
	alter 'NBAFinal2010', {METHOD => 'table_att', MAX_FILESIZE => '10737418240'} 
	enable 'NBAFinal2010'

The alter table sets the max store file size to 10GB so they do not get split automatically.