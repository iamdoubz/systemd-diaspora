# systemd-diaspora

### Updated for 2021 for diaspora version 0.7.15.0

### Updated for 2023 for diaspora version 0.7.18.2

### Updated for 2024 for diaspora version 0.9.0.0

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
ruby-3.3.5@diaspora
diaspora@computer:~/diaspora$ /bin/bash -lc "rvm current"
ruby-3.3.5@diaspora
```
4. Make sure you can manually run the following two commands and diaspora _starts_:
    - `/bin/bash -lc "bin/bundle exec puma -C config/puma.rb"`
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
Description=diaspora* social network (puma)
PartOf=diaspora.target
StopWhenUnneeded=true

[Service]
User=diaspora
WorkingDirectory=/home/diaspora/diaspora
PIDFile=/home/diaspora/diaspora/tmp/pids/web.pid
Environment=RAILS_ENV=production
ExecStart=/bin/bash -lc "bin/bundle exec puma -C config/puma.rb"
ExecReload=/bin/bash -lc "bin/pumactl -F config/puma.rb restart"
ExecStop=/bin/bash -lc "bin/pumactl -F config/puma.rb stop"
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
🟢 d-web.service - diaspora* social network (puma)
     Loaded: loaded (/etc/systemd/system/d-web.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-10-14 10:02:27 CDT; 3h 13min ago
   Main PID: 825413 (ruby)
      Tasks: 8 (limit: 19776)
     Memory: 329.4M
     CGroup: /system.slice/d-web.service
             └─825413 puma 6.4.2 (unix://tmp/diaspora.sock) [diaspora]

Oct 14 10:02:31 computer bash[825413]: Rack::SSL is enabled
Oct 14 10:02:34 computer bash[825413]: * Listening on unix://tmp/diaspora.sock
Oct 14 10:02:34 computer bash[825413]: Use Ctrl-C to stop


diaspora@computer:/etc/systemd/system$ sudo systemctl status d-side
🟢 d-side.service - diaspora* social network (sidekiq)
     Loaded: loaded (/etc/systemd/system/d-side.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-10-14 10:02:27 CDT; 3h 16min ago
   Main PID: 825412 (ruby)
      Tasks: 18 (limit: 19776)
     Memory: 661.3M
     CGroup: /system.slice/d-side.service
             └─825412 sidekiq 6.5.12 diaspora [0 of 10 busy]
```
19. Navigate to your website, and you shouldn't have a 503 error!

### tag cloud

#diaspora #admin #podmin #question #help #answer #systemd #systemctl #service #target

### sources
- [diaspora* Wiki](https://wiki.diasporafoundation.org/Alternative_startup_methods)
- [diaspora* Discourse](https://discourse.diasporafoundation.org/t/pid-file-could-not-be-created/1640)
- [HowtoForge](https://www.howtoforge.com/how-to-install-diaspora-decentralized-social-media-on-debian-10/)
