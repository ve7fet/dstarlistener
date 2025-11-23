# dstarlistener
A notification system that watches the D-STAR database (D-STAR Gateway v3.2) for new user creation, and sends a notification email.

The Python script `dstarlistener` connects to the D-STAR user database, and watches for a notification on the `query_changes` channel from `pg_notify`. When it sees a notification of a new user being inserted in the database, it will send an email via Gmail to notify a SysOp.


## Software Pre-requisites

Install the following pre-requisite packages:

```
sudo dnf install python3-psycopg2 python3-dotenv
```

## Database Credentials

The credentials for accessing the D-STAR database can be found in `/opt/products/dstar/dstar_gw/dsipsvd/dsipsvd.conf`. You will need those for your `.env` file during setup.


## Gmail Account

The `dstarlistener` should work with most mailservers, if you have the right credentials. It is tested using a Gmail account, using an app password. It is an exercise for the user to setup a Gmail account for use with this script. If you are using Gmail, you will need to enable app passwords, but that can only be done after you setup 2-factor authentication.


## PostgreSQL Trigger and Function Setup

The "meat" of this process involves configuring a trigger in PostgreSQL that will call a function when a new user is inserted into the dataase. The `dstarlistener` runs in the background, watching for a notification from the function, and takes the appropriate action to send an email.


### Function Setup

Connect to your D-STAR database:

```
psql -U dstar -d dstar_global
```

Paste the following to create a function:

```
CREATE OR REPLACE FUNCTION notify_trigger() RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('query_changes', NEW.user_cs :: text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This creates a function called `notify_trigger` that sends data to the `query_changes` channel using `pg_notify`.

To confirm the function was created, you can query the functions in the database:

```
dstar_global=# \df
```

```
                            List of functions
 Schema |      Name      | Result data type | Argument data types | Type
--------+----------------+------------------+---------------------+------
 public | notify_trigger | trigger          |                     | func
(1 row)
```


### Trigger Setup

Connect to your D-STAR database:

```
psql -U dstar -d dstar_global
```

Paste the following to create a trigger:

```
CREATE TRIGGER query_changes_trigger AFTER INSERT ON unsync_user_mng
FOR EACH ROW EXECUTE FUNCTION notify_trigger();‍
```

This creates a tigger called `query_changes_trigger` that will call the `notify_trigger` function when a row is inserted in the `unsync_user_mng` table (a new user registers).

To confirm the trigger was created, you can query the triggers in the database:

```
dstar_global=# select * from information_schema.triggers;
```

```
 trigger_catalog | trigger_schema |     trigger_name      | event_manipulation | event_object_catalog | event_object_schema | event_object_table | action_order | action_condition |
         action_statement          | action_orientation | action_timing | action_reference_old_table | action_reference_new_table | action_reference_old_row | action_reference_new_
row | created
-----------------+----------------+-----------------------+--------------------+----------------------+---------------------+--------------------+--------------+------------------+
-----------------------------------+--------------------+---------------+----------------------------+----------------------------+--------------------------+----------------------
----+---------
 dstar_global    | public         | query_changes_trigger | INSERT             | dstar_global         | public              | unsync_user_mng    |            1 |                  |
 EXECUTE FUNCTION notify_trigger() | ROW                | AFTER         |                            |                            |                          |
    |
(1 row)
```

Exit from the PostgreSQL command line with `\q`.


## Configure `dstarlistener`

We will use a `.env` file to hold most of the sensitive information (usernames and passwords). You do **NOT** need to set the credentials in the actual `dstarlistener` script, put them in the `.env` file, the defaults in the `dstarlistener` script are used only if the corresponding keys are not found in the `.env` file (like `SMTP_PORT`, which would default to `465` if an alternate port isn't found in the `.env` file).

Edit the `RENAME.env` file from this repo, and modify it with your necessary environment variables (there are a few more you can add, if desired, see `dstarlistener`).

Use the credentials from `dsipsvd.conf`, as noted above, to fill out `DB_NAME`, `DB_USER`, and `DB_PASSWORD`.

The `EMAIL_USER` and `EMAIL_PASS` are your Gmail username and app password (or the user/pass for your mailserver).

The `RECIPIENT_EMAIL` is the destination you want the notifications sent to.

Once you have edited this file with your particulars, save it, and copy it to the final destination:

```
sudo cp RENAME.env /dstar/scripts/.env
```
Note that when we copy the file, we change the name to `.env`, which will make it a ***hidden*** file in `/dstar/scripts`.

If you need to make changes to `dstarlistener` for you custom mailserver configuration, do that now. Otherwise, move it to its final destination:

```
sudo cp ./dstarlistener /dstar/scripts/dstarlistener
```


## Test!

Now would be a good time to test and see if things work. First, `cd /dstar/scripts` to get to the folder where the script is. If you run `dstarlistener` from the command line, it should start reporting that it is checking for notifications:

```
[dstar@dstar scripts]$ /usr/bin/python3 ./dstarlistener
Waiting for notifications on channel 'query_changes'...
Timeout, checking again...
Timeout, checking again...
Timeout, checking again...
Timeout, checking again...
```

Go to your D-STAR Gateway Registration Page, and register a new user. When you submit the registation, you should see the notification come through:

```
Timeout, checking again...
Timeout, checking again...
Received notification: Channel=query_changes, Payload=W1AW  , PID=211202
Email sent successfully with subject: New D-STAR Gateway User Registration
```

You should receive an email in your inbox with the callsign of the new user that registered. 

Use `ctrl-c` to stop the listener script.


## Configure Logging

We're going to use systemd to run the listener in the final configuration, so let's setup a log file for it:

```
sudo mkdir /var/log/dstarlistener
sudo chown postgres:postgres  /var/log/dstarlistener
sudo touch /var/log/dstarlistener/dstarlistener.log
sudo chown postgres:postgres /var/log/dstarlistener/dstarlistener.log
sudo chmod 664 /var/log/dstarlistener/dstarlistener.log
```


## Auto-start on Boot

AlmaLinux uses systemd to control services. A unit file, `dstar_listener.service` is included to start `dstarlistener` automatically on boot, and restart it if it fails (hopefully).

Copy the unit file where systemd needs it, and enable it:

```
sudo cp ./dstar_listener.service /usr/lib/systemd/system/dstar_listener.service
sudo systemctl daemon-reload
sudo systemctl enable dstar_listener.service
sudo systemctl start dstar_listener.service
sudo systemctl status dstar_listener.service
```

That should get the service up and running, and show you the status.

## Configure Logrotate

You may (probably) want to configure `logrotate` to keep from accidentally filling you hard drive if things go sideways.

Create a file in `/etc/logrotate.d`, and put the following in it (`dstarlistener` would be a good filename):

```
/var/log/dstarlistener/dstarlistener.log
{
		rotate 5
		size 100M
		compress
		compresscmd /bin/xz
		compressext .xz
		create 0640 postgres postgres
		copytruncate
		notifempty
		missingok
		dateext
		dateformat _%Y%m%d-%s
}
```
That will limit the file size to 100MB, compress the archives, and keep 5 previous rotations.

