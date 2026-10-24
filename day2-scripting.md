# Day 2: I Learned to Write Scripts (and My Lab Now Starts With One Command)

> Part 2 of building a 20-person IT department on a laptop. Today: bash scripting, safe automation, and turning my daily routine into two commands.

**Stack:** CachyOS · Docker · fish shell · bash scripts · No cloud · No money

---

## What I Did Today

Yesterday I built the lab. Today I made it *convenient*.

I wrote three scripts:

| Script | What it does |
|--------|-------------|
| `start_lab.sh` | Starts all 4 Docker stacks with one command |
| `stop_lab.sh` | Stops everything with one command |
| `check_all.sh` | Pings all nodes, checks disk, memory, and running services |

My daily routine went from 8 commands to 2:

Morning: ~/scripts/start_lab.sh
Evening: ~/scripts/stop_lab.sh


---

## The "Mix and Match" Philosophy

Here's the thing that clicked for me: **a script is just commands I already know, saved in a file.**

I don't need to "learn to code." I need to:

1. Know what commands do what
2. Stack them in a file
3. Add safety (backup, logging, stop-on-error)

That's it. That's scripting for a sysadmin.

### My reusable building blocks

| Part | What it does |
|------|-------------|
| `#!/bin/bash` | Line 1, always. Tells the system "this is bash" |
| `set -e` | Line 2, always. Stops the script if ANY command fails |
| `echo "..."` | Print a label so I know what's running |
| `$(date)` | Timestamp for the log |
| `ansible -m shell -a "..."` | Run a command on my nodes |
| `journalctl -n 20` | Get the last 20 log entries |
| `grep` | Filter the output |
| `tail -10` | Get the last 10 lines |
| `df -h` | Disk usage |
| `free -h` | Memory usage |
| `cp file file.bak` | Backup before I change something |
| `> /var/log/script.log` | Save output to a file |

I pick the pieces I need, drop them in a file, and run it.

---

## My Scripts

### `start_lab.sh`

```bash
#!/bin/bash
set -e

echo "=== Starting IT Lab ==="
echo "Date: $(date)"
echo ""

cd ~/ansible-lab && docker compose up -d
echo "✅ Ansible lab started"

cd ~/otobo && docker compose up -d
echo "✅ OTOBO started"

cd ~/zabbix && docker compose up -d
echo "✅ Zabbix started"

cd ~/wiki && docker compose up -d
echo "✅ Wiki started"

echo ""
echo "=== All containers running ==="
docker ps --format "table {{.Names}}\t{{.Status}}"

stop_lab.sh
#!/bin/bash
set -e

echo "=== Stopping IT Lab ==="
echo "Date: $(date)"
echo ""

cd ~/ansible-lab && docker compose down
echo "✅ Ansible lab stopped"

cd ~/otobo && docker compose down
echo "✅ OTOBO stopped"

cd ~/zabbix && docker compose down
echo "✅ Zabbix stopped"

cd ~/wiki && docker compose down
echo "✅ Wiki stopped"

echo ""
echo "=== All containers stopped ==="

check_all.sh
#!/bin/bash
set -e

echo "=== Lab Health Check ==="
echo "Date: $(date)"
echo ""

echo "--- Ping all nodes ---"
ansible -i /root/inventory.ini all -m ping

echo ""
echo "--- Disk usage ---"
ansible -i /root/inventory.ini all -m shell -a "df -h / | tail -1"

echo ""
echo "--- Memory ---"
ansible -i /root/inventory.ini all -m shell -a "free -h | head -2"

echo ""
echo "--- Services running ---"
ansible -i /root/inventory.ini all -m shell -a "ps aux | grep -E 'nginx|sshd|zabbix' | grep -v grep | wc -l"

echo ""
echo "=== Done ==="

The Safety Rules I Follow
Before I run any script, I check:

[ ] Does it have set -e? (stops on error)
[ ] Did I test it on ONE node first?
[ ] Is there a backup of anything I'm modifying?
[ ] Do I know how to undo it?
[ ] Did I log what it did?
If I can check all five, I run it. If not, I fix the gap first.

The undo principle
In Docker, my undo button is:

docker compose down && docker compose up -d

10 seconds. No data loss. No angry users. No 3am page.

That's why Docker is the perfect place to learn scripting. I can be reckless. Try destructive things. Run a bad command. It doesn't matter. Rebuild in 30 seconds.

Failures Today
#1: nano: command not found

Rebuilt the containers, so nano was gone again. Fix: apt-get install -y nano. I keep forgetting this.

#2: chmod +x missing

Wrote stop_lab.sh, tried to run it, got "exists but is not an executable file." I forgot chmod +x. This will happen to me forever.

#3: The "is that a good resume thing?" panic

I spent 10 minutes wondering if a blog post about my home lab was "professional enough." It is. The failures are the point.

What I Learned
A script is not "code." It's a to-do list that runs itself.
set -e is your best friend. Without it, a broken script keeps going and makes things worse.
chmod +x is a step. Not optional. Every single time.
The second time you do something manually, write the script. First time = learning. Second time = automating.
Docker is my safety net. Break it, rebuild it, move on. No consequences.
The Practice Loop (Updated)
1. OTOBO: Create a ticket
2. Wiki.js: Look up the runbook
3. Ansible: Run the playbook (or the script)
4. Zabbix: Verify it's green
5. OTOBO: Close the ticket
6. If I did it manually: write a script for next time

Step 6 is new. Every manual task becomes a script eventually.

What's Next
[ ] Gitea (git repo for playbooks + scripts)
[ ] Graylog (log aggregation)
[ ] Restic (backups)
[ ] Convert ad-hoc commands into proper Ansible roles
[ ] "Something broke" scenarios
[ ] Write a script that auto-creates a new node and adds it to the inventory
The Honest Part
I still can't SSH into a production server at work. Company policy.

But I can start my entire IT department with one command. I can check the health of all my servers with one command. I can stop everything and free my RAM with one command.

That's the job. I'm doing it. Just... at home. In Docker. With no one to answer to.

And honestly? It's more fun that way.
