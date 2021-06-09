# systemd-diaspora

### Updated for 2021 for diaspora version 0.7.14.0

## Assumptions

- You are using a dedicated user to launch diaspora scripts named "diaspora"
- Your home directory is "/home/diaspora"

## Steps

1. Change directory to where you would normally manually start your script. 
    - `cd /home/diaspora/diaspora`
2. Run the following two commands:
    - `rvm current`
    - `/bin/bash -lc "rvm current"`
3. The output of both of these needs to match _exactly_ for the service and target files to work properly!
```bash
diaspora@computer:~/diaspora$ rvm current
ruby-2.6.5@diaspora
diaspora@computer:~/diaspora$ /bin/bash -lc "rvm current"
ruby-2.6.5@diaspora
```
4. Make sure you can manually run the following two commands and diaspora _starts_:
    - `/bin/bash -lc "bin/bundle exec unicorn -c config/unicorn.rb -E production"`
    - `/bin/bash -lc "bin/bundle exec sidekiq"`
    - **NOTE:** to stop the above commands, hit Ctrl + C.
5. If for any reason you cannot start these commands, your install of rvm probably isn't right (not in scope of this post).
6. Change directory into where we will be storing the services `cd /etc/systemd/system/`
7. Create a target file that will call the diaspora unicorn and sidekiq service files `sudo nano diaspora.target`
8. Contents:
```ini
[Unit]
Description=diaspora* social network
## Postgres section
#Wants=postgresql.service redis-server.service
#After=redis-server.service syslog.target network.target postgresql.service
## MySQL section
Wants=redis-server.service
After=redis-server.service syslog.target network.target mysqld.service

[Install]
WantedBy=multi-user.target
```
9. **NOTE:** If you use postgresql, make sure to comment out the MySQL section, and uncomment the Postgres section.
10. Create the diaspora web unicorn service file `sudo nano d-web.service`
11. Contents:
```ini
[Unit]
Description=diaspora* social network (unicorn)
PartOf=diaspora.target
StopWhenUnneeded=true

[Service]
User=diaspora
WorkingDirectory=/home/diaspora/diaspora
PIDFile=/home/diaspora/diaspora/tmp/pids/web.pid
Environment=RAILS_ENV=production
ExecStart=/bin/bash -lc "bin/bundle exec unicorn -c config/unicorn.rb -E production"
ExecReload=/bin/kill -USR2 $MAINPID
Restart=always

[Install]
WantedBy=diaspora.target
```
12. Create the diaspora sidekiq service file `sudo nano d-side.service`
13. Contents:
```ini
[Unit]
Description=diaspora* social network (sidekiq)
PartOf=diaspora.target
StopWhenUnneeded=true

[Service]
User=diaspora
WorkingDirectory=/home/diaspora/diaspora
PIDFile=/home/diaspora/diaspora/tmp/pids/sidekiq1.pid
Environment=RAILS_ENV=production
ExecStart=/bin/bash -lc "bin/bundle exec sidekiq"
Restart=always

[Install]
WantedBy=diaspora.target
```
14. Reload systemd daemon `sudo systemctl daemon-reload`
15. Enable the newly created files `sudo systemctl enable diaspora.target d-web.service d-side.service`
16. Cross your fingers and start the diaspora services `sudo systemctl start diaspora.target`
17. CHECK TO MAKE SURE EVERYTHING IS WORKING `sudo systemctl status d-web` and `sudo systemctl status d-side`
18. If everything went well, you should get:
```bash
diaspora@computer:/etc/systemd/system$ sudo systemctl status d-web
🟢 d-web.service - diaspora* social network (unicorn)
   Loaded: loaded (/etc/systemd/system/d-web.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-01-21 11:22:17 CST; 10s ago
 Main PID: 23443 (ruby)
    Tasks: 9 (limit: 4915)
   CGroup: /system.slice/d-web.service
           ├─23443 unicorn master -c config/unicorn.rb -E production
           ├─23806 unicorn worker[0] -c config/unicorn.rb -E production
           ├─23808 unicorn worker[1] -c config/unicorn.rb -E production
           └─23810 unicorn worker[2] -c config/unicorn.rb -E production

Jan 21 11:22:17 computer systemd[1]: Started diaspora* social network (unicorn).
Jan 21 11:22:18 computer bash[23443]: I, [2021-01-21T11:22:18.341946 #23443]  INFO -- : Refreshing Gem list
Jan 21 11:22:20 computer bash[23443]: Rack::SSL is enabled
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.655211 #23443]  INFO -- : unlinking existing socket=/home/diaspora/diaspora/tmp/diaspora.sock
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.655347 #23443]  INFO -- : listening on addr=/home/diaspora/diaspora/tmp/diaspora.sock fd=10
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.663995 #23806]  INFO -- : worker=0 ready
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.665690 #23443]  INFO -- : master process ready
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.666855 #23808]  INFO -- : worker=1 ready
Jan 21 11:22:22 computer bash[23443]: I, [2021-01-21T11:22:22.670572 #23810]  INFO -- : worker=2 ready

diaspora@computer:/etc/systemd/system$ sudo systemctl status d-side
🟢 d-side.service - diaspora* social network (sidekiq)
   Loaded: loaded (/etc/systemd/system/d-side.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-01-21 11:22:17 CST; 32s ago
 Main PID: 23444 (ruby)
    Tasks: 12 (limit: 4915)
   CGroup: /system.slice/d-side.service
           └─23444 sidekiq 5.2.8 diaspora [0 of 5 busy]

Jan 21 11:22:17 computer systemd[1]: Started diaspora* social network (sidekiq).
Jan 21 11:22:20 computer bash[23444]: Rack::SSL is enabled
```
19. Navigate to your website, and you shouldn't have a 503 error!

### tag cloud

#diaspora #admin #podmin #question #help #answer #systemd #systemctl #service #target

### sources
- [diaspora* Wiki](https://wiki.diasporafoundation.org/Alternative_startup_methods)
- [diaspora* Discourse](https://discourse.diasporafoundation.org/t/pid-file-could-not-be-created/1640)
- [HowtoForge](https://www.howtoforge.com/how-to-install-diaspora-decentralized-social-media-on-debian-10/)
