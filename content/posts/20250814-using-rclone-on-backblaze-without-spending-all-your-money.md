---
title: "Using rclone on Backblaze without spending all your money"
date: 2025-08-14
tags: ["self-host", "backblaze", "backup", "rclone"]
slug: using-rclone-on-backblaze-without-spending-all-your-money
---

[Backblaze](https://www.backblaze.com/) is an object storage company that offers a S3 compatible cloud object storage with very competitive prices. With prices starting at USD6 / TB / month (or less, as you pay proportionally to the amount of data stored), it was a perfect choice to store live data and backups of my projects and self-hosted services.

One of the services that I use is [Immich](https://immich.app/), a self-hosted photo and video management solution that allows me to have fully control on my photos (even in mobile). It doesn't allows me to store my photos directly in a object storage as Backblaze, but I decided that it would be a good idea to keep a backup of the files of the server (photos, videos, and some other data that I need to keep safe).

One tool that simplifies the process is [rclone](https://rclone.org/), a tool that make it easier to sync data in cloud providers (their maintainers call it as "rsync for cloud storage"). I followed of the instructions they provide to [configure Backblaze](https://rclone.org/b2/) and then I was able to run the following command:

`rclone sync /home/immich/library/ b2:my-immich-backup-bucket`

This command syncs automatically only new and changed files (e.g. updated photos metadata), reducing the time needed to backup my photos and keeping them updated, uploading to my Backblaze bucket.

There are many other options that I can use with rclone, and scenarios where this backup scheme doesn't cover. However, as a starting point to give me a bit more of confidence that I will not lose my photos if my VPS provider suddenly decides to stop providing me services, it is good enough.

I configured [crontab](https://www.man7.org/linux/man-pages/man5/crontab.5.html) to run it hourly, and then I start to get my files synchronized.

**So I got the first charge after enabling this scheme!** Usually I was paying less than USD2/month, but the first month I got a USD10 bill, then USD60, then USD70. I wasn´t storing terabytes of data. Looking at my billing report I noticed that now I have a not negligible amount of **Class C (Charged)** transactions, that are responsible for me to pay that much.

We interact with Backblaze using API calls to manage our storage, so we can upload and download files, get information about them and the buckets, and many other actions. Each call is grouped in a different class of transaction (A, B and C), and they are [charged](https://www.backblaze.com/cloud-storage/transaction-pricing) differently. Class A transactions are always free, but Class B and C transactions have a daily limit of 2,500 calls each and they charge for extra calls.

When rclone executes, it gets information from the remote location about the files already there to decide if the local files need to be uploaded or not. This is done by the Class C API call `[b2_list_file_names](
 https://www.backblaze.com/apidocs/b2-list-file-names)` that returns a list of filenames in chunks of 1.000 files each call.

But when we have our files in nested directories, a call is made for each of them. Immich organizes the uploaded files in hundreds of nested directories, and because of that, every time my cron was executed, thousands of `b2_list_file_names` calls were made. Given that I was running it hourly, I reached 17M calls in a month easily (around 500k a day). As I have only 2.5K free calls, I found the reason of the amount I was charged.

![Immich nested directories](immich-nested-directories.png)

The solution is to add `[--fast-list](https://rclone.org/docs/#fast-list)` parameter to the rclone command, that requires fewer transactions for highly recursive operations as we have in this scenario. This parameter [is mentioned](https://rclone.org/b2/) in rclone documentation that explains how to setup it with Backblaze, but I didn't consider that this is basically a mandatory parameter based on how things are charged there!

So I updated my rclone command to:

`rclone sync /home/immich/library/ b2:my-immich-backup-bucket --fast-list`

I reactivated cron again and after almost a day, instead of 500K calls, less than 1K calls were made (which is within the free amount I am entitled to). Probably when I have the double of the number of files I have right now, the daily quota I have will not be enough, but then I will know how to deal with it (possibly with a better backup strategy).