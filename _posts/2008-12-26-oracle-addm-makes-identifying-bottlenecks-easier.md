---
layout: post
title: "Oracle ADDM helps diagnosing bottlenecks
date: 2008-12-26
---

Me, an Oracle advocate? I hope I'm **not** becoming a //pragmatic programmer// :).

As you may already know, we have our highest traffic peak every year on Dec, 22th. We offer the users a way to quickly check if their Christmas ticket has won any prize, and use banners to try to make them buy tickets for the next big draw, on Jan, 6th. In Spain, I heard (from journalists, so it's not very reliable) that we spend //in average//, around 150 euros **each**. That means a lot of traffic, and problems trying to get response times reasonable.

We deal with three separate systems: Oracle, Jetty processes, and Apache. We have 6 app servers per stage per location, 2 //stages// per location, 2 locations; 10 servers dedicated to Apache in cluster, per location; and a single Oracle 10G in one location, and a redundant (but stand-by) one in the other.

Our problems usually start when Nagios notifies of too many connections between Jetty to Oracle, too much load in the Oracle server, or too many //httpd-es// processes. At the same time, we inspect Etracker and get Watchmouse reports. Our tools: check sessions in Oracle getting too much resources or taking too long, restarting Jetty processes, toggling stages in each location, balancing traffic to specific locations or not. That's all. We elaborate hipothesis on the run, while trying to survive. We don't dare to make 'last-minute' releases, although we had some concerns about the query all traffic is triggering.

For years, we convinced ourselves that 'tomorrow we'll analyze and understand what happened', but the problem was not related to a lack of time, but a lack of tools and transparency in almost every aspect: We don't have JMX enabled in Jetty (and are not using Oracle's DMS-enabled driver either); we have serious doubts about how mod_jk is performing, after our research on our high percentage of unexpected 503 errors (which is affected by mod_jk configuration, the size of the thread pool in Jetty, and the Jetty release); and Oracle is, in our case, the perfect example of opaqueness: we have had //selects blocking selects// in the past, and no (known) way to find out why.

We used the //statspack// but didn't get conclusions. Last time I recall, we got an absurd number of something called //rollbacks// (about 4 millions in a day), when almost all work was just simple selects; so I concluded that the term should be referring to something else.

Anyway, I found out that Oracle 10 has pre-shipped a (r)evolution of the statspatck: a tool called Automatic Database Diagnostic Monitor - ADDM [1]. I read a technical article explaining the reasons and the concepts, and I got interested.

Lucky us, Automatic Workload Repository - AWR is enabled by default, so I generated some reports to find out the problems from the database point of view, the day after. Here's a verbatim transcript of the main problem:
```
FINDING 1: 100% impact (236784 seconds)
----------------------------------&#x2013;&#x2014;
Host CPU was a bottleneck and the instance was consuming 92% of the host CPU.
All wait times will be inflated by wait for CPU.

   RECOMMENDATION 1: Host Configuration, 100% benefit (236784 seconds)
      ACTION: Consider adding more CPUs to the host or adding instances
         serving the database on other hosts.
      ACTION: Also consider using Oracle Database Resource Manager to
         prioritize the workload from various consumer groups.

   ADDITIONAL INFORMATION:
      Host CPU consumption was 92%.  CPU runqueue statistics are not available
      from the host's OS. This disables ADDM's ability to estimate the impact
      of this finding.

FINDING 2: 89% impact (210500 seconds)
---------------------------------&#x2013;&#x2014;
SQL statements consuming significant database time were found.

   RECOMMENDATION 1: SQL Tuning, 100% benefit (284659 seconds)
      ACTION: Investigate the SQL statement with SQL<sub>ID</sub> "grprqybxz2sdx" for
         possible performance improvements.
Nothing we could do easily for the main point, but we could certainly invest time on the second finding. So I reviewed what ADDM had to say for previous, not-so-difficult days:

  RECOMMENDATIONS
[..]
 SQL ID     : grprqybxz2sdx
SQL Text   : select n, province<sub>id</sub>, g<sub>administration</sub><sub>id</sub>, special<sub>number</sub>,
             available from ( select tm.N, tm.province<sub>id</sub>,
[..]
 --------------------------------------------------------------------------&#x2013;&#x2014;
FINDINGS SECTION (1 finding)
```

---

1- Index Finding (see explain plans section below)

---

  The execution plan of this statement can be improved by creating one
or more indices.

Recommendation (estimated benefit: 100%)

---
- Consider running the Access Advisor to improve the physical schema
design or creating the recommended index.
    create index VEN24GAMES.IDX$$<sub>09AF0001</sub> on
    VEN24GAMES.G<sub>NAT</sub><sub>LOT</sub><sub>TICKET</sub><sub>METADATA</sub>('G<sub>NAT</sub><sub>LOT</sub><sub>TICKET</sub><sub>TYPE</sub><sub>ID</sub>','BOOKED<sub>BY</sub>'
    );

---
Rationale
---
    Creating the recommended indices significantly improves the
execution plan of this statement. However, it might be preferable to
run "Access Advisor" using a representative SQL workload as opposed to
a single statement. This will allow to get comprehensive index
recommendations which takes into account index maintenance overhead and
additional space consumption.
[..]&lt;/pre&gt;
I did as suggested, but executing the Access Advisor threw an error and I took the second option, reducing therefore the impact of such problem from there on. One interesting point is that testing the recommendations for such findings can be done in test or pre-production servers, since what is used for identifying the sql sentence is its hash, and therefore is the same across servers.
</p>

<p>
I think this is an useful tool, and hopefully is able to find also problems regarding structure of physical schemas, tablespaces, discs, etc.
</p>

<p>
[1]  <a href="http://www.oracle.com/technology/products/manageability/database/pdf/twp03/TWP_manage_automatic_performance_diagnosis.pdf">http://www.oracle.com/technology/products/manageability/database/pdf/twp03/TWP_manage_automatic_performance_diagnosis.pdf</a>
</p>
