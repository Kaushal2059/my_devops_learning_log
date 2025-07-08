# 🐧 Linux Fundamentals and virtualization
 This section documents what I learned during the Linux fundamentals module of my DevOps training. It includes essential command-line operations, user and group management, 
 file editing, service handling, and virtualization using VMs.

## 📂 Basic Linux Operations

### ✅ Topics Covered:
- Navigating directories using `cd`, `ls`, `pwd`
- Managing files and directories using `mkdir`, `touch`, `rm`, `mv`, `cp`
- Viewing file content using `cat`, `less`, `more`, `head`, `tail`
- Understanding file permissions (`chmod`, `chown`)
- Using `man` and `help` for command documentation

  📝 File Editing Tools
Learned Tools:
Nano – Easy terminal-based text editor
Vim – Advanced and powerful editor with command/insert modes

Example: 1> creating a editble file using vim 
![image](https://github.com/user-attachments/assets/fc376362-0402-4cd5-8552-5203755629c6)
   2> Writing a file by enable editing using "i" key
   ![image](https://github.com/user-attachments/assets/5d47a602-2f62-46f7-a0df-f53fa8be906f)
   3> saving the file usineg kesys ecs and :x and viewing the file using cat
   ![image](https://github.com/user-attachments/assets/58be88c4-e707-480a-adc3-991c4d26650f)

🔐 Sudo Privileges & Installing and Deleting Packages/Package management
What I Learned:
The purpose of sudo for privilege escalation
Installing software packages using package managers(apt)
Updating and upgrading the system
Example Commands:
![image](https://github.com/user-attachments/assets/afd74f77-ae0d-4222-9cb3-9f907a7b2e13)

![image](https://github.com/user-attachments/assets/85dcd479-081d-4d1a-89da-36c718000c1e)

 🔧 Service Management
Concepts Practiced:
Starting, stopping, and enabling services
Checking service status

Manually creating a service using systemd
Example:
 ![image](https://github.com/user-attachments/assets/2af1423d-f6b7-47d2-a4ce-181d041625df)
![image](https://github.com/user-attachments/assets/3b485f51-ea7d-43fd-9c7f-cad5b7bf31c1)

🛠 Created a Custom Service:
Wrote a .service file to run a custom Python script
Placed it in /usr/lib/systemd/system/
Enabled and started it via systemctl and daemon reload
Example:
![image](https://github.com/user-attachments/assets/61c140dd-0c82-48b8-a1c8-c33dfab67640)

⚙️ Bash Scripting for Automation
What I Learned:
Wrote simple bash scripts to automate repetitive tasks
Used loops, conditionals, and variables
Made scripts executable and reusable
Sample Script:

![image](https://github.com/user-attachments/assets/d784f08a-3363-4952-8e5e-e7faa0f3d595)
Commands Used:
bash
chmod +x calculatormain.sh
./calculatormain.sh

🕒 Scheduled Tasks with Crontab
What I Learned:
Used crontab to schedule scripts
Wrote cron expressions for minute/hour/day scheduling
Command: crontab -e
wrontab a crontab file to pull form a repo after every 5 miutes
      */5 * * * * kaushal20 /home/kaushal20/gitpull.sh 
      ![image](https://github.com/user-attachments/assets/f29e465c-68df-4619-babb-36b43c80a09d)
created a file to store logs of gitpull:
      ![image](https://github.com/user-attachments/assets/00f84004-429c-411f-abf7-d5466e69a266)



