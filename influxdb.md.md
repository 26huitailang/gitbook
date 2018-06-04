# influxdb.md

\# 安装



for debian 9 strech version，这样才能是最新版本:







\`\`\`

curl -sL 

https://repos.influxdata.com/influxdb.key

 \| sudo apt-key add -

source /etc/os-release

test $VERSION\_ID = "9" && echo "deb 

https://repos.influxdata.com/debian

 stretch stable" \| sudo tee /etc/apt/sources.list.d/influxdb.list

sudo apt-get update && sudo apt-get install influxdb

sudo service influxdb start

\`\`\`



\# 配置



将默认配置导出到文件，\\`influxd -config /etc/influxdb/influxdb.conf\\`，修改配置信息，打开认证。



\`\`\`

\[http\]

enabled = true

bind-address = ":8086"

auth-enabled = true \# ✨

\`\`\`



\# 规则



\[Downsampling and data retention\]\(https://docs.influxdata.com/influxdb/v1.5/guides/downsampling\_and\_retention/\)



\#\# \[retention policy\]\(https://docs.influxdata.com/influxdb/v1.5/query\_language/database\_management/\#retention-policy-management\\)



保留多久的数据，\`CREATE RETENTION POLICY "one\_day\_only" ON "NOAA\_water\_database" DURATION 1d REPLICATION 1 default\`，对数据库NOAA\\_water\\_database创建策略one\\_day\\_only，保留一天并设置为默认的策略。



\#\# \[continuous query\]\(https://docs.influxdata.com/influxdb/v1.5/query\_language/continuous\_queries/\#common-issues-with-advanced-syntax\\)



按什么频率采样



\`\`\`

Create the CQ

Now that we’ve set up our RPs, we want to create a CQ that will automatically and periodically downsample the ten-second resolution data to the 30-minute resolution, and store those results in a different measurement with a different retention policy.

Use the CREATE CONTINUOUS QUERY statement to generate a CQ:

&gt; CREATE CONTINUOUS QUERY "cq\_30m" ON "food\_data" BEGIN

SELECT mean\("website"\) AS "mean\_website",mean\("phone"\) AS "mean\_phone"

INTO "a\_year"."downsampled\_orders"

FROM "orders"

GROUP BY time\(30m\)

END

That query creates a CQ called cq\_30m in the database food\_data. cq\_30m tells InfluxDB to calculate the 30-minute average of the two fields website and phone in the measurement orders and in the DEFAULT RP two\_hours. It also tells InfluxDB to write those results to the measurement downsampled\_orders in the retention policy a\_year with the field keys mean\_website and mean\_phone. InfluxDB will run this query every 30 minutes for the previous 30 minutes.

Note: Notice that we fully qualify \\(that is, we use the syntax "&lt;retention\\_policy&gt;"."&lt;measurement&gt;"\\) the measurement in the INTO clause. InfluxDB requires that syntax to write data to an RP other than the DEFAULT RP. 

\`\`\`



\#\# 实际使用语句说明



continuous query



\`\`\`sql

CREATE CONTINUOUS QUERY cq\_30m ON testdb BEGIN SELECT sum\(value\) INTO org\_stream\_30m FROM org\_stream GROUP BY time\(30m\) END

\`\`\`



在testdb上创建连续查询的规则cq\\_30m：org\\_stream按照30m分钟的频率采样插入到org\\_stream\\_30m，group by time\\(30m\\)，每30分钟执行一次。创建后执行的时间是now\\(\\) - 30m，所以过往的数据是没有的。



\# CLI



\#\# 时间 pretty



进入influx后，输入\`precision rfc3339。\`



\# 备份



\[backing and restore\]\(https://docs.influxdata.com/influxdb/v1.4/administration/backup\_and\_restore/\#backing-up-a-database\\)



