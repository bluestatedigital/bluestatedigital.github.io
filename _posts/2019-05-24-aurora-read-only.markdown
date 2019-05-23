---
layout: post
title: "A read-only quirk in AWS Aurora"
date:   2019-05-24 17:00:00 -0500
categories: databases, AWS
---

Over the past year, we've migrated many of our legacy MySQL databases hosted on EC2 over to [AWS Aurora](https://aws.amazon.com/rds/aurora/). Generally, we've been extremely happy with this transition: our systems are now more stable, and we have to worry less about failovers and other operational concerns.

One of the things that AWS does promise about this service is that Aurora provides a MySQL-compatible interface. As we started our migrations, we didn't notice any of the problems in our applications, including running against Aurora read nodes.

Until one day:

```
SQLSTATE[HY000]: General error: 1836 Running in read-only mode
```

We hadn't seen this before in our MySQL on EC2 machines, and were wondering what was up. So we broke apart our mess of subqueries until we found an equivalent case:

```
create temporary table test0 (
  c1 int(11), 
  c2 int (11)
);

insert into test0 values (1, (select count(*) from mysql.user));
```

Which we could successfully execute on standard MySQL, but not against an Aurora read node, even as the root user.

We were perplexed. So we filed a support ticket with AWS, and ultimately, after support initially confirmed that they could replicate our example, we heard:

```
I have heard back from the internal team regarding the error ... This has been identified as a bug where temporary table sub-query with a select throws an error. 
```

And so we found a bug in Aurora. With a relatively small change in our query, our application was back up with little effort. Later that week, we went digging into the MySQL source and found a [test case we believe replicates this issue](https://github.com/mysql/mysql-server/blob/5.6/mysql-test/t/read_only_innodb.test#L195).

Aurora has done a lot to reduce our operational load. But in this migration, and with this bug we found, we got a better taste of what life is like on the relative edge of database technologies. Sometimes, you're going to find unexpected hiccups, and that's part of the price of greener services.
