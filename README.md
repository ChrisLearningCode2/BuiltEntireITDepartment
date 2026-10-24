I Built an Entire IT Department on My Laptop in One Day (and Broke It About 40 Times)
A beginner's honest log of setting up Ansible, OTOBO, Zabbix, and Wiki.js with Docker on CachyOS

TL;DR
I went from "what is Ansible" to a fully functional multi-node lab with ticketing, monitoring, automation, a knowledge base, and mail — all running in Docker containers on my CachyOS desktop. It took one evening. I broke things roughly 40 times. Here's the whole thing, failures included.

The Goal
I work in a large enterprise. I don't have access to servers. Company policy. So I wanted to build my own practice environment at home — a "20-person business IT stack" — to prepare for a sysadmin/DevOps role.

The stack:

Tool	Purpose
Ansible	Deploy and configure servers
OTOBO	Ticketing (like ServiceNow)
Zabbix	Monitoring (like Datadog)
Wiki.js	Knowledge base (like Confluence)
Postfix	Mail server
Docker	Runs all of the above

All on my CachyOS laptop. No cloud. No VMs. No money.

Part 1: The Ansible Lab
What I did
Created a docker-compose.yml with 4 Ubuntu 22.04 containers: one "control" node (runs Ansible) and three "managed" nodes. Each runs an SSH server.

Where I failed
Failure #1: Pasting a Dockerfile into the terminal

I literally typed FROM ubuntu:22.04 at my fish prompt and got:

fish: Unknown command: FROM
fish: Unknown command: RUN

I didn't realize I needed to put it in a file. I'm not going to lie about how long that took me to figure out.

Failure #2: Heredocs in fish

Every multi-line file creation I tried used bash heredoc syntax (cat > file << 'EOF'). I'm on fish. Fish doesn't support heredocs. Every single attempt failed silently or produced garbage. I switched to bash, tried again, and the paste kept losing indentation or adding invisible characters.

Failure #3: The docker-compose.yml paste

My first paste into nano lost all indentation on the node3 section. It was flush-left instead of indented under services:. I had to manually add spaces to 6 lines.

Failure #4: "Permission denied while trying to connect to the docker API"

I'd added myself to the docker group but hadn't logged out. I thought "logging out" meant closing the terminal. It doesn't. The fix was newgrp docker in the current session.

Failure #5: SSH from inside a container to 127.0.0.1

I tried ssh-copy-id -p 2223 root@127.0.0.1 from inside the control container. Got "Connection refused." I didn't understand that 127.0.0.1 inside a container means the container itself, not the host. The port mappings only work from the host.

The fix: Docker Compose puts all containers on the same network. They can reach each other by name. ssh root@ansible-node3 just works. No ports, no 127.0.0.1.

Failure #6: ssh-copy-id kept failing with "Permission denied (publickey)"

The password password was never actually set in the containers. The chpasswd in the Dockerfile wasn't working reliably.

The fix: Skip ssh-copy-id entirely. Use docker exec from the host to write the public key directly into each container's /root/.ssh/authorized_keys. No password needed.

Failure #7: systemctl doesn't exist in Docker containers

Every time I tried to start a service with systemctl start nginx or systemctl restart zabbix-agent2, I got "System has not been booted with systemd." Containers don't have systemd. You have to run the binary directly: /usr/sbin/sshd, /usr/sbin/zabbix_agent2, etc.

Failure #8: Zabbix agent kept dying

Running /usr/sbin/zabbix_agent2 via Ansible started it, but it died when the SSH session ended. Tried -D flag (that's for Agent 1, not Agent 2). Tried & (didn't persist). Finally worked with nohup /usr/sbin/zabbix_agent2 > /dev/null 2>&1 &.

What I actually got working
ansible -i /root/inventory.ini all -m ping
# → pong from all 3 nodes

ansible-playbook -i /root/inventory.ini deploy_web.yml
# → nginx installed, started, page created, verified with HTTP 200

Part 2: OTOBO (Ticketing)
Where I failed
Failure #9: Wrong Docker image name

I guessed otobo/otobo:latest. That image doesn't exist. The correct one is rotheross/otobo:latest. I spent 20 minutes wondering why it wouldn't pull.

Failure #10: rm -rf ~/otobo while inside ~/otobo

I deleted the directory my shell was standing in. Every command after that said "Unable to read current working directory." Fix: cd ~.

Failure #11: Empty OTOBO_DB_ROOT_PASSWORD

I copied the .env template but never filled in the password. OTOBO refused to start: "The Database root password must be set."

Failure #12: Database password mismatch (the big one)

The .env file has OTOBO_DB_ROOT_PASSWORD=otobo_root. During the installer, I set a different password for the database connection. OTOBO kept trying to connect with one password, but MariaDB had a different one stored. The error was:

Access denied for user 'otobo'@'172.19.0.5' (using password: YES)

I tried:

Resetting the user password in MariaDB
Restarting containers
Checking the config file
None of it worked because the password OTOBO was sending (from its internal config) didn't match what was in MariaDB.

The fix: docker compose down -v (wipes the database), start fresh, and type the exact same password in the installer that's in the .env file. They have to match.

Failure #13: "Login failed"

I kept trying to log in with the .env password (otobo_root). That's the database password, not the login password. The login password is whatever I typed during the installer wizard. Two completely different passwords. I'd never clearly understood the distinction.

Failure #14: Daemon crash-looping

The OTOBO daemon kept restarting because "SecureMode is not activated." I tried to enable it via CLI and got command-not-found errors.

The fix: Ignore it. The daemon handles background jobs (email processing, scheduled tasks). The web interface works fine without it. For practice purposes, I don't need it.

What I got working
A full ticketing system at http://localhost. I created my first ticket: "Deploy web server for Project X."

Part 3: Zabbix (Monitoring)
Where I failed
Failure #15: host.docker.internal doesn't exist on Linux

I assumed host.docker.internal would work to let containers reach the host. It's a Docker Desktop (Mac/Windows) feature. On Linux, it doesn't resolve to anything.

The fix: docker network inspect to find the bridge gateway IP (172.17.0.1). Use that in both the Zabbix agent config and the Zabbix host interface.

Failure #16: Zabbix requires an IP, not a hostname

I tried putting host.docker.internal in the "DNS name" field. Zabbix 7.4 still demanded something in the "IP address" field. Red error: "This field cannot be empty."

The fix: Use the actual numeric IP (172.17.0.1) in the IP field.

Failure #17: Couldn't find where to add hosts

I looked under "Administration" → not there. "Inventory" → read-only list, no create button. "Configuration" → doesn't exist in Zabbix 7.

The actual location: Data collection → Hosts → Create host. I spent 10 minutes clicking through menus before finding it.

What I got working
Three nodes showing green in Zabbix with live CPU, memory, and network graphs. I can click a host and see real-time data.

Part 4: Wiki.js (Knowledge Base)
Where I failed
Failure #18: "Invalid DB Type"

I tried to create a config.yml file for Wiki.js. It kept saying the DB type was invalid. The YAML structure I used was wrong for the version.

The fix: Skip config.yml entirely. Put all database settings as environment variables in docker-compose.yml. Much simpler.

Failure #19: Permission denied writing to ~/wiki/data/

The Docker container created the data folder as a different user. My account couldn't write to it.

The fix: sudo chown -R anni ~/wiki/data

Failure #20: "Page path cannot contain illegal characters"

I titled a page "Deploy Web Server — How To" (with an em dash). Wiki.js uses the title as the URL path, and — is illegal in URLs.

The fix: Use hyphens instead of em dashes. "Deploy-Web-Server-How-To."

Failure #21: Couldn't find the delete button

I looked for three dots (⋮). Not there. It's a gear icon (⚙️) in the top-right. I almost deleted the page via direct SQL query into PostgreSQL.

What I got working
A knowledge base at http://localhost:3000 where I can store runbooks and documentation.

The Full Stack (Running)
Tool	URL	Status
OTOBO	http://localhost	✅
Wiki.js	http://localhost:3000	✅
Zabbix	http://localhost:8080	✅
Ansible	CLI (in container)	✅
Postfix	CLI (in container)	✅

Plus my real Microsoft Intune subscription and a Windows VM running AD.

Lessons Learned (the ones that actually mattered)
Fish ≠ Bash. Heredocs, [[ ]], and a bunch of other things just don't work. I kept trying bash syntax in fish and wondering why it failed.
Containers don't have systemd. No systemctl. Run the binary directly. This bit me with nginx, sshd, and zabbix-agent.
127.0.0.1 inside a container is the container, not the host. This is the #1 thing that confused me. Containers talk to each other by name (Docker DNS), not by port-mapped localhost.
Two passwords are not the same password. The database password and the application login password are completely different things. I confused them for 30 minutes.
host.docker.internal is a Docker Desktop thing. On Linux, find your bridge IP with docker network inspect.
Paste is fragile. Multi-line pastes into terminals and editors lose indentation, add invisible characters, and break heredocs. For a beginner, nano + typing (or very careful pasting) is more reliable than clever shell one-liners.
Read the error message. "Access denied for user 'otobo'@'172.19.0.5'" tells you exactly what's wrong. I kept restarting containers instead of fixing the password.
What I Can Do Now (the practice loop)
1. OTOBO: Create a ticket
2. Wiki.js: Look up the runbook
3. Ansible: Run the playbook
4. Zabbix: Verify the server is green
5. OTOBO: Close the ticket

That's the entire job. Ticket in, work done, ticket out.

What's Next
Gitea (git repo for playbooks + basic CI)
Graylog (log aggregation)
Restic (backups)
Convert all my ad-hoc commands into proper Ansible roles
Practice "something broke" scenarios
The Honest Part
I'm not a sysadmin. I don't have a cert. I don't have a degree in IT. I work in a job where I'm not allowed to touch servers.

But today, on my laptop, in Docker containers, I deployed a web server, a database, and a mail server. I built a ticketing system. I set up monitoring. I wrote runbooks. I broke things 20+ times and fixed them all.

That's the job. I just did it.

Stack: CachyOS (Arch-based), Docker, fish shell. No cloud. No VMs. No money spent.
