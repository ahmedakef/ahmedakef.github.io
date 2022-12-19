---
title: "on-call starter kit: first actions to do when there is an incident"
date: 2022-12-17
description: being on-call requires fast response to incidents, in this article I will mention the most useful commands I use to get an intuition on the problem.
draft: false
---
** I will add to this article when I learn something new
## Introduction
Being on-call requires a fast response to incidents, in this article I will
mention the most useful actions and commands I use to get a quick intuition on the problem.

I have divided the problems to:
1. Code errors.
2. Latency problems.
3. Configurations issues
## Code errors

### Check your monitoring platform
If your system is already integrated with a monitoring platforms like NewRelic, opening this platform will give you
a quick overview of the most occurring errors and useful aggregations to help you detect the problem quickly,
know the tool your system is using and discover its details to help you in the incident time.

### Check the logs
If your system already has good logging, logs will usually have things to say about the current problem.
One of the things I found useful is to filter out the noisy lines like health checks and successful requests
let's assume you can show the logs of your server using the command
```bash
k logs -f -l app=backend
```
and that noisy lines have this pattern:
```
Successful request /health_check/backend, response: 200
```
you can filter out this using `grep -v` which is `-v, --invert-match        select non-matching lines`
so the command will be:
```bash
k logs -f -l app=backend | grep -v "/health_check/backend, response: 200"
```
what if the error is `500 Internal Server Error` which may mean that there is an internal server error?
let's assume the logged stack trace usually contains this line `/var/app/`
you can filter on these lines using:
```bash
k logs -f app=backend | grep  "/var/app/"
```
mixing them the final line will look like this:
```bash
k logs -f -l app=backend | grep -v "/health_check/backend, response: 200" | grep  "/var/app/"
```

for sure every system has its tools that make things easier to debug, you have to ask your teammates about them first and try to document them to help other new developers.

## Latency problems
some times there are no errors but there is latency in your endpoints or in consuming the jobs.
### Check MySQL slow queries 
Check MySQL for the query that has a big execution time, usually, people use `show full processlist;`
but I have found that this query is more useful:
```sql
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST WHERE command != "sleep" ORDER BY TIME DESC LIMIT 10;
+--------+---------+--------------------+---------+---------+------+-----------+------------------------------------------------------------------------------------------+
| ID     | USER    | HOST               | DB      | COMMAND | TIME | STATE     | INFO                                                                                     |
+--------+---------+--------------------+---------+---------+------+-----------+------------------------------------------------------------------------------------------+
| 2222   | admin   | 197.168.1.1:2222   | db_name | Query   |    0 | executing | SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST WHERE command != "sleep" ORDER BY TIME DESC |
+--------+---------+--------------------+---------+---------+------+-----------+------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```
This query:
* Exclude the `sleep` process since it represents connections waiting for a timeout to terminate and it is usually created by the frameworks to maintain persistent connections to the database.
  * If you have a problem creating new connections, this might be useful tho.
* Order the processes by their time.
* Limit the results to 10.

Some ideas:
* If the query is processing a big number of entities, try batching.
### Check REDIS slow queries
Using this command Redis will show the most 10 slow queries:
```
SLOWLOG GET 10
```
According to [docs](https://redis.io/commands/slowlog-get/):
> The Redis Slow Log is a system to log queries that exceeded a specified execution time. The execution time does not include I/O operations like talking with the client, sending the reply and so forth, but just the time needed to actually execute the command (this is the only stage of command execution where the thread is blocked and can not serve other requests in the meantime).

The important for us is 3rd and 4th value of each entry:

3. The amount of time needed for its execution, in microseconds.
4. The array composing the arguments of the command.

If you think that a query consuming `10 ms` is not too much, you are wrong because when you multiply this in a 100k job for example this is ~16 minutes and this is only one operation in the job.

Some ideas to make use of the results:
- Detect the parts of the code that needs enhancement.
  - Add a limitation on some behaviours
- Develop a quick script to fix the issue without affecting the product.
- Is there a customer having weird behaviour that causes his queries to have big time?

## Configurations issues
Sometimes scaling (vertically or horizontally) isn't useful and even there is no latency issue, so we have to check the configurations.
### Check configurations
It may be related to missing configurations like:
* No worker is consuming from the queue.
* Missing environment variables that control the behaviour.
* Missing DB migrations.

In these scenarios, I have no clear way to detect them but I play with the resources I have to answer some questions:
- If I overscaled a worker, will it consume the jobs faster? if not, then there is a different problem than scaling.
- If I scaled down the worker to 0, will it fix the issue? if yes, the problem is with worker logic.
- If I changed the image of the code, will it fix the issue? if yes, the problem is with a recent release.

### Check server resources usage
Since every system has its way to check the resources, I won't list detailed commands but things to check:
- What is CPU usage percentage?
- What is memory usage percentage?
- What is the disk usage percentage?

If you have an incident because of one of these, you have to think about how to automate being alerted on them so you can take action earlier.

### Check the audit log
If all the above can't explain the issue, you need to check the actions that were made on your servers before the problem, probably someone made a change by mistake that caused things to not work out of the blue.

## Finally
You will learn on-call lessons by the time with each issue you face or by watching how your team members act on the problems and by getting more knowledge and experience with your system.

Keep calm, everything is solvable In Shaa Allah.

Share with me the useful tools you use to debug on-call issues.