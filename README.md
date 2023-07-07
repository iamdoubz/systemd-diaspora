# systemd-diaspora

### Updated for 2021 for diaspora version 0.7.15.0

### Updated for 2023 for diaspora version 0.7.18.1

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
ruby-2.7.8@diaspora
diaspora@computer:~/diaspora$ /bin/bash -lc "rvm current"
ruby-2.7.8@diaspora
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
     Active: active (running) since Fri 2023-07-07 09:33:59 CDT; 50s ago
   Main PID: 101479 (ruby)
      Tasks: 6 (limit: 17935)
     Memory: 268.6M
     CGroup: /system.slice/d-web.service
             ├─101479 unicorn master -c config/unicorn.rb -E production
             ├─101820 unicorn worker[0] -c config/unicorn.rb -E production
             └─101822 unicorn worker[1] -c config/unicorn.rb -E production

Jul 07 09:33:59 dwdu systemd[1]: Started diaspora* social network (unicorn).
Jul 07 09:34:00 dwdu bash[101479]: I, [2023-07-07T09:34:00.083554 #101479]  INFO -- : Refreshing Gem list
Jul 07 09:34:01 dwdu bash[101479]: Top level ::CompositeIO is deprecated, require 'multipart/post' and use `Multipart::Post::CompositeReadIO` instead!
Jul 07 09:34:01 dwdu bash[101479]: Top level ::Parts is deprecated, require 'multipart/post' and use `Multipart::Post::Parts` instead!
Jul 07 09:34:03 dwdu bash[101479]: Rack::SSL is enabled
Jul 07 09:34:05 dwdu bash[101479]: I, [2023-07-07T09:34:05.520983 #101479]  INFO -- : unlinking existing socket=/home/diaspora/diaspora/tmp/diaspora.sock
Jul 07 09:34:05 dwdu bash[101479]: I, [2023-07-07T09:34:05.521100 #101479]  INFO -- : listening on addr=/home/diaspora/diaspora/tmp/diaspora.sock fd=10
Jul 07 09:34:05 dwdu bash[101479]: I, [2023-07-07T09:34:05.530642 #101479]  INFO -- : master process ready
Jul 07 09:34:05 dwdu bash[101820]: I, [2023-07-07T09:34:05.534093 #101820]  INFO -- : worker=0 ready
Jul 07 09:34:05 dwdu bash[101822]: I, [2023-07-07T09:34:05.536925 #101822]  INFO -- : worker=1 ready

diaspora@computer:/etc/systemd/system$ sudo systemctl status d-side
🟢 d-side.service - diaspora* social network (sidekiq)
   Loaded: loaded (/etc/systemd/system/d-side.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2023-07-07 09:33:59 CDT; 7min ago
   Main PID: 101478 (ruby)
      Tasks: 12 (limit: 17935)
     Memory: 310.5M
     CGroup: /system.slice/d-side.service
             └─101478 sidekiq 6.5.1 diaspora [2 of 5 busy]
```
19. Navigate to your website, and you shouldn't have a 503 error!

### tag cloud

#diaspora #admin #podmin #question #help #answer #systemd #systemctl #service #target

### sources
- [diaspora* Wiki](https://wiki.diasporafoundation.org/Alternative_startup_methods)
- [diaspora* Discourse](https://discourse.diasporafoundation.org/t/pid-file-could-not-be-created/1640)
- [HowtoForge](https://www.howtoforge.com/how-to-install-diaspora-decentralized-social-media-on-debian-10/)
